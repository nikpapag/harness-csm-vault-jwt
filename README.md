# Harness + Vault JWT Integration

Complete setup for Harness JWT authentication with Vault templated policies for dynamic secret and database credential access.


## Architecture

### Single Role, Single Policy

**Role:** `harness-pipeline`
- Uses JWT claim mappings to populate Vault entity metadata
- Claims: `environment_id`, `service_id`, `service_name`, `environment_type`

**Policy:** `harness-dynamic-access` (templated)
- Resolves paths dynamically based on JWT claims
- No need for per-service or per-environment roles

### Dynamic Path Resolution

**KV Secrets:**
```
Template: harness/kv/data/{{environment_id}}/{{service_id}}/*

Examples:
  production + payment-service → harness/kv/data/production/payment-service/*
  staging + api-service → harness/kv/data/staging/api-service/*
```

**Database Credentials:**
```
Template: database/creds/harness-readonly-{{environment_id}}

Examples:
  production → database/creds/harness-readonly-production
  staging → database/creds/harness-readonly-staging
```

### Environment-Specific Database Roles

| Role | Environment | Access | TTL | Max TTL |
|------|-------------|--------|-----|---------|
| `harness-readonly-production` | production | SELECT | 1h | 24h |
| `harness-readonly-staging` | staging | SELECT | 2h | 12h |
| `harness-readwrite-staging` | staging | SELECT, INSERT, UPDATE, DELETE | 30m | 4h |

## Configuration

### Vault Settings

- **Vault Address:** `https://vault-cluster-public-vault.hashicorp.cloud:8200`
- **Namespace:** `admin`
- **JWT Auth Path:** `harness/jwt`

### Harness Settings

- **Account ID:** `account1234543Q`
- **OIDC Discovery URL:** `https://app.harness.io/ng/api/oidc/account/account1234543Q`

### Database Settings

- **Connection:** `harness-postgres`
- **Host:** `harnessdb.rds.amazonaws.com:5432`
- **Database:** `postgres`
- **SSL:** Required

## Security Notes

1. **Credential Rotation:** Database credentials are automatically rotated with each request
2. **Time-Limited:** All credentials have TTLs (30m - 24h depending on environment)
3. **Least Privilege:** Production has readonly access; staging has optional readwrite
4. **Audit Trail:** All access tied to Harness JWT claims (user, environment, service)
5. **No Shared Credentials:** Each request generates unique database users

## Harness Custom Secret Manager Template

This repository includes a ready **Harness Custom Secret Manager template** that enables dynamic secret retrieval from HashiCorp Vault. The template supports **both KV secrets and database dynamic credentials** through a single, unified interface.

### What It Does

The Custom Secret Manager template allows Harness pipelines to:
- **Retrieve KV secrets** from Vault's Key-Value v2 secrets engine
- **Generate dynamic database credentials** (read-only or read-write) on-demand
- **Enforce environment-based access control** using JWT claims
- **Support service-scoped secrets** for multi-tenant architectures
- **Provide flexible secret extraction** (whole secret or specific keys)

All authentication and authorization is handled automatically using Harness OIDC tokens with custom claims that map to Vault templated policies.

