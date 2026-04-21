# Bedrock Setup

Amazon Bedrock requires explicit model access to be enabled before any API calls can succeed. This is a one-time console action per AWS account per region.

---

## Why This Step Is Required

Bedrock models are not available by default — AWS requires you to request access to each model family. This is an account-level gate, separate from IAM permissions. Even if the Lambda's IAM role has `bedrock:InvokeModel` permission, calls will fail with `AccessDeniedException` until model access is enabled.

---

## Step 1 — Enable Model Access

1. Open the [Amazon Bedrock console](https://console.aws.amazon.com/bedrock/).
2. Ensure you are in the correct region — this template defaults to **ap-southeast-2 (Sydney)**.
3. In the left navigation, click **Model access**.
4. Click **Modify model access** (or **Request model access** if this is your first time).
5. Find **Anthropic** and enable the model you intend to use:

   | Model | Model ID |
   |-------|---------|
   | Claude 3 Haiku | `anthropic.claude-3-haiku-20240307-v1:0` |
   | Claude 3.5 Haiku | `anthropic.claude-haiku-4-5-20251001` |
   | Claude 3.5 Sonnet | `anthropic.claude-sonnet-4-5` |
   | Claude 3.7 Sonnet | `anthropic.claude-sonnet-4-5-20251001` |

6. Check the box next to the target model and click **Save changes**.

Access is usually granted immediately for Anthropic models. You will see **Access granted** in the Model access table when it is ready.

---

## Step 2 — Configure the Model in Terraform

Set `bedrock_model_id` in `terraform.tfvars` to the model ID you enabled:

```hcl
# infra/environments/dev/terraform.tfvars
bedrock_model_id = "anthropic.claude-3-haiku-20240307-v1:0"
```

Run `terraform apply` to update the Lambda environment variable and IAM policy to match.

The Terraform module scopes the Lambda's `bedrock:InvokeModel` permission to the specific model ARN:

```
arn:aws:bedrock:ap-southeast-2::foundation-model/<bedrock_model_id>
```

Changing the model ID in tfvars and applying will update the IAM policy automatically.

---

## Step 3 — Verify with the Bedrock Endpoint

After enabling model access and deploying, test the integration:

```bash
# Get your API URL
API_URL=$(terraform -chdir=infra/environments/dev output -raw api_url)

# Call the Bedrock test endpoint (requires a valid Cognito JWT)
curl -H "Authorization: Bearer <your-id-token>" \
  "${API_URL}/bedrock-hello"
```

A successful response:

```json
{
  "message": "Hello from Bedrock",
  "model": "anthropic.claude-3-haiku-20240307-v1:0",
  "response": "Hello! How can I help you today?"
}
```

A failure due to model access not being enabled:

```json
{
  "error": "AccessDeniedException: You don't have access to the model with the specified model ID."
}
```

---

## Changing the Model

To switch models after the initial deployment:

1. Enable access to the new model in the Bedrock console (Step 1 above).
2. Update `bedrock_model_id` in `terraform.tfvars`.
3. Run `terraform apply` — this updates both the Lambda environment variable and the IAM policy.
4. No Lambda redeployment is needed; the environment variable change takes effect on the next invocation.

---

## Supported Regions

Bedrock model availability varies by region. The `anthropic.claude-3-haiku-20240307-v1:0` model is available in `ap-southeast-2`. If you change the deployment region, verify model availability in the [Bedrock documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html) before applying.

---

## IAM Permissions Reference

The Lambda execution role is granted the following Bedrock permission by the `lambda` Terraform module:

```json
{
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": "arn:aws:bedrock:ap-southeast-2::foundation-model/<bedrock_model_id>"
}
```

The `bedrock:InvokeModelWithResponseStream` action is not included by default. Add it to `infra/modules/lambda/main.tf` if your application uses streaming responses.

---

## Disabling Bedrock

If you do not need Bedrock in your project, see the [template usage guide](template-usage.md#removing-bedrock) for removal steps.
