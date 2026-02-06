# example-todo-infra

AWS infrastructure for the Todo API, deployed at `https://apitodo.jaik.me`.

## Architecture

This repo uses two CloudFormation templates:

| File | Deployed by | Contains |
|------|------------|----------|
| `infra.yaml` | Git Sync | CI/CD pipeline, IAM roles, S3 artifact bucket |
| `app.yaml` | The pipeline | Lambda, DynamoDB, API Gateway, custom domain |
| `deployment-file.yaml` | — | Git Sync configuration (points to `infra.yaml`) |

### CI/CD Workflow

1. **Git push** to `example-todo-infra` (main) triggers CloudFormation Git Sync to update `infra.yaml`.
2. **CodePipeline** starts automatically:
    * **Source**: Pulls Go code from the app repo and `app.yaml` from this repo.
    * **Build**: Compiles the Go binary (ARM64) and runs `sam package` to create `packaged.yaml`.
    * **Deploy**: CloudFormation uses `packaged.yaml` to create/update the application stack.

## Resources Created

### infra.yaml (Infra Stack)
* **S3 Bucket**: Stores artifacts (source code, binaries, templates) between pipeline stages.
* **CodeBuild Project**: Specialized Linux environment to compile Go and package the SAM template.
* **CodePipeline**: The orchestrator connecting GitHub to AWS.
* **IAM Roles**:
    * **CodeBuildServiceRole**: Logging and S3 access.
    * **CodePipelineServiceRole**: Permission to trigger builds and deploy stacks.
    * **CloudFormationDeployRole**: Permission to manage Lambda, DynamoDB, and the `ops.apigateway` Service-Linked Role.

### app.yaml (App Stack, deployed by pipeline)
* **Lambda Function**: ARM64 Go binary running on the `provided.al2023` runtime.
* **API Gateway**: REST API with the default SAM `Stage` stage.
* **DynamoDB Table**: NoSQL storage for todo items (`todos-{environment}`).
* **Custom Domain & DNS**: Maps `apitodo.jaik.me` to the API via Route 53.

## Prerequisites

The following exports must exist in the AWS account:

| Export Name | Description |
|-------------|-------------|
| `GlobalResourcesStack-GitHubConnectionArn` | AWS CodeStar Connection to GitHub |
| `TodoApi-CertificateArn` | SSL Certificate in us-east-1 |
| `Todo-HostedZoneId` | Route 53 Hosted Zone ID for jaik.me |

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `Environment` | `dev` | Target environment (`dev`, `staging`, `prod`) |
| `BuildProjectName` | `TodoGoBuild` | Name of the CodeBuild project |

## API Routes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/todos` | List all todos |
| POST | `/todos` | Create a todo |
| GET | `/todos/{id}` | Get a todo by ID |
| PUT | `/todos/{id}` | Update a todo by ID |
| DELETE | `/todos/{id}` | Delete a todo by ID |
| OPTIONS | `/todos` | CORS preflight |
| OPTIONS | `/todos/{id}` | CORS preflight |

## Deployment

This stack uses **CloudFormation Git Sync**. Pushing to the `main` branch of this repository triggers an automatic update of the infrastructure.
