# New App Checklist

Use this checklist each time you create a new project from the AWS Platform Template. Work through it top to bottom — each section builds on the previous one.

---

## 1. Repository Setup

- [ ] Create a new GitHub repository (from template or manual clone).
- [ ] Update the remote origin: `git remote set-url origin https://github.com/your-org/your-repo.git`
- [ ] Push the initial commit: `git push -u origin main`
- [ ] Create `dev` and `prod` environments in **GitHub → Settings → Environments**.
- [ ] (Optional) Add required reviewers and deployment branch restrictions to the `prod` environment.

---

## 2. Terraform Remote State

- [ ] Create an S3 bucket for Terraform state (globally unique name, versioning enabled, SSE enabled).
- [ ] Create a DynamoDB table for state locking (`LockID` string key, PAY_PER_REQUEST billing).
- [ ] Update the `backend` block in `infra/environments/dev/main.tf` with your bucket name, key, and table name.
- [ ] Repeat for `infra/environments/prod/main.tf` (use a different `key`, e.g., `prod/terraform.tfstate`).

---

## 3. Dev Infrastructure

- [ ] Copy `infra/environments/dev/terraform.tfvars.example` → `terraform.tfvars`.
- [ ] Set `project_name`, `environment = "dev"`, `github_org`, `github_repo`.
- [ ] Set `bedrock_model_id` (or remove Bedrock if not needed — see [template-usage.md](template-usage.md#removing-bedrock)).
- [ ] Set `cognito_callback_urls = ["http://localhost:5173"]` for the first apply.
- [ ] Run `terraform init && terraform apply` (first apply).
- [ ] Note the `frontend_url` output.
- [ ] Add the CloudFront URL to `cognito_callback_urls` in `terraform.tfvars`.
- [ ] Run `terraform apply` (second apply — registers Cognito callback URL).
- [ ] Record all Terraform outputs (see [deployment-flow.md](deployment-flow.md#key-outputs)).

---

## 4. Prod Infrastructure

- [ ] Copy `infra/environments/prod/terraform.tfvars.example` → `terraform.tfvars`.
- [ ] Set `project_name`, `environment = "prod"`, `github_org`, `github_repo`.
- [ ] Set `create_oidc_provider = false` (OIDC provider already exists from dev).
- [ ] Set `cognito_callback_urls = ["http://localhost:5173"]` for the first apply.
- [ ] Run `terraform init && terraform apply` (first apply).
- [ ] Add the prod CloudFront URL to `cognito_callback_urls`.
- [ ] Run `terraform apply` (second apply).
- [ ] Record all prod Terraform outputs.

---

## 5. Amazon Bedrock (if using)

- [ ] Open the Bedrock console in `ap-southeast-2` (or your chosen region).
- [ ] Navigate to **Model access** and enable access for the target Anthropic model.
- [ ] Confirm status shows **Access granted**.
- [ ] Verify `bedrock_model_id` in `terraform.tfvars` matches the enabled model ID.

See [bedrock-setup.md](bedrock-setup.md) for detailed steps.

---

## 6. GitHub Actions Secrets and Variables

For each environment (`dev` and `prod`):

**Secrets** (GitHub → Settings → Environments → {env} → Secrets):

- [ ] `INFRA_ROLE_ARN_{ENV}` → `terraform output infra_deployer_role_arn`
- [ ] `BACKEND_ROLE_ARN_{ENV}` → `terraform output backend_deployer_role_arn`
- [ ] `FRONTEND_ROLE_ARN_{ENV}` → `terraform output frontend_deployer_role_arn`
- [ ] `VITE_API_URL_{ENV}` → `terraform output api_url`
- [ ] `VITE_COGNITO_USER_POOL_ID_{ENV}` → `terraform output cognito_user_pool_id`
- [ ] `VITE_COGNITO_CLIENT_ID_{ENV}` → `terraform output cognito_client_id`
- [ ] `VITE_COGNITO_HOSTED_UI_DOMAIN_{ENV}` → `terraform output cognito_hosted_ui_domain`
- [ ] `VITE_COGNITO_REDIRECT_URI_{ENV}` → `terraform output frontend_url`

**Variables** (GitHub → Settings → Environments → {env} → Variables):

- [ ] `S3_BUCKET_NAME_{ENV}` → `terraform output frontend_bucket_name`
- [ ] `CLOUDFRONT_DISTRIBUTION_ID_{ENV}` → `terraform output cloudfront_distribution_id`
- [ ] `LAMBDA_FUNCTION_NAME_{ENV}` → `terraform output lambda_function_name`

See [github-oidc-setup.md](github-oidc-setup.md) for detailed steps and troubleshooting.

---

## 7. Initial Backend Deploy

- [ ] Run `cd backend && npm install && npm run build:package`.
- [ ] Deploy `function.zip` to the dev Lambda function.
- [ ] Verify the Lambda is updated in the AWS console.

---

## 8. Initial Frontend Deploy

- [ ] Run `terraform -chdir=infra/environments/dev output frontend_env_example` and copy the output into `frontend/.env`.
- [ ] Run `cd frontend && npm install && npm run build`.
- [ ] Sync `dist/` to the S3 bucket.
- [ ] Invalidate the CloudFront distribution.
- [ ] Open `frontend_url` in a browser and confirm the app loads.

---

## 9. End-to-End Smoke Test

- [ ] Sign up for a new Cognito user via the Hosted UI.
- [ ] Confirm the verification email arrives and the account activates.
- [ ] Sign in and confirm the frontend receives tokens.
- [ ] Call the `/hello` API endpoint and confirm a 200 response.
- [ ] (If using Bedrock) Call the `/bedrock-hello` endpoint and confirm a 200 response.

---

## 10. CI/CD Verification

- [ ] Trigger `deploy-backend.yml` manually from the Actions tab (dev environment).
- [ ] Confirm the **Configure AWS credentials** step succeeds.
- [ ] Confirm the Lambda is updated.
- [ ] Trigger `deploy-frontend.yml` manually and confirm S3 sync and CloudFront invalidation succeed.
- [ ] Make a small change to `backend/` and push to `main`. Confirm the backend workflow runs automatically.
- [ ] Make a small change to `frontend/` and push to `main`. Confirm the frontend workflow runs automatically.

---

## 11. Application Customisation

- [ ] Replace `frontend/src/` with your application code (keep auth helpers or replace as needed).
- [ ] Replace or extend `backend/src/handlers/` with your API logic.
- [ ] Add any new API routes to both `backend/src/index.ts` and `infra/modules/api_gateway/main.tf`.
- [ ] Update `frontend/.env.example` to reflect your app's environment variables.
- [ ] Remove template placeholder content (e.g., the `ApiDemo` component).

See [template-usage.md](template-usage.md) for detailed customisation instructions.

---

## 12. Final Checks

- [ ] `terraform.tfvars` is listed in `.gitignore` — confirm it is not tracked by Git.
- [ ] `frontend/.env` is listed in `.gitignore` — confirm it is not tracked by Git.
- [ ] No hardcoded AWS account IDs, region names, or credentials in application code.
- [ ] README updated with project-specific name, description, and any changed setup steps.
- [ ] Prod smoke test completed (repeat steps 7–9 for the prod environment).
