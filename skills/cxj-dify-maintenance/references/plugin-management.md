# Plugin Management

## Plugin Version Check

Query the `plugins` table for installed plugin versions:

```bash
kubectl --context={context} exec -n {namespace} {api-pod} -- python -c "
import psycopg2, os
conn = psycopg2.connect(host='{release}-postgresql', port='5432', dbname='dify', user=os.environ.get('DB_USERNAME','postgres'), password=os.environ.get('DB_PASSWORD'))
cur = conn.cursor()
cur.execute('SELECT plugin_id, plugin_unique_identifier FROM plugins ORDER BY plugin_id')
for row in cur.fetchall(): print(f'  {row[1]}')
conn.close()
"
```

Plugin unique identifier format: `{org}/{name}:{version}@{sha256}`

## Install Plugin from Marketplace

```bash
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_API_KEY" \
  -H "X-WORKSPACE-ID: $WID" \
  -H "Content-Type: application/json" \
  -d '{"plugin_unique_identifiers": ["{org}/{name}:{version}@{hash}"]}' \
  "https://$HOST/console/api/workspaces/current/plugin/install/marketplace"
```

Response: `{"all_installed": false, "task_id": "..."}` — async operation.

## Upgrade Plugin from Marketplace

```bash
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_API_KEY" \
  -H "X-WORKSPACE-ID: $WID" \
  -H "Content-Type: application/json" \
  -d '{
    "original_plugin_unique_identifier": "{org}/{name}:{old_version}@{old_hash}",
    "new_plugin_unique_identifier": "{org}/{name}:{new_version}@{new_hash}"
  }' \
  "https://$HOST/console/api/workspaces/current/plugin/upgrade/marketplace"
```

## Custom Plugin (Non-Marketplace) — GitLab Hosted

Custom plugins (e.g., `ciscotac/*`) are hosted on GitLab and not available on the Dify Marketplace.
The `upgrade/marketplace` and `upgrade/github` endpoints do NOT work for these plugins.
Use the **uninstall → upload → install** workflow instead.

### Plugin Source Repository

Custom plugins are in individual repos under `gitlab-cxj.cisco.com/dify-plugins/`:

| Plugin | Repository |
|---|---|
| webex | `gitlab-cxj.cisco.com/dify-plugins/webex` |
| bdb | `gitlab-cxj.cisco.com/dify-plugins/bdb` |
| cdets | `gitlab-cxj.cisco.com/dify-plugins/cdets` |
| cs1query | `gitlab-cxj.cisco.com/dify-plugins/cs1query` |
| topic | `gitlab-cxj.cisco.com/dify-plugins/topic` |
| (others) | `gitlab-cxj.cisco.com/dify-plugins/{name}` |

Local clones are at: `~/ghq/gitlab-cxj.cisco.com/dify-plugins/{name}/`

Each repo has a `manifest.yaml` with the current version.

### Step 1: Get the .difypkg from GitLab Release

```bash
# List releases via GitLab API
curl -s --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab-cxj.cisco.com/api/v4/projects/dify-plugins%2F{name}/releases" \
  | python3 -c "import json,sys; [print(f\"{r['tag_name']}: {r.get('assets',{}).get('links',[{}])[0].get('url','N/A')}\") for r in json.load(sys.stdin)[:5]]"

# Download the .difypkg artifact
curl -s -L --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab-cxj.cisco.com/dify-plugins/{name}/-/jobs/{job_id}/artifacts/raw/public/{name}-{version}.difypkg" \
  -o /tmp/{name}-{version}.difypkg
```

> **Note**: If the GitLab artifacts are publicly accessible, `PRIVATE-TOKEN` header can be omitted.
> Verify the downloaded file: `file /tmp/{name}-{version}.difypkg` should show "Zip archive data".

### Step 2: Get the current plugin_installation_id

```bash
kubectl --context={context} exec -n {namespace} deploy/{release}-api -- python -c "
import psycopg2, os
conn = psycopg2.connect(host='{release}-postgresql', port='5432', dbname='dify', user=os.environ.get('DB_USERNAME','postgres'), password=os.environ.get('DB_PASSWORD'))
cur = conn.cursor()
cur.execute(\"SELECT pi.id FROM plugin_installations pi JOIN plugins p ON pi.plugin_id = p.plugin_id WHERE p.plugin_unique_identifier LIKE '{org}/{name}:%' AND pi.tenant_id = '{workspace_id}'\")
for row in cur.fetchall(): print(row[0])
conn.close()
"
```

