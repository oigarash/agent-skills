# Provider Credential Schemas

## Gemini (provider-level)

```json
{"credentials": {"google_api_key": "AIzaSy..."}, "name": "GCP"}
```

## Amazon Bedrock (provider-level)

```json
{"credentials": {"auth_method": "API_Key", "bedrock_api_key": "...", "aws_region": "us-east-2"}, "name": "API_KEY1"}
```

## Vertex AI (provider-level)

```json
{"credentials": {"vertex_project_id": "...", "vertex_location": "asia-northeast1", "vertex_service_account_key": "..."}, "name": "API_KEY1"}
```

## Azure OpenAI (model-level, not provider-level)

Azure OpenAI uses **customizable-model** configuration. Each model is added individually.

> **IMPORTANT**: The endpoint is `/models/credentials` (NOT `/models`).
> `/models` POST is for load-balancing config only and returns `{"result":"success"}` without saving credentials.

```bash
# Add a custom model credential (e.g., gpt-5.2)
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_API_KEY" \
  -H "X-WORKSPACE-ID: $WID" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2",
    "model_type": "llm",
    "credentials": {
      "openai_api_base": "https://{resource}.openai.azure.com/openai/deployments/{deployment}/chat/completions?api-version=2025-01-01-preview",
      "auth_method": "api_key",
      "openai_api_key": "...",
      "openai_api_version": "2025-01-01-preview",
      "base_model_name": "gpt-5.2"
    },
    "name": "gpt-5.2-team"
  }' \
  "https://$HOST/console/api/workspaces/current/model-providers/langgenius%2Fazure_openai%2Fazure_openai/models/credentials"
```

Payload fields (from `ParserCreateCredential`): `model`, `model_type`, `credentials`, `name` (max 30 chars).

For embeddings, use `/embeddings?api-version=...` in the URL instead of `/chat/completions?api-version=...`.

> **IMPORTANT**: Use `.openai.azure.com` (NOT `.cognitiveservices.azure.com`) in `openai_api_base`. The `.cognitiveservices.azure.com` hostname fails DNS resolution from Utena pods.

> **IMPORTANT**: `base_model_name` must be recognized by the Azure OpenAI plugin. If it returns `"Variable base_model_name is not in options"`, the plugin version is too old. See `references/plugin-management.md` for upgrade instructions.
