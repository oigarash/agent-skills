---
name: cxj-dify-maintenance
description: >
  Operational maintenance and Admin API management for cxj-dify platform
  across all environments (dev, tool, team, sandbox). Covers credential
  rotation, health checks, LLM provider configuration, plugin management,
  database operations, and periodic upkeep. Use this skill when: rotating
  API keys, checking pod health, verifying credentials, configuring LLM
  providers (Gemini, Azure OpenAI, Bedrock, Vertex AI), managing plugins,
  investigating incidents, or running scheduled maintenance on cxj-dify
  environments. Triggers on: maintenance, key rotation, credential rotation,
  health check, backup, restore, database, incident, INC ticket, GCP key,
  service account, cleanup, LLM provider, model provider, Admin API,
  model settings, credentials, workspace, provider config, enable model,
  disable model, Azure OpenAI, Gemini, plugin, GCP API key, pod restart.
---

# cxj-dify Maintenance

Operational guide for maintaining and managing cxj-dify environments on the
Utena platform. For Kubernetes environment details (namespaces, contexts, pods,
Helm, logs, troubleshooting), see the `utena-deploy` skill.

## Environments

| Env | Purpose | Host |
|-----|---------|------|
| **dev** | Development & CI testing | cxj-dify-dev.cisco.com |
| **tool** | Tool production | cxj-dify-tool.cisco.com |
| **team** | Team production | cxj-dify-team.cisco.com |
| **sandbox** | User-facing testing | cxj-dify-sandbox.cisco.com |

For kubectl context, namespace, deploy/rollout, and Helm operations,
refer to `utena-deploy` skill's Environment table.

### Workspace IDs (for Admin API)

| Env | Workspace ID |
|-----|-------------|
| dev | `c9f8a404-1bf9-46be-8f73-418bf0c856f3` |
| tool | `0ae1ec29-902f-4404-b4a1-65a2e8ef459b` |
| team | `71a5ff30-b441-41ff-a6a0-67b842d8bbec` |
| sandbox | `a4d1e7bc-74d4-42cb-9d80-1daae65f40d1` |

> Workspace IDs are per-tenant. They change if the database is recreated.
> Verify with DB query if unsure (see `references/database-queries.md`).

## Admin API

All API calls require two headers:

```
Authorization: Bearer {ADMIN_API_KEY}
X-WORKSPACE-ID: {workspace_id}
```

`ADMIN_API_KEY` is stored in GitLab CI variable and deployed as an env var
on the API deployment. Retrieve from any environment:

```bash
kubectl --context {ctx} -n {ns} get deploy {ns}-api \
  -o jsonpath='{.spec.template.spec.containers[0].env}' | \
  python3 -c "
import sys, json
for e in json.load(sys.stdin):
    if e.get('name') == 'ADMIN_API_KEY':
        print(e['value'])
"
```

### Provider Names

| Display Name | Provider ID |
|---|---|
| Gemini | `langgenius/gemini/google` |
| Azure OpenAI | `langgenius/azure_openai/azure_openai` |
| Amazon Bedrock | `langgenius/bedrock/bedrock` |
| Vertex AI | `langgenius/vertex_ai/vertex_ai` |
| OpenAI | `langgenius/openai/openai` |
| Anthropic | `langgenius/anthropic/anthropic` |

### API Endpoints

Base URL: `https://{host}/console/api/workspaces/current`

Provider name must be URL-encoded in paths (`/` → `%2F`).

#### Provider Credentials (CRUD)

| Operation | Method | Path |
|---|---|---|
| List providers | GET | `/model-providers` |
| Get credentials | GET | `/model-providers/{provider}/credentials` |
| Create credentials | POST | `/model-providers/{provider}/credentials` |
| Update credentials | PUT | `/model-providers/{provider}/credentials` (requires `credential_id`) |
| Delete credentials | DELETE | `/model-providers/{provider}/credentials` (requires `Content-Type: application/json` + `credential_id` in body) |
| Validate credentials | POST | `/model-providers/{provider}/credentials/validate` |

#### Model Credentials (Azure OpenAI customizable-model)

| Operation | Method | Path |
|---|---|---|
| Add model | POST | `/model-providers/{provider}/models/credentials` |

> **IMPORTANT**: Use `/models/credentials` (NOT `/models`). `/models` POST is for load-balancing only.

#### Model Enable/Disable (predefined-model providers)

| Operation | Method | Path |
|---|---|---|
| Enable | PATCH | `/model-providers/{provider}/models/enable` |
| Disable | PATCH | `/model-providers/{provider}/models/disable` |
| List active | GET | `/models/model-types/{type}` (`llm`, `text-embedding`) |

Payload: `{"model": "{name}", "model_type": "llm"}`

> **Note on model visibility**: Models with credentials but no `provider_model_settings`
> record are **enabled by default**. The settings record is only created when explicitly toggled.

### Plugin Management

Plugins control which models are available. See `references/plugin-management.md` for details.

#### Marketplace plugins (langgenius/*)

| Operation | Method | Path |
|---|---|---|
| Install | POST | `/plugin/install/marketplace` |
| Upgrade | POST | `/plugin/upgrade/marketplace` |
| Check task | GET | `/plugin/tasks/{task_id}` |

#### Custom plugins (ciscotac/* — GitLab hosted)

No `upgrade/pkg` endpoint exists. Use the **uninstall → upload → install** workflow.
See `references/plugin-management.md` for step-by-step instructions.

> **IMPORTANT**: If registering a model returns `"Variable base_model_name is not in options"`,
> the plugin is too old. Upgrade it first.

## GCP Access

GCP Project `gcp-cxcentersjapang-prd-48251` hosts Gemini API keys and the
`vertex-ai-runner` service account.

```bash
gcloud auth login
gcloud projects list --filter="gcp-cxcentersjapang-prd-48251"
```

## Health Check Checklist

Run periodically (recommended: monthly) to verify environment health.

1. **GCP API Keys**: Check `createTime` — rotate if older than 90 days
   ```bash
   gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 \
     --format="table(displayName,uid,createTime)"
   ```
2. **Service Account Keys**: Flag USER_MANAGED keys without expiry or older than 90 days
   ```bash
   gcloud iam service-accounts keys list \
     --iam-account=vertex-ai-runner@gcp-cxcentersjapang-prd-48251.iam.gserviceaccount.com \
     --project=gcp-cxcentersjapang-prd-48251 \
     --format="table(name.basename(),keyType,validAfterTime,validBeforeTime)"
   ```
3. **Dify provider credentials**: Cross-check Dify-configured keys match GCP/Azure
4. **Plugin versions**: Compare across environments for drift
5. **Pod health / disk / Helm**: See `utena-deploy` skill

## Reference Files

| File | Content |
|---|---|
| `references/gcp-api-key-rotation.md` | GCP API Key rotation procedure for Gemini provider |
| `references/gcp-sa-key-rotation.md` | GCP Service Account Key rotation for vertex-ai-runner |
| `references/gemini-gcp-management.md` | GCP API key inventory, Gemini E2E setup workflow |
| `references/azure-openai-resources.md` | Azure subscription, resources, deployment CLI commands, E2E workflow |
| `references/credential-schemas.md` | JSON schemas for each provider (Gemini, Bedrock, Vertex AI, Azure OpenAI) |
| `references/plugin-management.md` | Plugin install, upgrade, version check, DB tables |
| `references/database-queries.md` | kubectl+psycopg2 snippets for DB queries, schema reference |
| `references/troubleshooting.md` | Common issues: 401, status unknown, DNS, encryption, plugin version |
