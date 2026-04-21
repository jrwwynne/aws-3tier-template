# AWS Platform Template

A reusable, production-minded starter template for deploying web applications on AWS.

This repository is a **platform template**, not a one-off application.  Clone it, adapt it, and deploy multiple separate projects from it — each with its own infrastructure, environments, and CI/CD pipeline — without changing the underlying patterns.

---

## What this includes

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + Vite 5 + TypeScript |
| Backend | AWS Lambda (Node.js 20 + TypeScript) |
| API | API Gateway HTTP API v2 |
| Auth | Amazon Cognito (Hosted UI, PKCE) |
| Infrastructure | Terraform (reusable modules) |
| CI/CD | GitHub Actions (OIDC, no stored keys) |
| AI connectivity | Amazon Bedrock (Claude, configurable model) |
| Region | ap-southeast-2 (Sydney) |

---

## Repository structure

```
/
├── frontend/                    React + Vite + TypeScript SPA
│   ├── src/
│   │   ├── auth/                Cognito PKCE auth helpers
│   │   ├── hooks/               useAuth hook
│   │   └── components/          ApiDemo component
│   └── .env.example             Environment variable reference
│
├── backend/                     Lambda function (Node.js 20)
│   └── src/
│       ├── handlers/            hello.ts, bedrockHello.ts
│       ├── lib/                 bedrockClient.ts wrapper
│       └── index.ts             Router entry point
│
├── infra/
│   ├── modules/                 Reusable Terraform child modules
│   │   ├── cognito/             User pool, app client, hosted UI
│   │   ├── api_gateway/         HTTP API, JWT authoriser, routes
│   │   ├── lambda/              Function, IAM role, Bedrock policy
│   │   ├── frontend_hosting/    S3 + CloudFront (OAC)
│   │   └── github_oidc_roles/   OIDC provider + deployment roles
│   └── environments/
│       ├── dev/                 Dev root configuration
│       └── prod/                Prod root configuration
│
├── .github/
│   ├── workflows/
│   │   ├── deploy-infra.yml     Terraform deploy (wrapper)
│   │   ├── deploy-backend.yml   Lambda deploy (wrapper)
│   │   ├── deploy-frontend.yml  Frontend deploy (wrapper)
│   │   ├── _reusable-terraform.yml
│   │   ├── _reusable-lambda.yml
│   │   └── _reusable-frontend.yml
│   └── actions/
│       ├── setup-aws/           Composite: OIDC credential setup
│       ├── terraform-plan-apply/Composite: init, validate, plan, apply
│       ├── build-backend/       Composite: npm ci + esbuild + zip
│       └── build-frontend/      Composite: npm ci + vite build + S3 sync
│
├── docs/
│   ├── architecture.md
│   ├── deployment-flow.md
│   ├── template-usage.md
│   ├── github-oidc-setup.md
│   ├── bedrock-setup.md
│   └── new-app-guide.md
│
├── scripts/
│   └── local-setup.sh
└── README.md
```

---

## Quick start — local development

```bash
# Install dependencies
./scripts/local-setup.sh

# Configure the frontend (values come from Terraform outputs)
cp frontend/.env.example frontend/.env
# Edit frontend/.env

# Start the frontend dev server
cd frontend && npm run dev

# Build the Lambda package
cd backend && npm run build:package
```

---

## Deploying to AWS

### Prerequisites

- Terraform >= 1.6.0
- AWS CLI configured with credentials that can create IAM, S3, Lambda, CloudFront, Cognito, API Gateway resources
- Node.js 20
- An S3 bucket + DynamoDB table for Terraform remote state (see [deployment-flow.md](docs/deployment-flow.md))

### Step 1 — Bootstrap infrastructure (dev)

```bash
cd infra/environments/dev
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars — set project_name, github_org, github_repo
# Leave cognito_callback_urls as ["http://localhost:5173"] for the first apply

terraform init
terraform apply
```

Note the `frontend_url` output, then add it to `cognito_callback_urls` in `terraform.tfvars` and apply again:

```bash
terraform apply
```

This two-step process resolves the Cognito ↔ CloudFront circular dependency.
See [deployment-flow.md](docs/deployment-flow.md) for the full walkthrough.

### Step 2 — Deploy the backend

```bash
cd backend
npm install
npm run build:package

aws lambda update-function-code \
  --function-name $(terraform -chdir=../infra/environments/dev output -raw lambda_function_name) \
  --zip-file fileb://function.zip
```

### Step 3 — Deploy the frontend

```bash
# Get all config values from Terraform outputs
terraform -chdir=infra/environments/dev output frontend_env_example
# Copy the output into frontend/.env

cd frontend
npm install
npm run build

aws s3 sync dist/ s3://$(terraform -chdir=../infra/environments/dev output -raw frontend_bucket_name)/ --delete
aws cloudfront create-invalidation \
  --distribution-id $(terraform -chdir=../infra/environments/dev output -raw cloudfront_distribution_id) \
  --paths "/*"
```

Open the `frontend_url` in your browser.

### Deploy to prod

Repeat steps 1–3 in `infra/environments/prod/`.  Note that `create_oidc_provider` should be `false` in prod if you already applied dev (the OIDC provider is shared per account).

---

## GitHub Actions CI/CD

### How it works

Three separate workflows deploy each layer independently, using **AWS OIDC federation** — no long-lived AWS credentials are stored in GitHub.

| Workflow | Trigger | Role used |
|---------|---------|----------|
| `deploy-infra.yml` | Push to `main` (infra changes) or manual | `infra-deployer` |
| `deploy-backend.yml` | Push to `main` (backend changes) or manual | `backend-deployer` |
| `deploy-frontend.yml` | Push to `main` (frontend changes) or manual | `frontend-deployer` |

