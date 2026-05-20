# Generic ECS CloudFormation Deployment Scaffold

This repository provides reusable GitHub Actions workflows to build a container image, promote it across ECR environments, and deploy an ECS Fargate service with CloudFormation.

## Repository Layout

- `.github/workflows/build-and-deploy.yaml`
  - Top-level orchestrator (manual trigger).
- `.github/workflows/build-push-image.yaml`
  - Reusable workflow: build and push image to DEV ECR.
- `.github/workflows/promote-image.yaml`
  - Reusable workflow: copy image between ECR environments.
- `.github/workflows/ecs-cloudformation-deploy.yaml`
  - Reusable workflow: deploy ECS service stack.
- `common/cloudformation/templates/ecs-service.yaml`
  - Generic ECS service template.
- `common/cloudformation/templates/application-dev.yaml`
- `common/cloudformation/templates/application-staging.yaml`
- `common/cloudformation/templates/application-prod.yaml`
  - Per-environment application configuration files.

## End-to-End Flow

- `build-and-deploy.yaml` validates inputs and normalizes `service_name` to lowercase.
- It builds and pushes the image to DEV ECR.
- Based on `deploy_target`, it deploys:
  - `dev` -> deploy DEV only
  - `staging` -> deploy DEV, promote to STAGING, deploy STAGING
  - `prod` -> deploy DEV, promote to STAGING, deploy STAGING, promote to PROD, deploy PROD

## Where Each Variable Comes From

### 1) Workflow Run Inputs (provided at run time)

Defined in `build-and-deploy.yaml` under `workflow_dispatch.inputs`:

- `service_name`
- `image_tag`
- `context`
- `dockerfile`
- `platforms`
- `deploy_target`
- `stack_name`
- `template_file`
- `config_prefix`

Important behavior:

- `service_name` is normalized to lowercase before use.
- If `image_tag` is empty, workflows use `GITHUB_RUN_ID`.
- If `stack_name` is empty, stack name defaults to `<normalized_service_name>-<env>`.
- `template_file` lets each service use its own CloudFormation template.
- `config_prefix` lets each service use its own `application-<env>.yaml` files.

### 2) GitHub Repository or Environment Variables

These provide infrastructure wiring and account-specific values.

Global:

- `AWS_REGION`

DEV:

- `DEV_ROLE_ARN`
- `DEV_ECR_URL`
- `DEV_ECS_CLUSTER_NAME`
- `DEV_SUBNET_IDS`
- `DEV_SECURITY_GROUP_IDS`
- `DEV_TASK_EXEC_ROLE_ARN`
- `DEV_TASK_ROLE_ARN`

STAGING:

- `STAGING_ROLE_ARN`
- `STAGING_ECR_URL`
- `STAGING_ECS_CLUSTER_NAME`
- `STAGING_SUBNET_IDS`
- `STAGING_SECURITY_GROUP_IDS`
- `STAGING_TASK_EXEC_ROLE_ARN`
- `STAGING_TASK_ROLE_ARN`

PROD:

- `PROD_ROLE_ARN`
- `PROD_ECR_URL`
- `PROD_ECS_CLUSTER_NAME`
- `PROD_SUBNET_IDS`
- `PROD_SECURITY_GROUP_IDS`
- `PROD_TASK_EXEC_ROLE_ARN`
- `PROD_TASK_ROLE_ARN`

### 3) Application Config Files (`application-<env>.yaml`)

These are the source of truth for service behavior and runtime configuration.

Supported deployment keys:

- `DesiredCount`
- `Cpu`
- `Memory`
- `ContainerPort`
- `AssignPublicIp`
- `LogRetentionInDays`
- `EnableExecuteCommand`
- `DeploymentMinHealthyPercent`
- `DeploymentMaxPercent`
- `TargetGroupArn`
- `HealthCheckGracePeriodSeconds`
- `PlatformVersion`
- `DeploymentCircuitBreakerEnabled`
- `DeploymentCircuitBreakerRollback`

Supported app runtime keys:

- Plain environment variables:
  - `Env.<NAME>: <value>`
  - `Environment.<NAME>: <value>` (alias)
- Secret environment variables:
  - `Secret.<NAME>: <valueFromArn>`
  - `Secrets.<NAME>: <valueFromArn>` (alias)

Validation rules enforced in deploy workflow:

- Variable names must match `^[A-Z][A-Z0-9_]*$`.
- Empty values are rejected.
- Max 10 plain env vars and 10 secret env vars per file.

## How Values Reach ECS Task Definition

- `ecs-cloudformation-deploy.yaml` reads:
  - workflow inputs
  - GitHub env-specific vars
  - the selected `application-<env>.yaml`
- It builds CloudFormation parameters and runs `aws cloudformation deploy`.
- `ecs-service.yaml` consumes those parameters and sets:
  - task image: `${EcrUrl}/${RepoName}:${ServiceName}-${ImageTag}`
  - container `Environment`
  - container `Secrets`
  - service scaling/deployment settings

## Example Application File

Use this format in `application-dev.yaml` (same pattern for staging/prod):

- Deployment keys for infrastructure behavior
- `Env.*` keys for plain runtime env vars
- `Secret.*` keys for secret runtime env vars

Example entries:

- `DesiredCount: 1`
- `Cpu: 256`
- `Env.APP_ENV: dev`
- `Env.LOG_LEVEL: info`
- `Secret.API_KEY: arn:aws:secretsmanager:ap-south-1:111111111111:secret:my-dev-api-key`

## Running the Pipeline

In GitHub Actions, run `Generic ECS CloudFormation Pipeline` and fill inputs:

- `service_name`: logical service identifier (any case accepted; pipeline lowercases it).
- `deploy_target`: `dev`, `staging`, or `prod`.
- `context` and `dockerfile`: Docker build config.
- `image_tag`: optional fixed tag (default is run id).
- `stack_name`: optional stack prefix.
- `template_file`: service-specific template path if not using default.
- `config_prefix`: service-specific application config prefix.

## Reusing for Multiple Services

- Keep one common template, or create one template per service.
- For per-service files, choose a service-specific prefix and pass it in `config_prefix`.
  - Example prefix: `services/transaction-catalog/application`
  - Then workflow resolves:
    - `services/transaction-catalog/application-dev.yaml`
    - `services/transaction-catalog/application-staging.yaml`
    - `services/transaction-catalog/application-prod.yaml`

## Operational Notes

- Promotion jobs are bound to GitHub Environments for approvals/policies.
- Deploy command uses `--no-fail-on-empty-changeset`.
- ECR repository names are normalized to lowercase.
- Ensure task execution role has secret read permissions when using `Secret.*`:
  - `secretsmanager:GetSecretValue` and/or `ssm:GetParameters`
  - KMS decrypt permission when required
