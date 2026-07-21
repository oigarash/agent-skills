# Gemini / GCP API Key Management

## GCP Project

- **Project**: CX Centers Japan GenAI (`gcp-cxcentersjapang-prd-48251`)
- **Enabled APIs**: `aiplatform.googleapis.com`, `generativelanguage.googleapis.com`
- **Service Account**: `vertex-ai-runner` — **DELETED 2026-06-15** (Vertex AI no
  longer used; migrated to the Gemini API. See `gcp-sa-key-rotation.md`.)

## API Keys (as of 2026-06-15)

| Key Name | Environment | UID |
|---|---|---|
| `gemini-dify-dev` | DEV | db3f7d2c-5051-4f96-94ef-04a8272ba69a |
| `gemini-dify-team` | TEAM | f72ffac3-1873-442a-b514-dc30adc26c47 |
| `gemini-dify-tool` | TOOL | 0aed3ba7-b5f6-44a2-a60a-aea4850e4aa3 |
| `gemini-dify-sandbox` | SANDBOX | b6ba779b-9f71-40f7-9dc5-5a0bc68ca338 |
| `gemini-evaluation` | Evaluation (unused by Dify) | c1886ff0-0569-4c4c-92c7-3b1475e66fbb |

All keys are restricted to `generativelanguage.googleapis.com` + `aiplatform.googleapis.com`.

> `gemini-dify-{dev,team,tool,sandbox}` were rotated 2026-06-15 (≈96d old).
> `gemini-evaluation` was NOT rotated (consumer not identified — its key is not
> wired into any Dify provider). Identify its consumer before rotating.

## Create a New API Key

```bash
gcloud services api-keys create \
  --project=gcp-cxcentersjapang-prd-48251 \
  --display-name="gemini-dify-{env}" \
  --api-target=service=generativelanguage.googleapis.com \
  --api-target=service=aiplatform.googleapis.com \
  --format="value(response.keyString)"
```

## Rename an API Key

```bash
gcloud services api-keys update \
  $(gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 --format="value(name)" --filter="displayName={old_name}") \
  --display-name="{new_name}"
```

## List API Keys

```bash
gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 \
  --format="table(displayName,uid,restrictions.apiTargets.service)"
```

## Get Key String

```bash
gcloud services api-keys get-key-string --project=gcp-cxcentersjapang-prd-48251 \
  $(gcloud services api-keys list --project=gcp-cxcentersjapang-prd-48251 --format="value(name)" --filter="displayName={key_name}")
```

## Test Key with Gemini API

```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models?key={API_KEY}" | \
  python3 -c "import sys,json; [print(m['name']) for m in json.load(sys.stdin).get('models',[])]"
```

## Gemini Plugin Identifier (latest known)

- `langgenius/gemini:0.7.4@5faf411e694e0da130a337347a625a3812b1b7502a0183382d66d92c841bf05a`

## End-to-end: Add Gemini to a New Environment

1. **Create GCP API key**: `gcloud services api-keys create ...`
2. **Install Gemini plugin**: POST to `/plugin/install/marketplace` (check task status)
3. **Register credentials**: POST to `/model-providers/langgenius%2Fgemini%2Fgoogle/credentials`
4. **Enable target models**: PATCH to `/model-providers/langgenius%2Fgemini%2Fgoogle/models/enable`
5. **Disable unwanted models**: PATCH to `/model-providers/langgenius%2Fgemini%2Fgoogle/models/disable`
