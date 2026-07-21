# GCP API Key Rotation (Gemini Provider)

Each Dify environment has a dedicated GCP API Key for the Gemini provider.
Keys follow the `gemini-dify-{env}` naming convention and are restricted to
`generativelanguage.googleapis.com` and `aiplatform.googleapis.com`.

## Current State (as of 2026-03)

Refer to `dify-admin-api/references/gemini-gcp-management.md` for the
current key UIDs and GCP project details.

## Rotation Procedure

### Step 1: Audit current keys

Compare GCP-side keys with Dify-configured credentials.

```bash
# List GCP API keys
gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 \
  --format="table(displayName,uid,createTime)"

# Get masked key for each environment and compare with Dify
for env in dev tool team sandbox; do
  key_string=$(gcloud services api-keys get-key-string --project=gcp-cxcentersjapang-prd-48251 \
    $(gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 \
      --format="value(name)" --filter="displayName=gemini-dify-$env") \
    --format="value(keyString)" 2>/dev/null)
  echo "gemini-dify-$env: ${key_string:0:6}****${key_string: -2}"
done
```

Check Dify credentials via Admin API (see cxj-dify-maintenance SKILL.md for
ADMIN_API_KEY retrieval):

```bash
curl -s "https://{host}/console/api/workspaces/current/model-providers/langgenius%2Fgemini%2Fgoogle/credentials" \
  -H "Authorization: Bearer {ADMIN_API_KEY}" \
  -H "X-WORKSPACE-ID: {workspace_id}"
# Returns: {"credentials": {"google_api_key": "AIzaSy************xx"}}
```

### Step 2: Create new API keys

Create one new key per environment:

```bash
for env in dev tool team sandbox; do
  gcloud services api-keys create \
    --project=gcp-cxcentersjapang-prd-48251 \
    --display-name="gemini-dify-${env}-new" \
    --api-target=service=generativelanguage.googleapis.com \
    --api-target=service=aiplatform.googleapis.com \
    --format="value(response.keyString)"
done
```

For standalone keys (e.g. `gemini-evaluation`), create individually with the same command.

### Step 3: Test new keys

Verify each new key returns models from the Gemini API:

```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models?key={NEW_API_KEY}" | \
  python3 -c "
import sys, json
models = json.load(sys.stdin).get('models', [])
print(f'OK: {len(models)} models') if models else print('FAIL')
"
```

### Step 4: Update Dify credentials

For each environment, get the `credential_id` and update:

```bash
# Get credential_id from provider list
curl -s "https://{host}/console/api/workspaces/current/model-providers" \
  -H "Authorization: Bearer {ADMIN_API_KEY}" \
  -H "X-WORKSPACE-ID: {workspace_id}" | \
  python3 -c "
import sys, json
for p in json.load(sys.stdin):
    if 'gemini' in p.get('provider',''):
        print(p['custom_configuration']['current_credential_id'])
"

# Update credentials with new key
curl -s -X PUT \
  "https://{host}/console/api/workspaces/current/model-providers/langgenius%2Fgemini%2Fgoogle/credentials" \
  -H "Authorization: Bearer {ADMIN_API_KEY}" \
  -H "X-WORKSPACE-ID: {workspace_id}" \
  -H "Content-Type: application/json" \
  -d '{
    "credential_id": "{credential_id}",
    "credentials": {"google_api_key": "{NEW_API_KEY}"},
    "name": "GCP"
  }'
# Expected: {"result": "success"}
```

### Step 5: Verify update

Confirm the new key suffix appears in Dify:

```bash
curl -s "https://{host}/console/api/workspaces/current/model-providers/langgenius%2Fgemini%2Fgoogle/credentials" \
  -H "Authorization: Bearer {ADMIN_API_KEY}" \
  -H "X-WORKSPACE-ID: {workspace_id}"
```

### Step 6: Clean up old keys

Rename, then delete old keys from GCP:

```bash
for env in dev tool team sandbox; do
  # Rename old -> -old
  gcloud services api-keys update \
    $(gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 \
      --format="value(name)" --filter="displayName=gemini-dify-${env}") \
    --display-name="gemini-dify-${env}-old"

  # Rename new -> correct name
  gcloud services api-keys update \
    $(gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 \
      --format="value(name)" --filter="displayName=gemini-dify-${env}-new") \
    --display-name="gemini-dify-${env}"

  # Delete old
  gcloud services api-keys delete \
    $(gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 \
      --format="value(name)" --filter="displayName=gemini-dify-${env}-old") \
    --project=gcp-cxcentersjapang-prd-48251 --quiet
done
```

### Step 7: Update documentation

Update UID table in `dify-admin-api/references/gemini-gcp-management.md`
with new UIDs and rotation date.

## Rotation History

| Date | Scope | Reason |
|------|-------|--------|
| 2026-06-15 | gemini-dify-{dev,tool,team,sandbox} | Proactive rotation (~96d old, ahead of CIS 90d). gemini-evaluation NOT rotated — consumer unidentified. |
| 2026-03-11 | gemini-dify-{dev,tool,team,sandbox}, gemini-evaluation | INC10735084 — scheduled rotation |
| 2026-02-26 | gemini-dify-{dev,tool,team,sandbox} | Initial key creation (per-environment split) |
