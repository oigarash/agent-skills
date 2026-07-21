# Azure OpenAI Resource Management

Azure subscription: **Japan CX Centers GenAI** (`a809bb53-16fd-4aab-83d9-d6d71a9ce045`)

```bash
az account set --subscription "a809bb53-16fd-4aab-83d9-d6d71a9ce045"
```

## Azure OpenAI Resources

| Resource | Location | Resource Group | Purpose |
|---|---|---|---|
| `cxj-dify-restricted` | Japan East | cxj-dify-restricted | Main: gpt-4.1, gpt-4.1-mini, gpt-5-nano, embeddings per env |
| `cxj-dify-restricted-useast2` | East US 2 | cxj-dify-restricted | gpt-5.2 (models not available in Japan East) |
| `cjx-dify-genai` | Japan East | cxj-dify | gpt-4o, gpt-4o-mini, o1, embeddings (older models) |
| `cxj-dify-us` | East US | cxj-dify | o1-mini (o4-mini), training/guest deployments |
| `restrict-ip` | Japan East | openai-eval-rg | Evaluation use |

> **Note**: `cxj-dify-restricted` and `cxj-dify-restricted-useast2` have VNet restrictions. Connections must come from whitelisted IPs (Utena cluster pods are whitelisted).

## Deployment Naming Convention

Pattern: `{model}-{env}` (e.g., `gpt-5.2-team`, `gpt-4.1-mini-tool`, `text-embedding-ada-002-dev`)

## Create a New Azure Deployment

```bash
az cognitiveservices account deployment create \
  -g {resource-group} \
  -n {resource-name} \
  --deployment-name {model}-{env} \
  --model-name {model} \
  --model-version "{version}" \
  --model-format OpenAI \
  --sku-name GlobalStandard \
  --sku-capacity 250
```

## Get API Key for a Resource

```bash
az cognitiveservices account keys list -g {rg} -n {resource} --query "key1" -o tsv
```

## List Deployments

```bash
az cognitiveservices account deployment list -g {rg} -n {resource} \
  --query "[].{name:name, model:properties.model.name, version:properties.model.version, capacity:sku.capacity}" -o table
```

> **IMPORTANT**: After creating a new Azure GlobalStandard deployment, wait **3-5 minutes** for propagation before registering in Dify. The Azure API returns `DeploymentNotFound` (404) during this period.

## End-to-end Workflow: Add a New Model to an Environment

1. **Create Azure deployment**: `az cognitiveservices account deployment create ...`
2. **Wait 3-5 min** for Azure propagation
3. **Get API key**: `az cognitiveservices account keys list ...`
4. **Verify connectivity from pod** (optional but recommended):
   ```bash
   kubectl --context={context} exec -n {namespace} {api-pod} -- python -c "
   import httpx
   resp = httpx.post('{azure_endpoint}',
       json={'messages':[{'role':'user','content':'hi'}],'max_completion_tokens':5},
       headers={'api-key': '{key}'}, timeout=30)
   print(f'status={resp.status_code}')
   print(resp.text[:200])
   "
   ```
   When it returns 200 or 400 (not 404), the deployment is ready.
5. **Register in Dify**: POST to `/models/credentials` (see `references/credential-schemas.md`)
6. **Verify**: Check via model-providers API or Dify Web UI

## Known Deployment Name Typos

- TOOL: `text-embedding-3-large-sandobx` (should be `-sandbox` or `-tool`)
- TEAM: `gpt-4.1-nano-tool` (should be `-team`)
