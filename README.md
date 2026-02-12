# example-todo-infra

AWS infrastructure for the Todo API, deployed at `https://apitodo.jaik.me` (prod), `https://staging.apitodo.jaik.me` (staging), and `https://dev.apitodo.jaik.me` (dev).

## Architecture

This repo uses two CloudFormation templates:

| File | Deployed by | Contains |
|------|------------|----------|
| `infra.yaml` | Git Sync | CI/CD pipeline, IAM roles, S3 artifact bucket |
| `app.yaml` | The pipeline | Lambda, DynamoDB, API Gateway, custom domain |
| `deployment-dev.yaml` | — | Git Sync configuration for dev (Environment=dev, BranchName=dev) |
| `deployment-staging.yaml` | — | Git Sync configuration for staging (Environment=staging, BranchName=staging) |
| `deployment-prod.yaml` | — | Git Sync configuration for prod (Environment=prod, BranchName=main) |

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
* **API Gateway**: REST API with the default SAM `Prod` stage.
* **DynamoDB Table**: NoSQL storage for todo items (`todos-{environment}`).
* **Custom Domain & DNS**: Maps the environment-specific domain (`apitodo.jaik.me`, `staging.apitodo.jaik.me`, or `dev.apitodo.jaik.me`) to the API via Route 53.

## Prerequisites

The following exports must exist in the AWS account:

| Export Name | Description |
|-------------|-------------|
| `GlobalResourcesStack-GitHubConnectionArn` | AWS CodeStar Connection to GitHub |
| `Projects-CertificateArn` | SSL Certificate in us-east-1 |
| `Todo-HostedZoneId` | Route 53 Hosted Zone ID for jaik.me |

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `Environment` | — | Target environment (`dev`, `staging`, `prod`) |
| `BranchName` | `main` | Git branch to pull source from |
| `BuildProjectName` | `TodoGoBuild` | Base name of the CodeBuild project (suffixed with `-{Environment}`) |

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

Each environment has its own deployment file and Git Sync configuration:

* **dev**: `deployment-dev.yaml` — tracks the `dev` branch
* **staging**: `deployment-staging.yaml` — tracks the `staging` branch
* **prod**: `deployment-prod.yaml` — tracks the `main` branch

Pushing to the corresponding branch triggers CloudFormation Git Sync to update that environment's infrastructure stack.
