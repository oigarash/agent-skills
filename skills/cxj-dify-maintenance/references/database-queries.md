# Database Direct Access

When API returns `status: unknown` (plugin daemon issues), query the DB directly.

## Find API Pod

```bash
# dev
kubectl --context=sake-dev get pods -n cxj-dify-dev -o name | grep api | head -1

# production (tool/team/sandbox)
kubectl --context=sake get pods -n cxj-dify-{env} -o name | grep api | head -1
```

## Query Provider Credentials

```bash
kubectl --context={context} exec -n {namespace} {api-pod} -- python -c "
import psycopg2, os, json
conn = psycopg2.connect(
    host='{release}-postgresql', port='5432', dbname='dify',
    user=os.environ.get('DB_USERNAME','postgres'),
    password=os.environ.get('DB_PASSWORD'))
cur = conn.cursor()
cur.execute('SELECT provider_name, credential_name, encrypted_config FROM provider_credentials ORDER BY created_at')
for row in cur.fetchall():
    config = json.loads(row[2]) if row[2] else {}
    for k in config:
        if 'key' in k.lower() or 'secret' in k.lower():
            v = str(config[k])
            config[k] = v[:6]+'****'+v[-4:] if len(v)>8 else '****'
    print(f'provider={row[0]} name={row[1]} config={json.dumps(config, indent=2)}')
conn.close()
"
```

## Query Model-level Credentials (Azure OpenAI)

```bash
kubectl --context={context} exec -n {namespace} {api-pod} -- python -c "
import psycopg2, os, json
conn = psycopg2.connect(
    host='{release}-postgresql', port='5432', dbname='dify',
    user=os.environ.get('DB_USERNAME','postgres'),
    password=os.environ.get('DB_PASSWORD'))
cur = conn.cursor()
cur.execute('SELECT provider_name, model_name, credential_name, encrypted_config FROM provider_model_credentials ORDER BY created_at')
for row in cur.fetchall():
    config = json.loads(row[3]) if row[3] else {}
    for k in config:
        if 'key' in k.lower() or 'secret' in k.lower():
            v = str(config[k])
            config[k] = v[:6]+'****'+v[-4:] if len(v)>8 else '****'
    print(f'provider={row[0]} model={row[1]} name={row[2]} config={json.dumps(config, indent=2)}')
conn.close()
"
```

## Query Model Enable/Disable Settings

```bash
kubectl --context={context} exec -n {namespace} {api-pod} -- python -c "
import psycopg2, os
conn = psycopg2.connect(
    host='{release}-postgresql', port='5432', dbname='dify',
    user=os.environ.get('DB_USERNAME','postgres'),
    password=os.environ.get('DB_PASSWORD'))
cur = conn.cursor()
cur.execute('SELECT provider_name, model_name, model_type, enabled FROM provider_model_settings ORDER BY provider_name, model_name')
for row in cur.fetchall():
    status = 'ON' if row[3] else 'OFF'
    print(f'[{status}] {row[0]} / {row[1]} ({row[2]})')
conn.close()
"
```

## Query Workspace (Tenant) IDs

```bash
kubectl --context={context} exec -n {namespace} {api-pod} -- python -c "
import psycopg2, os
conn = psycopg2.connect(
    host='{release}-postgresql', port='5432', dbname='dify',
    user=os.environ.get('DB_USERNAME','postgres'),
    password=os.environ.get('DB_PASSWORD'))
cur = conn.cursor()
cur.execute('SELECT id, name, status FROM tenants ORDER BY created_at')
for row in cur.fetchall(): print(f'{row[0]} | {row[1]} | {row[2]}')
conn.close()
"
```

## Query Plugin Versions

```bash
kubectl --context={context} exec -n {namespace} {api-pod} -- python -c "
import psycopg2, os
conn = psycopg2.connect(
    host='{release}-postgresql', port='5432', dbname='dify',
    user=os.environ.get('DB_USERNAME','postgres'),
    password=os.environ.get('DB_PASSWORD'))
cur = conn.cursor()
cur.execute('SELECT plugin_id, plugin_unique_identifier FROM plugins ORDER BY plugin_id')
for row in cur.fetchall(): print(f'  {row[1]}')
conn.close()
"
```

## Database Schema Reference

| Table | Purpose |
|---|---|
| `provider_credentials` | Provider-level API keys (Gemini, Bedrock, Vertex AI) |
| `provider_model_credentials` | Model-level API keys (Azure OpenAI custom models) |
| `provider_model_settings` | Model enable/disable toggles |
| `providers` | Provider status tracking (is_valid, last_used) |
| `tenant_default_models` | Default model per type (LLM, embedding, etc.) |
| `tenant_preferred_model_providers` | Preferred provider type (custom vs system) |
| `plugins` | Installed plugin metadata (plugin_id, plugin_unique_identifier) |
| `plugin_installations` | Per-tenant plugin installation records |

### Key Columns

**provider_credentials**: `id`, `tenant_id`, `provider_name`, `credential_name`, `encrypted_config`

**provider_model_credentials**: `id`, `tenant_id`, `provider_name`, `model_name`, `model_type`, `credential_name`, `encrypted_config`

**provider_model_settings**: `id`, `tenant_id`, `provider_name`, `model_name`, `model_type`, `enabled`, `load_balancing_enabled`

> **Note**: `encrypted_config` values starting with `SFlCUk...` are RSA-encrypted. Plain JSON configs are readable directly. The encryption key pair is stored in the Dify API secret and can be reset with `flask reset-encrypt-key-pair` (this invalidates all stored credentials).
