# Troubleshooting

## API returns status: unknown for all providers

The plugin daemon cannot communicate with providers. Check:

```bash
kubectl --context={context} -n {namespace} logs deploy/{release}-plugin-daemon --tail=50
kubectl --context={context} -n {namespace} exec deploy/{release}-api -- curl -s http://localhost:5001/health
```

## ADMIN_API_KEY not working (401 Unauthorized)

1. Verify the key is set on the deployment:
   ```bash
   kubectl --context={context} get deployment {release}-api -n {namespace} \
     -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}' | grep ADMIN
   ```
2. Ensure `X-WORKSPACE-ID` header is included (required for admin key auth)
3. Verify workspace ID exists in the database

## Azure OpenAI DeploymentNotFound when registering model

Dify validates credentials by calling the Azure endpoint during registration. If the Azure deployment was just created, the endpoint may return 404 for 3-5 minutes.

**Solution**: Wait and retry. Verify from a Pod in the cluster (see `references/azure-openai-resources.md` for the connectivity check command).

## Azure OpenAI endpoint URL format

Use `.openai.azure.com` (NOT `.cognitiveservices.azure.com`):
- Correct: `https://cxj-dify-restricted.openai.azure.com/openai/deployments/{name}/chat/completions?api-version=...`
- Wrong: `https://cxj-dify-restricted.cognitiveservices.azure.com/...` (DNS fails from pods)

## Credentials encrypted vs plain text

If `encrypted_config` contains JSON starting with `{`, it is plain text. If it starts with `SFlCUk` (base64), it is RSA-encrypted. After Dify upgrade or DB restore, credentials may need re-encryption via the UI or by resetting the encrypt key pair.

## Plugin version too old for new models

If registering a model returns `"Variable base_model_name is not in options"`, upgrade the provider plugin. Check current version in DB, find the latest version from another environment, then use the upgrade marketplace API. See `references/plugin-management.md`.

## Provider credentials exist but models not visible

If `current_credential_id` is `None` in the model-providers API response, the credential is not linked as active. **Delete and re-create the credential** to fix this. Simply creating a new credential without deleting the orphan does NOT resolve it.

## DELETE API requires Content-Type header

All Dify Admin API mutations (including DELETE) require `Content-Type: application/json`. Without it, you get `415 Unsupported Media Type`. DELETE also requires `credential_id` in the request body.

## Custom plugin "no available node, plugin runtime not found" after upgrade

After upgrading Dify (especially to 1.13+), custom plugins may fail with
`PluginDaemonInternalServerError: no available node, plugin runtime not found`.

**Root causes:**

1. **pyproject.toml in Poetry format**: The plugin daemon uses `uv sync` which only
   reads PEP 621 `[project.dependencies]`, not Poetry's `[tool.poetry.dependencies]`.
   The venv ends up empty and `dify_plugin` module is not found.

2. **SDK version constraint too restrictive**: Plugins with `dify_plugin~=0.4.3` or
   `<0.7.0` constraints cannot install `dify_plugin>=0.7.0` which daemon 0.5.3+ requires.

3. **`install/pkg` API broken (API↔daemon param mismatch)**: The `decode/from_identifier`
   endpoint expects `PluginUniqueIdentifier` (PascalCase) but the API sends
   `plugin_unique_identifier` (snake_case). Use daemon's `install/identifiers` endpoint
   directly as a workaround.

**Fix checklist:**

- Convert `pyproject.toml` from Poetry to PEP 621 format (`[project]` + `dependencies`)
- Update SDK version to `dify-plugin>=0.7.0,<0.8.0` in both `pyproject.toml` and `requirements.txt`
- Regenerate `uv.lock` with `uv lock`
- Repackage with `dify plugin package`
- Install via daemon API directly:
  1. Upload: `POST /console/api/workspaces/current/plugin/upload/pkg`
  2. Install: call daemon `POST /plugin/{tenant}/management/install/identifiers`
     with `source: "package"` (not `"pkg"`)

## Known deployment name typos

- TOOL: `text-embedding-3-large-sandobx` (should be `-sandbox` or `-tool`)
- TEAM: `gpt-4.1-nano-tool` (should be `-team`)