Each workflow delegates to a reusable workflow (`_reusable-*.yml`), which in turn uses composite actions.  The reusable workflows can be extracted to an organisation-level template repository for cross-repo reuse.

### Setting up GitHub Actions

Full setup instructions: [docs/github-oidc-setup.md](docs/github-oidc-setup.md)

Summary:
1. Apply Terraform — the `github_oidc_roles` module creates the OIDC provider and IAM roles automatically.
2. Create `dev` and `prod` GitHub environments in your repository settings.
3. Add the role ARNs and Terraform output values as GitHub environment secrets/variables.

### Required GitHub secrets (per environment)

| Secret | Source |
|--------|--------|
| `INFRA_ROLE_ARN_{ENV}` | `terraform output infra_deployer_role_arn` |
| `BACKEND_ROLE_ARN_{ENV}` | `terraform output backend_deployer_role_arn` |
| `FRONTEND_ROLE_ARN_{ENV}` | `terraform output frontend_deployer_role_arn` |
| `VITE_API_URL_{ENV}` | `terraform output api_url` |
| `VITE_COGNITO_USER_POOL_ID_{ENV}` | `terraform output cognito_user_pool_id` |
| `VITE_COGNITO_CLIENT_ID_{ENV}` | `terraform output cognito_client_id` |
| `VITE_COGNITO_HOSTED_UI_DOMAIN_{ENV}` | `terraform output cognito_hosted_ui_domain` |
| `VITE_COGNITO_REDIRECT_URI_{ENV}` | `terraform output frontend_url` |

### Required GitHub variables (per environment)

| Variable | Source |
|----------|--------|
| `S3_BUCKET_NAME_{ENV}` | `terraform output frontend_bucket_name` |
| `CLOUDFRONT_DISTRIBUTION_ID_{ENV}` | `terraform output cloudfront_distribution_id` |
| `LAMBDA_FUNCTION_NAME_{ENV}` | `terraform output lambda_function_name` |

---

## Authentication

Authentication uses **Cognito Hosted UI with OAuth 2.0 PKCE**.

- The frontend redirects to the Cognito Hosted UI for sign-in.
- Cognito handles credential collection, email verification, and password reset.
- On sign-in, Cognito redirects back with an authorisation code.
- The frontend exchanges the code for tokens using PKCE (no client secret needed).
- The ID token is sent as `Authorization: Bearer <token>` on API calls.
- API Gateway validates the token using a Cognito JWT authoriser.

### Adding users

Users can self-register via the Hosted UI, or you can create them in the Cognito console:

```
AWS Console → Cognito → User pools → {your-pool} → Users → Create user
```

### Customising the Hosted UI

The Hosted UI can be branded in the Cognito console (logo, colours, CSS).  For full control, implement your own sign-in UI using the Cognito token endpoint (the PKCE flow is already in `frontend/src/auth/`).

---

## Bedrock integration

The `/bedrock-hello` endpoint performs a minimal connectivity test against Amazon Bedrock.

**Before using it**, enable model access in the Bedrock console.  See [docs/bedrock-setup.md](docs/bedrock-setup.md).

To change the model, update `bedrock_model_id` in `terraform.tfvars` and run `terraform apply`.

---

## Adapting the template for a new project

### Swapping in a real frontend

1. Replace the contents of `frontend/src/` with your application code.
2. Keep `VITE_*` environment variables in `.env.example` and read them from `import.meta.env`.
3. Ensure `npm run build` outputs to `frontend/dist/`.
4. The S3 + CloudFront hosting and CI/CD workflows are unchanged.

### Swapping in a real backend

1. Replace or extend the handlers in `backend/src/handlers/`.
2. Add routes to `backend/src/index.ts` and `infra/modules/api_gateway/main.tf`.
3. Ensure `npm run build:package` produces `backend/function.zip`.
4. The Lambda, IAM role, and CI/CD workflow are unchanged.

### Creating a new project from this template

See [docs/new-app-guide.md](docs/new-app-guide.md) for the full checklist.

---

## Security

- No AWS credentials are stored in this repository or in GitHub.
- OIDC federation issues temporary credentials per workflow run.
- IAM roles are scoped to the repository and GitHub environment.
- The Lambda execution role is scoped to CloudWatch logs and the specific Bedrock model.
- The frontend S3 bucket blocks all public access; only CloudFront can read from it.
- Cognito PKCE prevents authorisation code interception.
- API Gateway rejects calls without a valid Cognito JWT.

### Where secrets and configuration live

| Item | Location |
|------|----------|
| AWS credentials | Never stored — OIDC only |
| Terraform variables | `terraform.tfvars` (gitignored) |
| Frontend build-time config | GitHub environment secrets/variables |
| Lambda runtime config | Terraform → Lambda environment variables |
| Cognito client ID / pool ID | Public (embedded in frontend JS) — not secret |

---

## Docs

- [Architecture overview](docs/architecture.md)
- [Deployment flow](docs/deployment-flow.md)
- [Template usage guide](docs/template-usage.md)
- [GitHub OIDC setup](docs/github-oidc-setup.md)
- [Bedrock setup](docs/bedrock-setup.md)
- [New app checklist](docs/new-app-guide.md)

---

## Design principles

- **Standardisation over cleverness** — consistent patterns across all projects
- **Reuse over duplication** — modules and reusable workflows for sharing
- **Maintainability over complexity** — minimal abstractions, obvious structure
- **Separation of concerns** — infra, backend, and frontend deploy independently
- **Least privilege** — separate IAM roles per deployment type
# aws-3tier-template
