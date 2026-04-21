# Template Usage Guide

This guide explains how to adapt the AWS Platform Template for a real project — swapping in your own frontend, backend, and configuration while keeping the infrastructure and CI/CD patterns intact.

---

## What stays the same

When you use this template, you keep:

- All Terraform modules (`infra/modules/`)
- The GitHub Actions workflow structure (`.github/workflows/`, `.github/actions/`)
- The local development scripts (`scripts/local-setup.sh`)
- The deployment patterns documented in [deployment-flow.md](deployment-flow.md)

What you replace is the application code inside `frontend/src/` and `backend/src/handlers/`.

---

## Creating a New Project from This Template

### Option A — GitHub template repository

If this repository is marked as a GitHub template:

1. Click **Use this template** → **Create a new repository**.
2. Name the new repo and choose its visibility.
3. Clone the new repo locally.

### Option B — Manual clone

```bash
git clone https://github.com/your-org/aws-3tier-template.git my-new-project
cd my-new-project
git remote set-url origin https://github.com/your-org/my-new-project.git
git push -u origin main
```

---

## Adapting the Frontend

### Replace application code

Replace everything inside `frontend/src/` with your application. Keep:

- `frontend/src/auth/` — Cognito PKCE helpers (unless you are replacing Cognito entirely)
- `frontend/src/hooks/useAuth.ts` — auth state hook
- `frontend/.env.example` — update this to reflect your app's env vars

Rules:
- Read all config from `import.meta.env.VITE_*` variables.
- Ensure `npm run build` outputs to `frontend/dist/` (Vite default).
- Do not hardcode API URLs or Cognito identifiers.

### Add new environment variables

1. Add the variable to `frontend/.env.example` with a placeholder value.
2. Add the variable to the `Build frontend` step in `.github/actions/build-frontend/action.yml`.
3. Add the corresponding GitHub environment secret or variable (see [github-oidc-setup.md](github-oidc-setup.md)).

---

## Adapting the Backend

### Add a new Lambda handler

1. Create a new file in `backend/src/handlers/`:

```typescript
// backend/src/handlers/myFeature.ts
import { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from 'aws-lambda';

export const handler = async (
  event: APIGatewayProxyEventV2
): Promise<APIGatewayProxyResultV2> => {
  return {
    statusCode: 200,
    body: JSON.stringify({ message: 'Hello from myFeature' }),
  };
};
```

2. Register the route in `backend/src/index.ts`:

```typescript
import { myFeatureHandler } from './handlers/myFeature';

// Inside the router:
if (routeKey === 'GET /my-feature') return myFeatureHandler(event);
```

3. Add the route to `infra/modules/api_gateway/main.tf`:

```hcl
resource "aws_apigatewayv2_route" "my_feature" {
  api_id             = aws_apigatewayv2_api.this.id
  route_key          = "GET /my-feature"
  authorization_type = "JWT"
  authorizer_id      = aws_apigatewayv2_authorizer.cognito.id
  target             = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}
```

Apply Terraform and redeploy the Lambda to activate the route.

### Unauthenticated routes

To expose a route without JWT auth, set `authorization_type = "NONE"` on the API Gateway route.

### Add Lambda environment variables

Add the variable in `infra/modules/lambda/main.tf` under the `environment` block:

```hcl
environment {
  variables = {
    EXISTING_VAR  = var.existing_var
    MY_NEW_CONFIG = var.my_new_config
  }
}
```

Declare the variable in `infra/modules/lambda/variables.tf` and pass it from the environment root (`infra/environments/dev/main.tf`).

---

## Adapting Infrastructure

### Rename the project

Set `project_name` in `terraform.tfvars`. All AWS resource names are derived from this value — no other changes are needed.

### Change the AWS region

Update the `provider` block region in `infra/environments/dev/main.tf` and `infra/environments/prod/main.tf`. Also update the remote state bucket region and the `--region` flag in any `aws` CLI commands.

### Add a new environment

Copy `infra/environments/dev/` to `infra/environments/staging/` and update:

- `terraform.tfvars` — set `environment = "staging"`
- `main.tf` — update the backend `key` to `staging/terraform.tfstate`
- GitHub Actions workflows — add `staging` as a target environment

---

## Renaming GitHub Secrets and Variables

The workflow files reference secrets and variables using the `{ENV}` suffix convention (e.g., `INFRA_ROLE_ARN_DEV`). If you change the environment name, update both the GitHub environment name and the secret/variable names to match.

See the full list of required secrets in [github-oidc-setup.md](github-oidc-setup.md).

---

## Removing Bedrock

If your project does not use Bedrock:

1. Delete `backend/src/handlers/bedrockHello.ts` and `backend/src/lib/bedrockClient.ts`.
2. Remove the `/bedrock-hello` route from `backend/src/index.ts`.
3. Remove the `aws_apigatewayv2_route.bedrock_hello` resource from `infra/modules/api_gateway/main.tf`.
4. Remove the Bedrock IAM policy from `infra/modules/lambda/main.tf`.
5. Remove `bedrock_model_id` from `infra/modules/lambda/variables.tf` and all environment tfvars files.

---

## Removing Cognito

If your project uses a different auth provider:

1. Remove the `cognito` module call from the environment root `main.tf`.
2. Replace the `authorization_type = "JWT"` + `authorizer_id` on API Gateway routes with your chosen authoriser.
3. Remove the Cognito auth helpers from `frontend/src/auth/` and replace with your auth library.
4. Remove Cognito-related `VITE_*` variables from `.env.example` and GitHub secrets.

---

## Keeping the Template Up to Date

If you want to pull in upstream improvements after forking:

```bash
git remote add template https://github.com/your-org/aws-3tier-template.git
git fetch template
git merge template/main --allow-unrelated-histories
```

Resolve conflicts manually — your application code in `frontend/src/` and `backend/src/handlers/` will not conflict with template infrastructure changes.
