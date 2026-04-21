# Deployment Flow

This document covers the full deployment sequence for the AWS Platform Template — from bootstrapping Terraform remote state through to a running application with CI/CD.

---

## Overview

Deployment happens in three layers, each deployed independently:

```
1. Infrastructure  →  Terraform (Cognito, Lambda, API GW, CloudFront, S3, IAM)
2. Backend         →  Lambda function (Node.js 20, built with esbuild)
3. Frontend        →  React SPA (built with Vite, hosted on S3 + CloudFront)
```

The first time you deploy, all three must be applied in order. After that, each layer can be updated independently.

---

## Prerequisites

| Tool | Minimum version |
|------|----------------|
| Terraform | 1.6.0 |
| Node.js | 20 |
| AWS CLI | 2.x |
| Git | any |

You also need:
- An AWS account with permissions to create IAM, S3, Lambda, CloudFront, Cognito, and API Gateway resources.
- An S3 bucket and DynamoDB table for Terraform remote state (created once per AWS account).

---

## Step 0 — Bootstrap Terraform Remote State

Before running `terraform init`, create the backend resources. This is a one-time operation per AWS account.

```bash
# Create the S3 bucket (choose a globally unique name)
aws s3api create-bucket \
  --bucket my-project-terraform-state \
  --region ap-southeast-2 \
  --create-bucket-configuration LocationConstraint=ap-southeast-2

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-project-terraform-state \
  --versioning-configuration Status=Enabled

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket my-project-terraform-state \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Create the DynamoDB lock table
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-southeast-2
```

Update the `backend` block in `infra/environments/dev/main.tf` and `infra/environments/prod/main.tf` with your bucket name and table name.

---

## Step 1 — Infrastructure (First Apply)

### Configure tfvars

```bash
cd infra/environments/dev
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:

```hcl
project_name         = "my-project"
environment          = "dev"
github_org           = "my-github-org"
github_repo          = "my-repo"
bedrock_model_id     = "anthropic.claude-3-haiku-20240307-v1:0"

# Leave this as localhost for the first apply — CloudFront URL isn't known yet
cognito_callback_urls = ["http://localhost:5173"]
```

### First apply

```bash
terraform init
terraform apply
```

Note the `frontend_url` output value — it is your CloudFront distribution URL.

### Second apply — add the real callback URL

Add the CloudFront URL to `cognito_callback_urls` in `terraform.tfvars`:

```hcl
cognito_callback_urls = [
  "http://localhost:5173",
  "https://d1234abcd.cloudfront.net"
]
```

```bash
terraform apply
```

This two-step sequence is required because Cognito's callback URL must reference the CloudFront URL, but CloudFront doesn't exist until after the first apply. Terraform cannot resolve this circular reference in a single pass.

### Key outputs

After applying, capture these values — they are used by the backend and frontend deployments:

```bash
terraform output
```

| Output | Used by |
|--------|---------|
| `lambda_function_name` | Backend deploy |
| `frontend_bucket_name` | Frontend deploy |
| `cloudfront_distribution_id` | Frontend deploy |
| `api_url` | Frontend `.env` |
| `cognito_user_pool_id` | Frontend `.env` |
| `cognito_client_id` | Frontend `.env` |
| `cognito_hosted_ui_domain` | Frontend `.env` |
| `frontend_url` | Frontend `.env` (redirect URI) |
| `infra_deployer_role_arn` | GitHub Actions |
| `backend_deployer_role_arn` | GitHub Actions |
| `frontend_deployer_role_arn` | GitHub Actions |

---

## Step 2 — Backend Deploy

```bash
cd backend
npm install
npm run build:package
```

`build:package` runs esbuild and produces `backend/function.zip`.

Deploy to Lambda:

```bash
aws lambda update-function-code \
  --function-name $(terraform -chdir=../infra/environments/dev output -raw lambda_function_name) \
  --zip-file fileb://function.zip \
  --region ap-southeast-2
```

---

## Step 3 — Frontend Deploy

### Build the frontend

Get the environment variables from Terraform outputs:

```bash
terraform -chdir=infra/environments/dev output frontend_env_example
```

Copy the printed block into `frontend/.env`.

```bash
cd frontend
npm install
npm run build
```

The build outputs to `frontend/dist/`.

### Sync to S3 and invalidate CloudFront

```bash
# Sync to S3
aws s3 sync dist/ \
  s3://$(terraform -chdir=../infra/environments/dev output -raw frontend_bucket_name)/ \
  --delete \
  --region ap-southeast-2

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id $(terraform -chdir=../infra/environments/dev output -raw cloudfront_distribution_id) \
  --paths "/*"
```

Open the `frontend_url` in your browser.

---

## Deploying to Prod

Repeat Steps 1–3 using `infra/environments/prod/`.

**Important:** In `prod/terraform.tfvars`, set `create_oidc_provider = false` if you already applied dev in the same AWS account. The OIDC provider is a per-account singleton — creating it twice will produce a Terraform error.

---

## CI/CD via GitHub Actions

After the first manual deployment, subsequent deployments are handled by GitHub Actions workflows. Each layer has its own workflow triggered by path-filtered pushes to `main`.

| Workflow | Trigger paths | IAM Role |
|---------|--------------|---------|
| `deploy-infra.yml` | `infra/**`, manual | `infra-deployer` |
| `deploy-backend.yml` | `backend/**`, manual | `backend-deployer` |
| `deploy-frontend.yml` | `frontend/**`, manual | `frontend-deployer` |

All workflows use OIDC federation — no AWS credentials are stored in GitHub. See [github-oidc-setup.md](github-oidc-setup.md) for setup instructions.

---

## Deployment Order for Breaking Changes

When a change touches multiple layers, deploy in this order to avoid downtime:

1. **Infrastructure** — API routes, Lambda config, Cognito settings
2. **Backend** — new Lambda code that matches the updated infrastructure
3. **Frontend** — new frontend code that calls the updated API

Reversing this order (frontend first) can briefly expose users to a frontend calling an API that doesn't yet exist.