### Architecture Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HARNESS PIPELINE                              │
│  References secret: <+secrets.getValue("my_secret_manager.db_pass")>│
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │  Custom Secret Manager   │
                    │  (Vault_Secret_Engines)  │
                    └───────────┬──────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        │    1. Get JWT Token   │   (with custom        │
        │       from Harness    │    claims)            │
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │   Harness OIDC API       │
                    │   Returns JWT with:      │
                    │   - environment_id       │
                    │   - service_id           │
                    │   - account_id           │
                    └───────────┬──────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        │    2. Authenticate    │   (JWT Auth Method)   │
        │       with Vault      │                       │
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │   HashiCorp Vault        │
                    │   - Validates JWT        │
                    │   - Maps claims to       │
                    │     entity metadata      │
                    │   - Issues Vault token   │
                    └───────────┬──────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        │    3. Evaluate        │   (Templated Policy)  │
        │       Policy          │                       │
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │   Policy Template        │
                    │   Resolves to:           │
                    │   harness/kv/data/       │
                    │     {env_id}/{svc_id}/*  │
                    │   OR                     │
                    │   database/creds/        │
                    │     harness-*-{env_id}   │
                    └───────────┬──────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        │    4. Read Secret     │   (Based on path_type)│
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │   Secret Retrieval       │
                    │                          │
                    │   PATH_TYPE = "kv"       │
                    │   → KV Secret Engine     │
                    │                          │
                    │   PATH_TYPE =            │
                    │   "db-readonly"          │
                    │   → Generate DB creds    │
                    │     (SELECT only)        │
                    │                          │
                    │   PATH_TYPE =            │
                    │   "db-readwrite"         │
                    │   → Generate DB creds    │
                    │     (full access)        │
                    └───────────┬──────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        │    5. Extract Key     │   (Optional)          │
        │       (if specified)  │                       │
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │   Return to Harness      │
                    │   Secret value stored    │
                    │   as pipeline variable   │
                    └──────────────────────────┘
```

### Script Logic Flow

The template follows this step-by-step process:

**Step 1: JWT Token Acquisition**
- Calls Harness OIDC API with custom claims
- Claims include: `account_id`, `environment_id`, `service_id`
- These claims become part of the JWT payload

**Step 2: Vault Authentication**
- Uses JWT to authenticate against Vault's `auth/harness/jwt` mount
- Vault validates JWT signature and audience
- Claim mappings populate entity metadata:
  ```
  environment_id → entity.aliases.{mount}.metadata.environment_id
  service_id → entity.aliases.{mount}.metadata.service_id
  ```

**Step 3: Path Construction**
- Based on `PATH_TYPE` environment variable:
  - `kv` → `harness/kv/data/{environment_id}/{service_id}[/{subpath}]`
  - `db-readonly` → `database/creds/harness-readonly-{environment_id}`
  - `db-readwrite` → `database/creds/harness-readwrite-{environment_id}`

**Step 4: Policy Evaluation**
- Vault evaluates templated policy against requested path
- Template variables resolve using entity metadata
- Access granted only if resolved path matches request

**Step 5: Secret Retrieval**
- For KV secrets: Returns data from Key-Value v2 engine
- For DB secrets: Generates dynamic credentials on-demand
  - Creates temporary database user
  - Assigns environment-specific role (readonly/readwrite)
  - Sets TTL for automatic revocation

**Step 6: Key Extraction (Optional)**
- If `key_name` specified: Extract specific key from secret
  - KV: `.data.data.{key_name}`
  - DB: `.data.{key_name}` (username/password)
- If not specified: Return full secret as JSON

**Step 7: Output**
- Prints secret value to stdout
- Harness captures output as secret value
- Can be referenced in pipeline as `<+secrets.getValue("identifier")>`

### Template Setup

#### 1. Create the Template in Harness

Save the following YAML as a template in your Harness account:

```yaml
template:
  name: Vault Secret Engines
  identifier: Vault_Secret_Engines
  versionLabel: v1
  type: SecretManager
  projectIdentifier: << REPLACE ME >>  # Your project identifier (e.g., "my_project")
  orgIdentifier: default
  tags: {}
  spec:
    shell: Bash
    source:
      type: Inline
      spec:
        script: |+
          # See vault-csm.sh for full script content
          # (Script is embedded inline in the template)
    delegateSelectors: []
    environmentVariables:
      - name: vault_addr
        type: String
        value: https://vault-cluster-public-vault.hashicorp.cloud:8200  # REPLACE with your Vault address
      - name: vault_namespace
        type: String
        value: admin  # REPLACE if using different namespace
      - name: env_id
        type: String
        value: <+input>  # Prompted at runtime (e.g., "prod", "staging")
      - name: service_id
        type: String
        value: <+input>  # Prompted at runtime (e.g., "api-service")
      - name: key_name
        type: String
        value: <+input>  # Prompted at runtime (e.g., "password", "api_key")
      - name: path_type
        type: String
        value: <+input>.selectOneFrom(kv,db-readonly)  # Dropdown selection
    outputVariables: []
    onDelegate: true
```

#### 2. Replace Placeholders

| Placeholder | Description | Example Value |
|-------------|-------------|---------------|
| `projectIdentifier` | Your Harness project ID | `myproject` or `default_project` |
| `<+secrets.getValue(<<REPLACE ME>>)>` | Reference to secret containing Harness API key | `<+secrets.getValue("harness_api_key")>` |
| `HARNESS_ACCOUNT_ID` | Your Harness account identifier | `abcdefg1234` |
| `vault_addr` | Your Vault cluster URL | `https://vault.company.com:8200` |
| `vault_namespace` | Vault namespace (if using Vault Enterprise) | `admin` or `production` |

**Critical:** Create a Harness secret for the API key first:
```bash
# In Harness UI: Account Settings → Secrets → New Secret
Name: harness_api_key
Type: Text
Value: pat.your-actual-harness-api-key
```

Then reference it in the template:
```bash
HARNESS_API_KEY=<+secrets.getValue("harness_api_key")>
```

#### 3. Add to Path Type Options (Optional)

To support read-write database access, modify the `path_type` environment variable:

```yaml
- name: path_type
  type: String
  value: <+input>.selectOneFrom(kv,db-readonly,db-readwrite)
```

### Using the Custom Secret Manager

#### Create a Secret Reference

1. **Navigate to:** Project → Secrets → New Secret → Secret Manager
2. **Select:** "Vault Secret Engines" template
3. **Configure Runtime Inputs:**

   **For KV Secrets:**
   ```
   env_id: production
   service_id: payment-service
   path_type: kv
   key_name: stripe_api_key
   ```

   **For Database Credentials (username):**
   ```
   env_id: staging
   service_id: api-service
   path_type: db-readonly
   key_name: username
   ```

   **For Database Credentials (password):**
   ```
   env_id: staging
   service_id: api-service
   path_type: db-readonly
   key_name: password
   ```

4. **Save with Identifier:** `my_db_password`

#### Reference in Pipeline

```yaml
pipeline:
  stages:
    - stage:
        type: Deployment
        spec:
          execution:
            steps:
              - step:
                  type: ShellScript
                  spec:
                    shell: Bash
                    script: |
                      # Access KV secret
                      STRIPE_KEY=<+secrets.getValue("stripe_api_key")>
                      
                      # Access database credentials
                      DB_USER=<+secrets.getValue("db_username")>
                      DB_PASS=<+secrets.getValue("db_password")>
                      
                      # Use in connection string
                      psql "postgresql://${DB_USER}:${DB_PASS}@host:5432/db"
```

### Supported Secret Types

#### 1. Key-Value Secrets (`path_type: kv`)

**Use Case:** API keys, connection strings, configuration values

**Vault Path Pattern:**
```
harness/kv/data/{environment_id}/{service_id}
```

**Example:**
- `env_id=production`, `service_id=payment-service`
- Resolves to: `harness/kv/data/production/payment-service`
- Returns: `{"api_key": "sk_live_123", "webhook_secret": "whsec_456"}`

**Extract Specific Key:**
- Set `key_name=api_key`
- Returns: `sk_live_123`

#### 2. Database Read-Only Credentials (`path_type: db-readonly`)

**Use Case:** Application database access, reporting queries

**Vault Path Pattern:**
```
database/creds/harness-readonly-{environment_id}
```

**Example:**
- `env_id=production`
- Resolves to: `database/creds/harness-readonly-production`
- Returns: `{"username": "v-harness-readonly-AbCd", "password": "temporary-pass-xyz"}`
- **Permissions:** `SELECT` only
- **TTL:** 1 hour (production), 2 hours (staging)

**Extract Username/Password:**
- Set `key_name=username` → Returns: `v-harness-readonly-AbCd`
- Set `key_name=password` → Returns: `temporary-pass-xyz`

#### 3. Database Read-Write Credentials (`path_type: db-readwrite`)

**Use Case:** Database migrations, admin operations

**Vault Path Pattern:**
```
database/creds/harness-readwrite-{environment_id}
```

**Example:**
- `env_id=staging`
- Resolves to: `database/creds/harness-readwrite-staging`
- Returns: `{"username": "v-harness-readwrite-XyZa", "password": "temporary-pass-abc"}`
- **Permissions:** `SELECT`, `INSERT`, `UPDATE`, `DELETE`
- **TTL:** 30 minutes
- ⚠️ **Not available for production** (enforced by policy)

### Security Features

1. **Claims-Based Authorization**
   - Secrets scoped to specific environments and services
   - Production payment-service cannot access staging api-service secrets
   - Environment isolation enforced at Vault policy level

2. **Dynamic Credentials**
   - Database users created on-demand with unique names
   - Automatic revocation after TTL expires
   - No shared credentials across pipeline runs

3. **Least Privilege**
   - Read-only access by default for databases
   - Read-write only available in non-production environments
   - Templated policies ensure path restrictions

4. **Audit Trail**
   - Every secret access logged in Vault
   - Includes JWT claims (user, environment, service)
   - Traceable to specific Harness pipeline execution

5. **No Credential Exposure**
   - Harness API key stored as Harness secret
   - Vault tokens never logged or stored
   - Database passwords never touch disk

## Vault Custom Secret Manager (vault-csm.sh)

The [vault-csm.sh](vault-csm.sh) script provides advanced secret retrieval capabilities specifically designed for Harness Custom Secret Manager integration. It offers:

### Key Features
- **Configurable Path Types**: Support for KV secrets, read-only database credentials, and read-write database credentials
- **Multiple Output Formats**: JSON, environment variables, or shell export statements
- **Key Extraction**: Extract specific keys from secrets
- **Quiet Mode**: Suppress logging for Harness CSM integration
- **Validation**: Automatic path and claims validation

### Configuration
Edit the script to configure:

```bash
# Service Context
ENVIRONMENT_ID="staging"           # prod, staging, dev, etc.
SERVICE_ID="api-service"          # Your service identifier

# Path Configuration
PATH_TYPE="kv"                    # Options: "kv", "db-readonly", "db-readwrite"
KV_SUBPATH=""                     # Optional subpath for KV secrets
KEY_NAME="database_url"           # Extract specific key (optional)

# Output Configuration
OUTPUT_FORMAT="json"              # Options: "json", "env", "shell"
QUIET=false                       # Set to true for Harness CSM usage
```

### Usage Examples

**Get full KV secret as JSON:**
```bash
./vault-csm.sh
```

**Extract specific key:**
```bash
KEY_NAME="stripe_api_key" ./vault-csm.sh
```

**Get database credentials:**
```bash
PATH_TYPE="db-readonly" ENVIRONMENT_ID="production" ./vault-csm.sh
```

**Export as environment variables:**
```bash
eval $(OUTPUT_FORMAT="shell" ./vault-csm.sh)
```

**Use in Harness CSM:**
```bash
QUIET=true KEY_NAME="password" ./vault-csm.sh
```

### Path Resolution

The script automatically constructs Vault paths based on configuration:

| PATH_TYPE | Pattern | Example |
|-----------|---------|---------|
| `kv` | `harness/kv/data/{env}/{service}[/{subpath}]` | `harness/kv/data/staging/api-service` |
| `db-readonly` | `database/creds/harness-readonly-{env}` | `database/creds/harness-readonly-production` |
| `db-readwrite` | `database/creds/harness-readwrite-{env}` | `database/creds/harness-readwrite-staging` |

### Security Notice
⚠️ The script currently contains hardcoded credentials for demonstration purposes. Before production use:
1. Remove hardcoded API keys and account IDs
2. Use environment variables or Harness secret management
3. Rotate any exposed credentials immediately

For detailed documentation, see the inline comments in [vault-csm.sh](vault-csm.sh).

---

## Files Reference

### Configuration Files
- `harness-policy.hcl` - Basic policy with sys/mounts access
- `role-config.json` - Role configuration reference
- `/tmp/harness-dynamic-policy-with-db.hcl` - Generated templated policy

### Data Files
- Test secrets created in `harness/kv/production/payment-service`
- Test secrets created in `harness/kv/staging/api-service`
