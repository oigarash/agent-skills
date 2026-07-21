# GCP Service Account Key Rotation

## Service Account

- **Name**: `vertex-ai-runner`
- **Email**: `vertex-ai-runner@gcp-cxcentersjapang-prd-48251.iam.gserviceaccount.com`
- **Role**: `roles/aiplatform.expressUser` (Vertex AI)
- **Project**: `gcp-cxcentersjapang-prd-48251`

## Key Types

- **SYSTEM_MANAGED**: Auto-rotated by GCP with short expiry — no action needed
- **USER_MANAGED**: Created manually, often without expiry — rotation target

## Audit

```bash
gcloud iam service-accounts keys list \
  --iam-account=vertex-ai-runner@gcp-cxcentersjapang-prd-48251.iam.gserviceaccount.com \
  --project=gcp-cxcentersjapang-prd-48251 \
  --format="table(name.basename(),keyAlgorithm,keyType,validAfterTime,validBeforeTime,disabled)"
```

Flag any USER_MANAGED key older than 90 days or with `validBeforeTime: 9999-12-31`.

## Rotation Procedure

### Step 1: Create new key

```bash
gcloud iam service-accounts keys create /tmp/vertex-ai-runner-new.json \
  --iam-account=vertex-ai-runner@gcp-cxcentersjapang-prd-48251.iam.gserviceaccount.com \
  --project=gcp-cxcentersjapang-prd-48251
```

### Step 2: Encode for deployment

```bash
base64 -i /tmp/vertex-ai-runner-new.json | tr -d '\n'
```

### Step 3: Update consumers

This key may be used in:

- **Dify Vertex AI provider**: Update via Admin API
  ```bash
  curl -s -X PUT \
    "https://{host}/console/api/workspaces/current/model-providers/langgenius%2Fvertex_ai%2Fvertex_ai/credentials" \
    -H "Authorization: Bearer {ADMIN_API_KEY}" \
    -H "X-WORKSPACE-ID: {workspace_id}" \
    -H "Content-Type: application/json" \
    -d '{
      "credential_id": "{cred_id}",
      "credentials": {
        "vertex_service_account_key": "{base64_key}",
        "vertex_project_id": "gcp-cxcentersjapang-prd-48251",
        "vertex_location": "asia-northeast1"
      },
      "name": "API_KEY1"
    }'
  ```

- **GCS storage** (if `STORAGE_TYPE=google-storage`):
  Update `GOOGLE_STORAGE_SERVICE_ACCOUNT_JSON_BASE64` in the environment's
  Helm values or K8s secret, then redeploy.

- **External consumers**: Check if any other services reference this SA.

### Step 4: Verify

Confirm the new key works for its consumers (Vertex AI model call, GCS upload, etc.).

### Step 5: Disable and delete old key

```bash
# Disable first (allows rollback if issues found)
gcloud iam service-accounts keys disable {OLD_KEY_ID} \
  --iam-account=vertex-ai-runner@gcp-cxcentersjapang-prd-48251.iam.gserviceaccount.com \
  --project=gcp-cxcentersjapang-prd-48251

# After confirming no issues, delete
gcloud iam service-accounts keys delete {OLD_KEY_ID} \
  --iam-account=vertex-ai-runner@gcp-cxcentersjapang-prd-48251.iam.gserviceaccount.com \
  --project=gcp-cxcentersjapang-prd-48251 --quiet
```

### Step 6: Clean up

```bash
rm /tmp/vertex-ai-runner-new.json
```

## Current State (as of 2026-06-15) — SA DECOMMISSIONED

**The `vertex-ai-runner` service account has been DELETED** (2026-06-15), along
with all its keys. This entire procedure is now historical reference only.

Background: a Wiz/S&TO incident (CIS GCP Benchmark v4 Rec 1.7, key age >180d)
flagged the USER_MANAGED key `5ab094db...` (created 2025-09-11, no expiry). The
key was confirmed **unused** — all cxj-dify environments (dev/tool/team/sandbox)
use the **Gemini API** provider (API key) + Azure OpenAI, none configure the
Vertex AI provider, and no env uses GCS storage (`STORAGE_TYPE` unset). The SA's
only role was `roles/aiplatform.expressUser` (Vertex AI only), which is no longer
used. So the key was deleted first, then the SA itself was decommissioned.

> **Recovery window**: a deleted SA can be undeleted within ~30 days via
> `gcloud iam service-accounts undelete <uniqueId>`.
> `vertex-ai-runner` uniqueId = `104508723178597538011`.
>
> **If Vertex AI is re-adopted**, prefer Workload Identity Federation over a new
> USER_MANAGED key. If a key is unavoidable, recreate the SA and follow the
> Rotation Procedure above, and set a key expiry.