### Step 3: Uninstall the old version

```bash
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_API_KEY" \
  -H "X-WORKSPACE-ID: $WID" \
  -H "Content-Type: application/json" \
  -d '{"plugin_installation_id": "{installation_id}"}' \
  "https://$HOST/console/api/workspaces/current/plugin/uninstall"
```

Response: `{"success": true}`

### Step 4: Upload the new .difypkg

```bash
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_API_KEY" \
  -H "X-WORKSPACE-ID: $WID" \
  -F "pkg=@/tmp/{name}-{version}.difypkg" \
  "https://$HOST/console/api/workspaces/current/plugin/upload/pkg"
```

Response includes `unique_identifier` — use this for the install step.

### Step 5: Install the new version

```bash
curl -s -X POST \
  -H "Authorization: Bearer $ADMIN_API_KEY" \
  -H "X-WORKSPACE-ID: $WID" \
  -H "Content-Type: application/json" \
  -d '{"plugin_unique_identifiers": ["{org}/{name}:{version}@{hash}"]}' \
  "https://$HOST/console/api/workspaces/current/plugin/install/pkg"
```

Response: `{"all_installed": false, "task_id": "..."}` — check task status to confirm.

### Alternative: Get identifier from another environment

If the target version is already installed in another environment (e.g., team has 0.0.7), you can get the `plugin_unique_identifier` from its DB and use it directly — no need to look up the GitLab Release hash.

```bash
kubectl --context={context} exec -n {namespace} deploy/{release}-api -- python -c "
import psycopg2, os
conn = psycopg2.connect(host='{release}-postgresql', port='5432', dbname='dify', user=os.environ.get('DB_USERNAME','postgres'), password=os.environ.get('DB_PASSWORD'))
cur = conn.cursor()
cur.execute(\"SELECT plugin_unique_identifier FROM plugins WHERE plugin_unique_identifier LIKE '{org}/{name}:%'\")
for row in cur.fetchall(): print(row[0])
conn.close()
"
```

## Check Task Status

```bash
curl -s \
  -H "Authorization: Bearer $ADMIN_API_KEY" \
  -H "X-WORKSPACE-ID: $WID" \
  "https://$HOST/console/api/workspaces/current/plugin/tasks/{task_id}"
```

Response: `{"task": {"status": "success|running|failed", "plugins": [{"status": "...", "message": "..."}]}}`

## Important Notes

- Plugin install/upgrade is **asynchronous**. Always check task status before proceeding.
- `base_model_name` options in Azure OpenAI depend on the plugin version. If a model name is rejected with "not in options", the plugin needs to be upgraded.
- When upgrading a plugin, use the exact `plugin_unique_identifier` from the `plugins` table as `original_plugin_unique_identifier`.
- Get the target version's identifier from an environment that already has it installed.
- **No `upgrade/pkg` endpoint exists** in Dify. For non-Marketplace plugins, use the uninstall → upload → install workflow.
- Plugin credentials are stored at the workspace level and survive uninstall/reinstall. However, verify functionality after the upgrade.

## Plugin SDK & pyproject.toml Requirements

- **SDK version**: Custom plugins must use `dify-plugin>=0.7.0,<0.8.0` (matching daemon 0.5.3+).
- **pyproject.toml format**: Must be PEP 621 (`[project]` with `dependencies`).
  Poetry format (`[tool.poetry.dependencies]`) is **silently ignored** by `uv sync`.
- **requirements.txt**: Also update SDK version here as fallback.
- After changing `pyproject.toml`, regenerate `uv.lock` with `uv lock`.

## Known API Issue: install/pkg Returns 500

The `POST /plugin/install/pkg` endpoint calls `decode/from_identifier` internally,
which fails due to a query parameter name mismatch (snake_case vs PascalCase).
**Workaround**: Use daemon's `install/identifiers` endpoint directly. Ensure
`source` is set to `"package"` (not `"pkg"`) to avoid PluginListResponse parse errors.

## DB Tables

| Table | Purpose |
|---|---|
| `plugins` | Installed plugin metadata (plugin_id, plugin_unique_identifier) |
| `plugin_installations` | Per-tenant plugin installation records |
| `plugin_declarations` | Plugin manifest/declaration cache |
