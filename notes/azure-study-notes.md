# Azure Complete — AI-Native Backend Developer Reference (April 2026)

> **Scope:** Everything you need to build production FastAPI backends on Azure as of April 2026.
> Forward-looking only — deprecated patterns (App Service for new builds, ARM templates, Single Server PostgreSQL, service principal key files) are called out and avoided.
>
> **Companion rules:** `~/.claude/rules/azure-patterns.md`

---

## Part 1: Azure Mental Model

### The Hierarchy Analogy

Think of Azure's structure like a **university system**:

- **Management Group** = the university system (e.g. "University of California") — governs policy across all campuses
- **Subscription** = a single campus (UCLA, Berkeley) — has its own budget, billing, and resource limits
- **Resource Group** = a department within a campus (CS Department, Engineering) — logical container for related resources
- **Resource** = an individual asset (a server room, a lab) — the actual thing you're paying for

Unlike AWS where an "account" is the isolation boundary and you typically have one account per team, Azure teams often share a subscription and use **Resource Groups** as the isolation unit. This is both a blessing (simpler billing) and a curse (misconfigured IAM can leak across resource groups).

```
Tenant (Entra ID)
└── Management Group: "Engineering"
    ├── Management Group: "Production"
    │   └── Subscription: "prod-services"
    │       ├── Resource Group: "rg-api-prod-eastus"
    │       │   ├── Container App: "api"
    │       │   ├── PostgreSQL Flexible Server: "db-prod"
    │       │   └── Key Vault: "kv-prod"
    │       └── Resource Group: "rg-ai-prod-eastus"
    │           └── Azure OpenAI: "aoai-prod"
    └── Subscription: "dev-services"
        └── Resource Group: "rg-api-dev-eastus"
```

### Entra ID Tenant vs Subscription — The Confusion

This trips up every AWS developer coming to Azure:

- **Entra ID Tenant** = your organization's identity directory. One tenant, multiple subscriptions. It owns users, groups, service principals, managed identities, and app registrations.
- **Subscription** = a billing and resource container. It's associated with exactly one tenant for identity purposes.

The mental model: Entra ID is *who you are*, subscription is *what you can spend money on*.

**AWS/GCP equivalents:**
| Azure | AWS | GCP |
|---|---|---|
| Subscription | Account | Project |
| Resource Group | (no direct equivalent — use tags or separate accounts) | Folder within project |
| Management Group | AWS Organizations OU | Folder in Resource Manager |
| Entra ID Tenant | AWS Organizations root account's IAM | Cloud Identity / Google Workspace |
| Managed Identity | IAM Role for EC2/ECS (instance profile) | Service Account |
| Key Vault | AWS Secrets Manager / KMS | Secret Manager / KMS |
| Container Apps | ECS Fargate / App Runner | Cloud Run |
| Azure OpenAI | Amazon Bedrock | Vertex AI |

### Azure Resource Manager (ARM) — Everything Is an API

Every Azure operation — creating a VM, adding a firewall rule, reading a secret — goes through the **Azure Resource Manager (ARM)** API at `management.azure.com`. This is both `az` CLI commands and the Azure Portal click-ops.

The implications:
- Every resource has a canonical **Resource ID**: `/subscriptions/{subId}/resourceGroups/{rg}/providers/{provider}/{type}/{name}`
- All resources support **RBAC at any scope** — assign a role at management group level, it flows down
- **Bicep/ARM templates** are just structured calls to this same API
- **Managed Identity** works because a compute resource can call ARM on your behalf using its own identity

This uniformity is Azure's superpower — once you understand the ARM model, every service follows the same pattern for access control, deployment, and querying.

---

## Part 2: az CLI (April 2026)

### Installation

```bash
# macOS
brew install azure-cli

# Linux (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verify
az version  # Should be 2.x
```

### Authentication

```bash
# Interactive browser login (development)
az login

# Login to a specific tenant
az login --tenant your-tenant-id.onmicrosoft.com

# Device code flow (headless servers)
az login --use-device-code

# Managed Identity login (from inside Azure compute — no password needed)
az login --identity

# Service principal login (CI/CD — prefer Workload Identity Federation instead)
az login --service-principal \
  --username $AZURE_CLIENT_ID \
  --password $AZURE_CLIENT_SECRET \
  --tenant $AZURE_TENANT_ID
```

### Account / Subscription Management

```bash
# List all subscriptions you have access to
az account list --output table

# Set active subscription
az account set --subscription "prod-services"
# or by ID
az account set --subscription "12345678-1234-1234-1234-123456789012"

# Show current context
az account show

# Named profiles don't exist natively in az CLI — use environment variables
# or the --subscription flag per command for multi-subscription workflows
export AZURE_SUBSCRIPTION_ID="12345678-..."
```

### JMESPath Queries with --query

`--query` uses JMESPath, a JSON path language. This is the most powerful az CLI feature you're not using:

```bash
# Get just the name and state of all Container Apps in a resource group
az containerapp list \
  --resource-group rg-api-prod \
  --query "[].{name:name, state:properties.runningStatus}" \
  --output table

# Get the FQDN of a specific Container App
az containerapp show \
  --name my-api \
  --resource-group rg-api-prod \
  --query "properties.configuration.ingress.fqdn" \
  --output tsv

# Filter resources by tag
az resource list \
  --tag environment=production \
  --query "[?type=='Microsoft.App/containerApps'].name" \
  --output json

# Get a secret value from Key Vault
az keyvault secret show \
  --vault-name kv-prod \
  --name database-url \
  --query "value" \
  --output tsv

# List Container App revisions sorted by creation time
az containerapp revision list \
  --name my-api \
  --resource-group rg-api-prod \
  --query "sort_by([].{name:name, active:properties.active, created:properties.createdTime}, &created)" \
  --output table
```

### Output Formats

```bash
--output json    # Full JSON (default)
--output yaml    # YAML (readable for complex objects)
--output table   # ASCII table (human readable)
--output tsv     # Tab-separated (for shell scripting)
--output none    # Suppress output (for create/update commands)
```

### Key Command Groups

```bash
# Container Apps
az containerapp --help
az containerapp up              # Build and deploy from source (fastest path)
az containerapp create          # Create from existing image
az containerapp update          # Update env vars, image, scale rules
az containerapp revision list   # List all revisions
az containerapp logs show       # Stream logs

# Azure OpenAI
az cognitiveservices account list --kind OpenAI
az cognitiveservices account deployment list \
  --name aoai-prod \
  --resource-group rg-ai-prod
az cognitiveservices account deployment create \
  --name aoai-prod \
  --resource-group rg-ai-prod \
  --deployment-name gpt-4o \
  --model-name gpt-4o \
  --model-version "2024-11-20" \
  --model-format OpenAI \
  --sku-capacity 10 \
  --sku-name "Standard"

# Key Vault
az keyvault create --name kv-prod --resource-group rg-prod --location eastus
az keyvault secret set --vault-name kv-prod --name my-secret --value "value"
az keyvault secret show --vault-name kv-prod --name my-secret

# Storage
az storage account create ...
az storage blob upload ...
az storage blob generate-sas ...  # For SAS tokens

# PostgreSQL Flexible Server
az postgres flexible-server create ...
az postgres flexible-server db create ...
az postgres flexible-server firewall-rule create ...
```

### Extensions for Preview Features

```bash
# List installed extensions
az extension list --output table

# Add the Container Apps extension (usually pre-bundled now)
az extension add --name containerapp

# Upgrade all extensions
az extension update --all

# Add the AI extension (for Azure AI Foundry CLI operations)
az extension add --name ai
```

### Deploying Bicep

```bash
# What-if deployment (preview changes without applying)
az deployment group what-if \
  --resource-group rg-api-prod \
  --template-file infra/main.bicep \
  --parameters @infra/main.prod.bicepparam

# Apply deployment
az deployment group create \
  --resource-group rg-api-prod \
  --template-file infra/main.bicep \
  --parameters @infra/main.prod.bicepparam \
  --name "deploy-$(date +%Y%m%d-%H%M%S)"

# View deployment outputs
az deployment group show \
  --resource-group rg-api-prod \
  --name deploy-20260414-120000 \
  --query "properties.outputs"
```

---

## Part 3: Compute — Azure Container Apps (Primary)

### The Mental Model: Container Apps as a Platform

Think of Azure Container Apps as **a managed Kubernetes cluster you never see**. The infrastructure (Kubernetes, Envoy ingress, KEDA for scaling) is all there, but you interact with a much simpler abstraction:

```
Container Apps Environment  (like a VNet + shared K8s cluster)
├── Container App: "api"            (like a Deployment)
│   ├── Revision: "api--abc123"     (like a ReplicaSet)
│   │   └── Replicas: 0-10         (like Pods)
│   └── Revision: "api--def456"    (old, 0% traffic)
├── Container App: "worker"
└── Container App: "scheduler"
```

**Key concepts:**

| Concept | Description | K8s Equivalent |
|---|---|---|
| Environment | Shared network + Log Analytics workspace | Namespace/Cluster |
| Container App | The logical application | Deployment |
| Revision | Immutable snapshot of an app config | ReplicaSet |
| Replica | A running instance | Pod |
| Ingress | HTTP/HTTPS routing config | Ingress + Service |

### Creating a Container Apps Environment

```bash
# Create environment with Log Analytics
az containerapp env create \
  --name cae-prod-eastus \
  --resource-group rg-api-prod \
  --location eastus \
  --logs-workspace-id $(az monitor log-analytics workspace show \
      --workspace-name law-prod \
      --resource-group rg-monitoring \
      --query customerId --output tsv) \
  --logs-workspace-key $(az monitor log-analytics workspace get-shared-keys \
      --workspace-name law-prod \
      --resource-group rg-monitoring \
      --query primarySharedKey --output tsv)
```

### Deploying a Container App

```bash
# Fast path: deploy from source (az containerapp up builds with Buildpacks)
az containerapp up \
  --name my-api \
  --resource-group rg-api-prod \
  --environment cae-prod-eastus \
  --source . \
  --target-port 8000 \
  --ingress external

# From an existing image
az containerapp create \
  --name my-api \
  --resource-group rg-api-prod \
  --environment cae-prod-eastus \
  --image myregistry.azurecr.io/my-api:v1.2.3 \
  --target-port 8000 \
  --ingress external \
  --cpu 0.5 \
  --memory 1.0Gi \
  --min-replicas 1 \
  --max-replicas 10 \
  --env-vars \
    "ENVIRONMENT=production" \
    "DATABASE_URL=secretref:database-url" \
  --secrets \
    "database-url=keyvaultref:https://kv-prod.vault.azure.net/secrets/database-url,identityref:system"
```

### Scaling Rules

Container Apps scales via **KEDA** (Kubernetes Event-Driven Autoscaling) under the hood. You configure scaling rules declaratively:

```bash
# HTTP-based scaling (most common for APIs)
az containerapp update \
  --name my-api \
  --resource-group rg-api-prod \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 50 \
  --min-replicas 1 \
  --max-replicas 20

# CPU-based scaling
az containerapp update \
  --name my-worker \
  --resource-group rg-api-prod \
  --scale-rule-name cpu-rule \
  --scale-rule-type cpu \
  --scale-rule-metadata type=Utilization value=70 \
  --min-replicas 1 \
  --max-replicas 10

# Azure Service Bus queue-based scaling (KEDA custom trigger)
az containerapp update \
  --name my-worker \
  --resource-group rg-api-prod \
  --scale-rule-name sb-rule \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata \
    queueName=my-queue \
    messageCount=5 \
    namespace=my-servicebus \
  --scale-rule-auth \
    "connection=servicebus-connection-string" \
  --min-replicas 0 \
  --max-replicas 20
```

Scaling to **zero** (`--min-replicas 0`) is fully supported — great for workers. For APIs, keep at least 1 to avoid cold start latency on the critical path.

### Managed Identity on Container Apps

```bash
# Enable system-assigned managed identity
az containerapp identity assign \
  --name my-api \
  --resource-group rg-api-prod \
  --system-assigned

# Get the principal ID of the identity
PRINCIPAL_ID=$(az containerapp show \
  --name my-api \
  --resource-group rg-api-prod \
  --query "identity.principalId" \
  --output tsv)

# Grant Key Vault access via RBAC
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope $(az keyvault show --name kv-prod --resource-group rg-prod --query id --output tsv)

# Grant Azure OpenAI access
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Cognitive Services OpenAI User" \
  --scope $(az cognitiveservices account show --name aoai-prod --resource-group rg-ai --query id --output tsv)

# Grant PostgreSQL Flexible Server access (Entra auth)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Contributor" \
  --scope /subscriptions/$SUB_ID/resourceGroups/rg-db/providers/Microsoft.DBforPostgreSQL/flexibleServers/db-prod
```

### Secrets from Key Vault (Key Vault References)

The recommended pattern: Container Apps pulls secrets from Key Vault at startup — your code never sees the Key Vault URL, just an environment variable.

```bash
# Secret using Key Vault reference with system-assigned identity
az containerapp secret set \
  --name my-api \
  --resource-group rg-api-prod \
  --secrets \
    "database-url=keyvaultref:https://kv-prod.vault.azure.net/secrets/database-url,identityref:system" \
    "redis-url=keyvaultref:https://kv-prod.vault.azure.net/secrets/redis-url,identityref:system"

# Reference the secret in an env var
az containerapp update \
  --name my-api \
  --resource-group rg-api-prod \
  --set-env-vars \
    "DATABASE_URL=secretref:database-url" \
    "REDIS_URL=secretref:redis-url"
```

In Python, your code just reads `os.environ["DATABASE_URL"]` — no Azure SDK needed.

### Container Apps Jobs

For batch workloads, migrations, cron tasks:

```bash
# Manual trigger job (run on demand)
az containerapp job create \
  --name db-migration-job \
  --resource-group rg-api-prod \
  --environment cae-prod-eastus \
  --trigger-type Manual \
  --image myregistry.azurecr.io/my-api:v1.2.3 \
  --command "python" \
  --args "-m" "alembic" "upgrade" "head" \
  --cpu 0.5 \
  --memory 1.0Gi

# Cron job
az containerapp job create \
  --name cleanup-job \
  --resource-group rg-api-prod \
  --environment cae-prod-eastus \
  --trigger-type Schedule \
  --cron-expression "0 2 * * *" \
  --image myregistry.azurecr.io/my-api:v1.2.3 \
  --command "python" \
  --args "-m" "app.tasks.cleanup"

# Run a job execution
az containerapp job start \
  --name db-migration-job \
  --resource-group rg-api-prod
```

### Ingress Configuration and Traffic Splitting

```bash
# External ingress (internet-facing)
az containerapp ingress enable \
  --name my-api \
  --resource-group rg-api-prod \
  --type external \
  --target-port 8000 \
  --transport auto  # auto detects HTTP/1.1 vs HTTP/2

# Internal ingress (only within the environment)
az containerapp ingress enable \
  --name my-worker-api \
  --resource-group rg-api-prod \
  --type internal \
  --target-port 8000

# Blue/green traffic splitting: send 10% to new revision
az containerapp ingress traffic set \
  --name my-api \
  --resource-group rg-api-prod \
  --revision-weight \
    "my-api--v2=10" \
    "my-api--v1=90"

# Complete cutover after validation
az containerapp ingress traffic set \
  --name my-api \
  --resource-group rg-api-prod \
  --revision-weight "latest=100"
```

### ACA vs AKS vs App Service (April 2026 Decision Guide)

| Criterion | Container Apps | AKS | App Service |
|---|---|---|---|
| **Use case** | Most containerized workloads | Complex custom K8s, operators, service mesh control | Legacy lift-and-shift (not recommended for new) |
| **Complexity** | Low | High | Low |
| **Managed** | Fully managed | Managed control plane, you manage nodes | Fully managed |
| **Scaling to zero** | Yes | Yes (via KEDA) | No (Basic/Standard tiers) |
| **Custom networking** | Yes (VNet injection) | Yes (full control) | Limited |
| **Dapr** | Built-in | Via Helm | No |
| **GPU workloads** | Limited (April 2026) | Yes | No |
| **Long-running processes** | Yes | Yes | Yes |
| **Cost (idle)** | Near zero (scale to 0) | Node VM cost even at 0 traffic | VM cost even at 0 traffic |
| **When to choose** | Default for new APIs, workers, batch | Multi-team microservices, custom operators, GPU, compliance-requiring node control | Never for new builds |

**Rule of thumb:** Use Container Apps unless you need direct Kubernetes API access, custom node pools, or specific operators.

### Dapr Integration (Optional)

Dapr (Distributed Application Runtime) provides building blocks for microservices — service invocation, state management, pub/sub, secret store — without code changes to your app.

```bash
# Enable Dapr on an environment
az containerapp env dapr-component set \
  --name cae-prod-eastus \
  --resource-group rg-api-prod \
  --dapr-component-name statestore \
  --yaml - <<EOF
componentType: state.azure.cosmosdb
version: v1
metadata:
  - name: url
    value: https://cosmos-prod.documents.azure.com:443/
  - name: database
    value: mydb
  - name: collection
    value: state
  - name: masterKey
    secretRef: cosmos-key
scopes:
  - my-api
EOF

# Enable Dapr on a Container App
az containerapp dapr enable \
  --name my-api \
  --resource-group rg-api-prod \
  --dapr-app-id my-api \
  --dapr-app-port 8000 \
  --dapr-app-protocol http
```

---

## Part 4: Database Selection (April 2026)

### Decision Tree

```
What are your data access patterns?
├── Relational data, complex queries, JOINs, ACID transactions
│   ├── Single-region, <500GB → Azure Database for PostgreSQL Flexible Server
│   └── Multi-region, >500GB, multi-tenant SaaS sharding → Cosmos DB for PostgreSQL (Citus)
├── Document/NoSQL, global distribution, variable schema
│   ├── <5 RU/s average, unpredictable traffic → Cosmos DB Serverless
│   └── Consistent high throughput → Cosmos DB Provisioned
└── Vector similarity search
    ├── Already using PostgreSQL → pgvector on Flexible Server
    └── Pure vector + document store → Cosmos DB NoSQL with DiskANN vectors (April 2026)
```

### Azure Database for PostgreSQL Flexible Server

**The standard choice** for relational data. Replaces the deprecated Single Server (which reached end-of-life March 2025).

```bash
# Create Flexible Server (zone-redundant HA)
az postgres flexible-server create \
  --name db-prod-eastus \
  --resource-group rg-db-prod \
  --location eastus \
  --admin-user dbadmin \
  --admin-password $(az keyvault secret show --vault-name kv-prod --name db-admin-password --query value -o tsv) \
  --sku-name Standard_D2ds_v5 \
  --tier GeneralPurpose \
  --storage-size 128 \
  --version 16 \
  --high-availability ZoneRedundant \
  --standby-zone 2 \
  --backup-retention 30 \
  --public-access None \
  --vnet /subscriptions/.../resourceGroups/rg-network/providers/Microsoft.Network/virtualNetworks/vnet-prod \
  --subnet /subscriptions/.../resourceGroups/rg-network/providers/Microsoft.Network/virtualNetworks/vnet-prod/subnets/snet-db

# Enable Entra ID authentication (for Managed Identity access)
az postgres flexible-server ad-admin set \
  --server-name db-prod-eastus \
  --resource-group rg-db-prod \
  --display-name "AzureAD Admin" \
  --object-id $(az ad user show --id admin@company.com --query id -o tsv)

# Enable pgvector extension
az postgres flexible-server parameter set \
  --server-name db-prod-eastus \
  --resource-group rg-db-prod \
  --name azure.extensions \
  --value "VECTOR,PG_TRGM,BTREE_GIN"

# Create application database
az postgres flexible-server db create \
  --server-name db-prod-eastus \
  --resource-group rg-db-prod \
  --database-name appdb
```

**Built-in PgBouncer connection pooling:**

```bash
# PgBouncer is built into Flexible Server on port 6432
# Connect via PgBouncer (transaction pooling mode)
# host: db-prod-eastus.postgres.database.azure.com
# port: 6432  (PgBouncer)
# port: 5432  (direct — for DDL, advisory locks, LISTEN/NOTIFY)
```

**Python asyncpg with Managed Identity:**

```python
import asyncpg
import struct
import time
from azure.identity.aio import ManagedIdentityCredential, DefaultAzureCredential

async def get_db_connection() -> asyncpg.Connection:
    """
    Connect to PostgreSQL Flexible Server using Managed Identity.
    The Managed Identity must be created as a PostgreSQL user:

    -- Run once as Entra admin:
    SELECT * FROM pgaadauth_create_principal('my-app-identity', false, false);
    GRANT CONNECT ON DATABASE appdb TO "my-app-identity";
    GRANT USAGE ON SCHEMA public TO "my-app-identity";
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO "my-app-identity";
    """
    credential = DefaultAzureCredential()

    # Get an access token for PostgreSQL
    # Scope: https://ossrdbms-aad.database.windows.net/.default
    token = await credential.get_token(
        "https://ossrdbms-aad.database.windows.net/.default"
    )

    # PostgreSQL Flexible Server expects the token as the password
    conn = await asyncpg.connect(
        host="db-prod-eastus.postgres.database.azure.com",
        port=6432,  # Use PgBouncer port
        database="appdb",
        user="my-app-identity",  # The Entra identity name
        password=token.token,
        ssl="require",
    )
    return conn


# For connection pools, refresh the token before it expires
class AzurePostgresPool:
    """Connection pool that refreshes Managed Identity tokens."""

    def __init__(self, host: str, database: str, user: str, min_size: int = 5, max_size: int = 20):
        self.host = host
        self.database = database
        self.user = user
        self.min_size = min_size
        self.max_size = max_size
        self.credential = DefaultAzureCredential()
        self._pool: asyncpg.Pool | None = None

    async def _get_password(self) -> str:
        token = await self.credential.get_token(
            "https://ossrdbms-aad.database.windows.net/.default"
        )
        return token.token

    async def get_pool(self) -> asyncpg.Pool:
        if self._pool is None:
            self._pool = await asyncpg.create_pool(
                host=self.host,
                port=6432,
                database=self.database,
                user=self.user,
                password=await self._get_password(),
                ssl="require",
                min_size=self.min_size,
                max_size=self.max_size,
                # Token expires every 60 min — reset connections to refresh
                max_inactive_connection_lifetime=3000,
            )
        return self._pool

    async def close(self) -> None:
        if self._pool:
            await self._pool.close()
```

**pgvector setup:**

```sql
-- Enable pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Create a table with an embedding column
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    embedding vector(1536),  -- text-embedding-3-small dimensions
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW index for fast approximate nearest neighbor search
-- Better for high-throughput queries (vs IVFFlat which requires training)
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Query: find 10 most similar documents
SELECT id, content, embedding <=> $1::vector AS distance
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

### Cosmos DB for NoSQL

**Globally distributed, serverless-capable document database.** Best when you need multi-region writes, variable schemas, or unpredictable traffic patterns.

**Partition key design — the most critical decision:**

```python
# BAD: Low-cardinality partition key causes hot partitions
{
    "partitionKey": "/status",   # Only 3 values: "active", "pending", "done"
}

# BAD: Monotonically increasing key causes hot partition at write head
{
    "partitionKey": "/createdAt"  # All new writes go to same partition
}

# GOOD: High-cardinality, evenly distributed
{
    "partitionKey": "/userId"     # Each user's data isolated
}

# GOOD: Synthetic partition key for multi-tenant SaaS
# Combine tenant + category for even distribution
item = {
    "id": "doc-123",
    "partitionKey": f"{tenant_id}:{document_type}",  # "acme:invoice"
    "tenantId": tenant_id,
    "type": document_type,
    ...
}
```

**Python SDK with Managed Identity:**

```python
from azure.cosmos.aio import CosmosClient
from azure.identity.aio import DefaultAzureCredential
from azure.cosmos import PartitionKey

COSMOS_ENDPOINT = "https://cosmos-prod.documents.azure.com:443/"

async def get_cosmos_client() -> CosmosClient:
    credential = DefaultAzureCredential()
    return CosmosClient(COSMOS_ENDPOINT, credential=credential)


async def create_document(container_name: str, item: dict) -> dict:
    async with await get_cosmos_client() as client:
        db = client.get_database_client("mydb")
        container = db.get_container_client(container_name)
        return await container.create_item(body=item)


async def query_documents(container_name: str, user_id: str) -> list[dict]:
    async with await get_cosmos_client() as client:
        db = client.get_database_client("mydb")
        container = db.get_container_client(container_name)

        # Always provide partition key when possible — much cheaper
        query = "SELECT * FROM c WHERE c.userId = @userId ORDER BY c._ts DESC"
        parameters = [{"name": "@userId", "value": user_id}]

        items = []
        async for item in container.query_items(
            query=query,
            parameters=parameters,
            partition_key=user_id,  # Avoids cross-partition query
        ):
            items.append(item)
        return items
```

**Serverless vs Provisioned:**

| | Serverless | Provisioned |
|---|---|---|
| **Billing** | Per RU consumed | Per RU/s reserved (hourly) |
| **Best for** | Dev/test, bursty low-average workloads | Steady, predictable high throughput |
| **Max throughput** | 5,000 RU/s per container | Unlimited (with autoscale) |
| **Geo-replication** | Not supported | Supported |
| **Cold start** | Yes | No |

**Change Feed for event-driven patterns:**

```python
from azure.cosmos.aio import CosmosClient
from azure.identity.aio import DefaultAzureCredential

async def process_change_feed(container_name: str):
    """
    Process new/updated documents from Cosmos DB Change Feed.
    Use this to trigger downstream processing without polling.
    """
    credential = DefaultAzureCredential()
    client = CosmosClient("https://cosmos-prod.documents.azure.com:443/", credential)

    db = client.get_database_client("mydb")
    container = db.get_container_client(container_name)

    # Read from beginning of change feed
    feed_iterator = container.query_items_change_feed(is_start_from_beginning=True)

    async for change in feed_iterator:
        await handle_change(change)
        # In production: checkpoint the continuation token to Blob Storage
        # to resume from where you left off after restarts
```

### Azure Cache for Redis / Valkey (April 2026)

**Standard tiers** use Redis OSS. **New option as of 2025**: Azure Cache with **Valkey** engine (the Redis OSS fork after the BSL license change) — same API, open-source license.

```bash
# Create Redis cache (standard tier, 1GB)
az redis create \
  --name redis-prod-eastus \
  --resource-group rg-cache-prod \
  --location eastus \
  --sku Standard \
  --vm-size c1 \
  --redis-version 7

# Get connection string
az redis show \
  --name redis-prod-eastus \
  --resource-group rg-cache-prod \
  --query "[hostName,sslPort,accessKeys.primaryKey]" \
  --output tsv
```

**Python with redis-py and Managed Identity** (Enterprise tier supports Entra auth):

```python
import redis.asyncio as aioredis
from azure.identity.aio import DefaultAzureCredential

# Standard tier: password-based (get from Key Vault)
async def get_redis_client(host: str, password: str) -> aioredis.Redis:
    return await aioredis.from_url(
        f"rediss://{host}:6380",  # SSL on port 6380
        password=password,
        decode_responses=True,
        socket_connect_timeout=5,
        socket_timeout=5,
        retry_on_timeout=True,
    )

# Enterprise tier: Entra ID token-based auth
async def get_redis_enterprise_client(host: str) -> aioredis.Redis:
    credential = DefaultAzureCredential()
    token = await credential.get_token("https://redis.azure.com/.default")

    return await aioredis.from_url(
        f"rediss://{host}:6380",
        username="my-app-identity",
        password=token.token,
        decode_responses=True,
    )
```

---

## Part 5: Azure OpenAI Service (April 2026)

### Mental Model: Deployments vs Models

Azure OpenAI is not the same as `api.openai.com`. The architecture:

```
Azure OpenAI Resource (in your subscription)
├── Deployment: "gpt-4o"         → routes to GPT-4o model
├── Deployment: "gpt-4o-mini"    → routes to GPT-4o-mini model
├── Deployment: "o3"             → routes to o3 reasoning model
├── Deployment: "o4-mini"        → routes to o4-mini reasoning model
└── Deployment: "text-embedding" → routes to text-embedding-3-small/large
```

You call your Azure OpenAI resource URL, not `api.openai.com`. The **deployment name** is what you put in `model=` parameter (not the model name itself).

Available models (April 2026):
- **GPT-4o** (`gpt-4o` with versions like `2024-11-20`) — multimodal, fast, general purpose
- **GPT-4o-mini** — cheaper, faster, good for high-volume tasks
- **o3** — reasoning model, slower, expensive, for complex multi-step problems
- **o4-mini** — efficient reasoning model, good price/performance for reasoning tasks
- **text-embedding-3-large** (3072 dims, supports Matryoshka Representation Learning truncation)
- **text-embedding-3-small** (1536 dims, good price/performance)

### Python SDK: AzureOpenAI Client

**Always use `AzureOpenAI`, never `OpenAI` for Azure OpenAI Service:**

```python
import os
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

# Method 1: Managed Identity (recommended for production)
credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential,
    "https://cognitiveservices.azure.com/.default"
)

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],  # https://aoai-prod.openai.azure.com/
    azure_ad_token_provider=token_provider,
    api_version="2025-01-01-preview",  # Use latest stable or preview
)

# Method 2: API key (for local dev only — never in production)
# client = AzureOpenAI(
#     azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
#     api_key=os.environ["AZURE_OPENAI_API_KEY"],
#     api_version="2025-01-01-preview",
# )
```

### Chat Completions with Streaming

```python
from openai import AzureOpenAI
from openai.types.chat import ChatCompletionChunk
from collections.abc import AsyncGenerator
import asyncio

async def stream_completion(
    client: AzureOpenAI,
    messages: list[dict],
    deployment_name: str = "gpt-4o",
    max_tokens: int = 4096,
) -> AsyncGenerator[str, None]:
    """Stream chat completion tokens."""

    # Note: AzureOpenAI has both sync and async variants
    # Use the async client for FastAPI
    from openai import AsyncAzureOpenAI
    from azure.identity.aio import DefaultAzureCredential
    from azure.identity import get_bearer_token_provider

    async_credential = DefaultAzureCredential()
    async_token_provider = get_bearer_token_provider(
        async_credential,
        "https://cognitiveservices.azure.com/.default"
    )

    async_client = AsyncAzureOpenAI(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        azure_ad_token_provider=async_token_provider,
        api_version="2025-01-01-preview",
    )

    async with async_client.chat.completions.stream(
        model=deployment_name,  # Your deployment name, not the model name
        messages=messages,
        max_tokens=max_tokens,
        temperature=0.7,
    ) as stream:
        async for chunk in stream:
            if chunk.choices and chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content


# FastAPI streaming endpoint
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest) -> StreamingResponse:
    async def generate():
        async for token in stream_completion(
            client=async_client,
            messages=request.messages,
        ):
            yield f"data: {token}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={"X-Accel-Buffering": "no"},  # Disable nginx buffering
    )
```

### Tool Use / Function Calling

```python
from openai import AsyncAzureOpenAI
import json

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City name, e.g. 'Seattle, WA'"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "default": "celsius"
                    }
                },
                "required": ["location"]
            }
        }
    }
]

async def run_with_tools(client: AsyncAzureOpenAI, user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools,
        tool_choice="auto",
        max_tokens=1024,
    )

    message = response.choices[0].message

    # Handle tool calls
    if message.tool_calls:
        messages.append(message)  # Append assistant message with tool calls

        for tool_call in message.tool_calls:
            function_name = tool_call.function.name
            arguments = json.loads(tool_call.function.arguments)

            # Dispatch to actual function
            if function_name == "get_weather":
                result = await fetch_weather(**arguments)
            else:
                result = {"error": f"Unknown function: {function_name}"}

            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result),
            })

        # Get final response with tool results
        final_response = await client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            max_tokens=1024,
        )
        return final_response.choices[0].message.content

    return message.content
```

### Reasoning Models: o3 and o4-mini

```python
# o3 and o4-mini have different parameters than GPT-4o
# They use reasoning_effort instead of temperature
# They do NOT support streaming in the same way (stream the reasoning summary)

response = await client.chat.completions.create(
    model="o3",  # or "o4-mini"
    messages=[
        {"role": "user", "content": "Solve this complex math problem: ..."}
    ],
    # Reasoning-specific parameters
    reasoning_effort="high",    # "low", "medium", "high"
    max_completion_tokens=8192, # Use max_completion_tokens, not max_tokens
    # temperature is not supported for o-series models
)

# Access reasoning summary (if enabled)
if response.choices[0].message.reasoning_content:
    reasoning = response.choices[0].message.reasoning_content
    print(f"Reasoning: {reasoning[:200]}...")

answer = response.choices[0].message.content
```

### Embeddings

```python
from openai import AsyncAzureOpenAI

async def embed_texts(
    client: AsyncAzureOpenAI,
    texts: list[str],
    deployment_name: str = "text-embedding-3-small",
    dimensions: int | None = None,  # MRL: truncate to fewer dimensions
) -> list[list[float]]:
    """
    Embed a batch of texts.

    text-embedding-3-large: 3072 dims (supports MRL truncation)
    text-embedding-3-small: 1536 dims (supports MRL truncation)

    MRL (Matryoshka Representation Learning) allows you to truncate
    embeddings to fewer dimensions with minimal quality loss:
    - 1536 → 256 dims: ~10% quality loss, 6x cheaper to store
    """
    response = await client.embeddings.create(
        model=deployment_name,
        input=texts,
        dimensions=dimensions,  # None = full dimensions
    )

    return [item.embedding for item in response.data]


# Cache embeddings in Redis (deterministic — same input = same output)
import hashlib
import json

async def embed_with_cache(
    client: AsyncAzureOpenAI,
    redis_client,
    text: str,
    deployment_name: str = "text-embedding-3-small",
) -> list[float]:
    cache_key = f"emb:v1:{deployment_name}:{hashlib.sha256(text.encode()).hexdigest()}"

    cached = await redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    embeddings = await embed_texts(client, [text], deployment_name)
    embedding = embeddings[0]

    # Cache for 7 days (embeddings don't change for the same model version)
    await redis_client.setex(cache_key, 86400 * 7, json.dumps(embedding))
    return embedding
```

### Prompt Caching (April 2026)

Available for GPT-4o on Azure OpenAI. Automatically caches prompt prefixes ≥1024 tokens. Check cache usage:

```python
response = await client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    max_tokens=1024,
)

# Check cache hit/miss
usage = response.usage
if hasattr(usage, "prompt_tokens_details"):
    cached_tokens = usage.prompt_tokens_details.cached_tokens
    print(f"Cache hit tokens: {cached_tokens}")
    print(f"Cache miss tokens: {usage.prompt_tokens - cached_tokens}")
```

### Content Filtering

Azure OpenAI has built-in content safety filters. Configure them via Azure AI Foundry:

```python
from openai import BadRequestError

try:
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        max_tokens=1024,
    )
except BadRequestError as e:
    if e.code == "content_filter":
        # Content was filtered
        error_body = e.body
        filter_results = error_body.get("innererror", {}).get("content_filter_results", {})

        for category, result in filter_results.items():
            if result.get("filtered"):
                severity = result.get("severity", "unknown")
                print(f"Content filtered: {category} (severity: {severity})")

        raise ValueError("Content filtered by safety system") from e
    raise
```

### Azure AI Foundry (April 2026)

The unified portal for all Azure AI services. Access at `ai.azure.com`.

- Create and manage Azure OpenAI deployments
- Prompt flow for chaining LLM calls
- Evaluation with built-in metrics (coherence, groundedness, relevance)
- Model catalog (includes open-source models via Azure ML)
- AI Foundry SDK (replaces older Azure ML Python SDK for AI workflows):

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

# Connect to your AI Foundry project
project_client = AIProjectClient(
    credential=DefaultAzureCredential(),
    endpoint="https://myproject.eastus.api.azureml.ms",
    subscription_id=os.environ["AZURE_SUBSCRIPTION_ID"],
    resource_group_name="rg-ai-prod",
    project_name="my-ai-project",
)

# Get Azure OpenAI connection from the project
openai_client = project_client.inference.get_azure_openai_client()
```

### PTU (Provisioned Throughput Units)

For guaranteed throughput SLAs in production. Priced per PTU-hour regardless of usage.

```python
# PTU deployments have the same API — just use a PTU deployment name
# Check for 429 responses and handle them differently:
# - Standard deployments: retry after backoff
# - PTU deployments: overflow to standard deployment (no retry needed)

from openai import RateLimitError

async def chat_with_ptu_fallback(
    ptu_client: AsyncAzureOpenAI,
    standard_client: AsyncAzureOpenAI,
    messages: list[dict],
) -> str:
    try:
        response = await ptu_client.chat.completions.create(
            model="gpt-4o-ptu",  # Your PTU deployment name
            messages=messages,
            max_tokens=2048,
        )
        return response.choices[0].message.content
    except RateLimitError:
        # PTU at capacity → overflow to pay-as-you-go standard
        response = await standard_client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            max_tokens=2048,
        )
        return response.choices[0].message.content
```

---

## Part 6: Messaging — Service Bus, Event Grid, Event Hubs

### Azure Service Bus — Enterprise Messaging

**Analogy:** Service Bus is like a **post office with guaranteed delivery**. Messages are durable (stored until consumed), ordered within a session, and have built-in retry/dead-letter handling.

**Queue vs Topic:**
- **Queue**: One sender → one receiver (point-to-point)
- **Topic + Subscriptions**: One sender → many receivers (fan-out/pub-sub)

```bash
# Create Service Bus namespace
az servicebus namespace create \
  --name sb-prod-eastus \
  --resource-group rg-messaging-prod \
  --location eastus \
  --sku Premium \
  --capacity 1

# Create a queue
az servicebus queue create \
  --name task-queue \
  --namespace-name sb-prod-eastus \
  --resource-group rg-messaging-prod \
  --max-size 1024 \
  --lock-duration PT1M \
  --max-delivery-count 5 \
  --dead-lettering-on-message-expiration true

# Create a topic
az servicebus topic create \
  --name events \
  --namespace-name sb-prod-eastus \
  --resource-group rg-messaging-prod

# Create subscriptions on the topic
az servicebus topic subscription create \
  --name notification-service \
  --topic-name events \
  --namespace-name sb-prod-eastus \
  --resource-group rg-messaging-prod \
  --max-delivery-count 5

az servicebus topic subscription create \
  --name analytics-service \
  --topic-name events \
  --namespace-name sb-prod-eastus \
  --resource-group rg-messaging-prod
```

**Python SDK — Publisher:**

```python
from azure.servicebus.aio import ServiceBusClient
from azure.servicebus import ServiceBusMessage
from azure.identity.aio import DefaultAzureCredential
import json

SERVICEBUS_NAMESPACE = "sb-prod-eastus.servicebus.windows.net"

async def publish_message(queue_or_topic: str, payload: dict, session_id: str | None = None) -> None:
    """Publish a message to a Service Bus queue or topic."""
    credential = DefaultAzureCredential()

    async with ServiceBusClient(SERVICEBUS_NAMESPACE, credential) as client:
        async with client.get_queue_sender(queue_or_topic) as sender:
            message = ServiceBusMessage(
                body=json.dumps(payload),
                content_type="application/json",
                subject="TaskCreated",  # Like a message type header
                session_id=session_id,  # For ordered processing
                correlation_id=payload.get("request_id"),
                time_to_live=timedelta(hours=24),
            )
            await sender.send_messages(message)


async def publish_batch(queue_name: str, payloads: list[dict]) -> None:
    """Publish multiple messages efficiently in a single batch."""
    credential = DefaultAzureCredential()

    async with ServiceBusClient(SERVICEBUS_NAMESPACE, credential) as client:
        async with client.get_queue_sender(queue_name) as sender:
            batch = await sender.create_message_batch()

            for payload in payloads:
                try:
                    batch.add_message(ServiceBusMessage(
                        body=json.dumps(payload),
                        content_type="application/json",
                    ))
                except ValueError:
                    # Batch is full — send current batch and start new
                    await sender.send_messages(batch)
                    batch = await sender.create_message_batch()
                    batch.add_message(ServiceBusMessage(body=json.dumps(payload)))

            if batch:
                await sender.send_messages(batch)
```

**Python SDK — Consumer (with DLQ handling):**

```python
from azure.servicebus.aio import ServiceBusClient
from azure.servicebus import ServiceBusReceivedMessage, ServiceBusSubQueue
import logging

async def process_queue(queue_name: str, handler) -> None:
    """
    Long-running queue processor.
    - Receives messages with auto lock renewal
    - Completes on success, Dead Letters on unrecoverable error
    - Abandons on transient error (Service Bus will retry)
    """
    credential = DefaultAzureCredential()

    async with ServiceBusClient(SERVICEBUS_NAMESPACE, credential) as client:
        async with client.get_queue_receiver(
            queue_name,
            max_wait_time=5,  # Seconds to wait for new messages
        ) as receiver:
            async for message in receiver:
                try:
                    payload = json.loads(str(message))
                    await handler(payload)
                    await receiver.complete_message(message)

                except ValueError as e:
                    # Unrecoverable (bad message format) → send to DLQ
                    logging.error(f"Unrecoverable error processing message: {e}")
                    await receiver.dead_letter_message(
                        message,
                        reason="InvalidMessageFormat",
                        error_description=str(e),
                    )

                except Exception as e:
                    # Transient error → abandon (Service Bus will retry)
                    logging.warning(f"Transient error, abandoning message: {e}")
                    await receiver.abandon_message(message)


async def process_dlq(queue_name: str) -> None:
    """Process Dead Letter Queue messages for investigation."""
    credential = DefaultAzureCredential()

    async with ServiceBusClient(SERVICEBUS_NAMESPACE, credential) as client:
        async with client.get_queue_receiver(
            queue_name,
            sub_queue=ServiceBusSubQueue.DEAD_LETTER,
        ) as dlq_receiver:
            async for message in dlq_receiver:
                reason = message.dead_letter_reason
                description = message.dead_letter_error_description
                body = str(message)

                logging.error(
                    f"DLQ message: reason={reason}, description={description}, "
                    f"delivery_count={message.delivery_count}, body={body}"
                )
                # After investigation, complete to remove from DLQ
                await dlq_receiver.complete_message(message)
```

**Sessions for ordered message processing:**

```python
# Sessions guarantee FIFO ordering per session_id
# Use for: per-user ordered events, per-order state machines

async def process_session_queue(queue_name: str) -> None:
    """Process a session-enabled queue. Each session is processed sequentially."""
    credential = DefaultAzureCredential()

    async with ServiceBusClient(SERVICEBUS_NAMESPACE, credential) as client:
        # Accept next available session
        async with client.get_queue_receiver(
            queue_name,
            session_id=NEXT_AVAILABLE_SESSION,  # Process any available session
        ) as session_receiver:
            session_id = session_receiver.session_id
            logging.info(f"Processing session: {session_id}")

            async for message in session_receiver:
                payload = json.loads(str(message))
                # All messages with same session_id arrive in order
                await process_ordered_event(session_id, payload)
                await session_receiver.complete_message(message)
```

### Azure Event Grid — Event-Driven Routing

**Analogy:** Event Grid is like a **switchboard operator** for events. An event happens (Blob uploaded, resource created) → Event Grid routes it to the right subscriber.

```python
from azure.eventgrid import EventGridPublisherClient, EventGridEvent
from azure.eventgrid.aio import EventGridPublisherClient as AsyncEventGridPublisherClient
from azure.identity.aio import DefaultAzureCredential
from datetime import datetime, timezone
import uuid

async def publish_custom_event(topic_endpoint: str, event_type: str, data: dict) -> None:
    """Publish a custom event to Event Grid using CloudEvents format."""
    credential = DefaultAzureCredential()

    async with AsyncEventGridPublisherClient(topic_endpoint, credential) as client:
        event = {
            "specversion": "1.0",
            "type": event_type,  # e.g. "com.myapp.order.created"
            "source": "/myapp/orders",
            "id": str(uuid.uuid4()),
            "time": datetime.now(timezone.utc).isoformat(),
            "datacontenttype": "application/json",
            "data": data,
        }

        await client.send([event])
```

### Azure Event Hubs — Streaming Ingestion

**Analogy:** Event Hubs is like a **Kafka cluster you don't manage**. For high-throughput event streaming (millions of events/second), telemetry ingestion, log aggregation.

```python
from azure.eventhub.aio import EventHubProducerClient, EventHubConsumerClient
from azure.eventhub import EventData
from azure.identity.aio import DefaultAzureCredential
import json

EVENTHUB_NAMESPACE = "eh-prod-eastus.servicebus.windows.net"
EVENTHUB_NAME = "telemetry"

async def send_telemetry(events: list[dict]) -> None:
    """Send a batch of telemetry events to Event Hubs."""
    credential = DefaultAzureCredential()

    async with EventHubProducerClient(
        EVENTHUB_NAMESPACE,
        EVENTHUB_NAME,
        credential,
    ) as producer:
        batch = await producer.create_batch()

        for event in events:
            batch.add(EventData(json.dumps(event)))

        await producer.send_batch(batch)


async def consume_telemetry(consumer_group: str = "$Default") -> None:
    """Consume events from Event Hubs using the consumer client."""
    credential = DefaultAzureCredential()

    async with EventHubConsumerClient(
        EVENTHUB_NAMESPACE,
        EVENTHUB_NAME,
        consumer_group,
        credential,
    ) as consumer:
        async def on_event(partition_context, event):
            data = json.loads(event.body_as_str())
            await process_telemetry(data)
            # Checkpoint to Blob Storage (tracks position per partition)
            await partition_context.update_checkpoint(event)

        async def on_error(partition_context, error):
            logging.error(f"Event Hubs error on partition {partition_context.partition_id}: {error}")

        await consumer.receive(
            on_event=on_event,
            on_error=on_error,
            starting_position="-1",  # -1 = from beginning, "@latest" = new events only
        )
```

**When Event Hubs vs Service Bus:**

| | Event Hubs | Service Bus |
|---|---|---|
| **Pattern** | Streaming (like Kafka) | Messaging (like RabbitMQ) |
| **Ordering** | Per-partition | Per-session (optional) |
| **Consumer model** | Pull (consumer reads) | Push (Service Bus delivers) |
| **Replay** | Yes (retention period) | No (once consumed, gone) |
| **Throughput** | Millions/sec | Thousands/sec |
| **Use case** | Telemetry, logs, clickstream | Workflows, commands, task queues |
| **DLQ** | No | Yes |

---

## Part 7: Storage — Blob Storage, ADLS, Files

### Blob Storage Tiers and Lifecycle

```bash
# Create storage account (locally redundant for dev, ZRS for prod)
az storage account create \
  --name stprodeastus01 \
  --resource-group rg-storage-prod \
  --location eastus \
  --sku Standard_ZRS \
  --kind StorageV2 \
  --access-tier Hot \
  --allow-blob-public-access false \
  --min-tls-version TLS1_2

# Create container
az storage container create \
  --name documents \
  --account-name stprodeastus01 \
  --auth-mode login  # Use Entra ID auth
```

**Lifecycle management (move to cheaper tiers automatically):**

```json
{
  "rules": [
    {
      "name": "archive-old-documents",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["documents/archived/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

### User Delegation SAS (Preferred, April 2026)

**Never use account key SAS in production.** User Delegation SAS is signed by Entra ID and can be revoked:

```python
from azure.storage.blob.aio import BlobServiceClient
from azure.identity.aio import DefaultAzureCredential
from datetime import datetime, timedelta, timezone

async def generate_upload_sas(
    storage_account: str,
    container_name: str,
    blob_name: str,
    expiry_hours: int = 1,
) -> str:
    """
    Generate a User Delegation SAS for a single blob.
    The caller (Container App) needs Storage Blob Delegator role.
    """
    from azure.storage.blob import (
        BlobSasPermissions,
        generate_blob_sas,
        UserDelegationKey,
    )

    credential = DefaultAzureCredential()
    account_url = f"https://{storage_account}.blob.core.windows.net"

    async with BlobServiceClient(account_url, credential) as service_client:
        # Get a user delegation key (valid up to 7 days)
        delegation_key: UserDelegationKey = await service_client.get_user_delegation_key(
            key_start_time=datetime.now(timezone.utc),
            key_expiry_time=datetime.now(timezone.utc) + timedelta(hours=expiry_hours + 1),
        )

    sas_token = generate_blob_sas(
        account_name=storage_account,
        container_name=container_name,
        blob_name=blob_name,
        user_delegation_key=delegation_key,
        permission=BlobSasPermissions(read=True, write=True, create=True),
        expiry=datetime.now(timezone.utc) + timedelta(hours=expiry_hours),
    )

    return f"https://{storage_account}.blob.core.windows.net/{container_name}/{blob_name}?{sas_token}"


async def upload_blob_with_managed_identity(
    storage_account: str,
    container_name: str,
    blob_name: str,
    data: bytes,
) -> str:
    """Upload a blob directly using Managed Identity (no SAS needed for service-to-service)."""
    credential = DefaultAzureCredential()
    account_url = f"https://{storage_account}.blob.core.windows.net"

    async with BlobServiceClient(account_url, credential) as service_client:
        container_client = service_client.get_container_client(container_name)
        blob_client = container_client.get_blob_client(blob_name)

        await blob_client.upload_blob(data, overwrite=True)
        return blob_client.url
```

### Static Website Hosting

```bash
az storage blob service-properties update \
  --account-name ststaticweb \
  --static-website \
  --index-document index.html \
  --404-document 404.html
```

---

## Part 8: Security — Managed Identity, Key Vault, Entra ID

### Managed Identity — The Golden Pattern

**Mental model:** Managed Identity is like a **badge that Azure issues to your compute**. The Container App *is* the identity — no password, no certificate, no rotation needed. Azure manages the credential lifecycle entirely.

```python
from azure.identity import DefaultAzureCredential
from azure.identity.aio import DefaultAzureCredential as AsyncDefaultAzureCredential

# DefaultAzureCredential tries these in order:
# 1. EnvironmentCredential (AZURE_CLIENT_ID + AZURE_CLIENT_SECRET or cert)
# 2. WorkloadIdentityCredential (AKS with Workload Identity)
# 3. ManagedIdentityCredential (Container Apps, VMs, Functions)
# 4. AzureCliCredential (local development: az login)
# 5. AzurePowerShellCredential
# 6. AzureDeveloperCliCredential (azd login)

# This single line works in production (Managed Identity) and locally (az login)
credential = DefaultAzureCredential()

# For user-assigned managed identity (when you have multiple identities on a resource)
from azure.identity import ManagedIdentityCredential

credential = ManagedIdentityCredential(
    client_id="12345678-1234-1234-1234-123456789012"  # User-assigned MI client ID
)
```

### Key Vault — Secrets, Keys, Certificates

```python
from azure.keyvault.secrets.aio import SecretClient
from azure.identity.aio import DefaultAzureCredential

async def get_secret(vault_name: str, secret_name: str) -> str:
    """Retrieve a secret from Key Vault using Managed Identity."""
    credential = DefaultAzureCredential()
    vault_url = f"https://{vault_name}.vault.azure.net"

    async with SecretClient(vault_url, credential) as client:
        secret = await client.get_secret(secret_name)
        return secret.value


# In FastAPI: use Key Vault references in Container Apps instead
# Key Vault → Container Apps → Environment Variable → os.environ["SECRET"]
# No Key Vault SDK needed in the application code at all

# For Bicep Key Vault reference (the preferred pattern):
# The Container App reads the secret at startup via Azure infrastructure
```

**Key Vault RBAC assignments (April 2026 standard):**

```bash
# Grant a Container App's identity access to specific secrets
PRINCIPAL_ID=$(az containerapp show \
  --name my-api \
  --resource-group rg-prod \
  --query "identity.principalId" -o tsv)

KV_ID=$(az keyvault show \
  --name kv-prod \
  --resource-group rg-prod \
  --query id -o tsv)

# Key Vault Secrets User: can read secrets (not list all secrets)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope "$KV_ID/secrets/database-url"  # Scope to specific secret

# Key Vault Secrets Officer: can create/update secrets (for deploy pipelines)
az role assignment create \
  --assignee $PIPELINE_PRINCIPAL_ID \
  --role "Key Vault Secrets Officer" \
  --scope $KV_ID
```

**Soft delete and purge protection (mandatory for compliance):**

```bash
az keyvault update \
  --name kv-prod \
  --resource-group rg-prod \
  --enable-soft-delete true \
  --retention-days 90 \
  --enable-purge-protection true
```

### Workload Identity Federation for GitHub Actions

Never store Azure credentials as GitHub secrets. Use Workload Identity Federation — GitHub's OIDC token authenticates to Entra ID:

```bash
# Step 1: Create an app registration
APP_ID=$(az ad app create \
  --display-name "github-actions-myrepo" \
  --query appId -o tsv)

SP_ID=$(az ad sp create \
  --id $APP_ID \
  --query id -o tsv)

# Step 2: Create federated credential (links GitHub repo to the app)
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-main-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:your-org/your-repo:ref:refs/heads/main",
    "description": "GitHub Actions main branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Also add for pull requests
az ad app federated-credential create \
  --id $APP_ID \
  --parameters '{
    "name": "github-pull-request",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:your-org/your-repo:pull_request",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Step 3: Grant the service principal permissions
az role assignment create \
  --assignee $SP_ID \
  --role "Contributor" \
  --scope /subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-prod
```

**GitHub Actions workflow:**

```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC — no secrets needed)
        uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}       # App registration client ID (not secret)
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Bicep
        run: |
          az deployment group create \
            --resource-group rg-prod \
            --template-file infra/main.bicep \
            --parameters @infra/main.prod.bicepparam

      - name: Deploy Container App
        run: |
          az containerapp update \
            --name my-api \
            --resource-group rg-prod \
            --image myregistry.azurecr.io/my-api:${{ github.sha }}
```

---

## Part 9: Azure MCP Servers (April 2026)

### What is azure-mcp-server?

Microsoft's official MCP server for Azure. It exposes Azure operations as tools that Claude (or any MCP client) can call — without writing az CLI scripts or opening the portal.

Tools available:
- `azure_resource_list` — list resources in a subscription/resource group
- `azure_blob_read` — read blob content
- `azure_blob_list` — list blobs in a container
- `azure_cosmos_query` — query Cosmos DB containers
- `azure_keyvault_secret_get` — retrieve secrets
- `azure_servicebus_peek` — peek at Service Bus messages
- `azure_deployment_create` — create ARM/Bicep deployments
- `azure_log_query` — query Log Analytics with KQL

### Installation and Configuration

```bash
# Install via npm (or use the Docker image)
npm install -g @azure/azure-mcp-server

# Or with Docker
docker pull mcr.microsoft.com/azure-mcp-server:latest
```

**Claude Code (`~/.claude.json`) configuration:**

```json
{
  "mcpServers": {
    "azure": {
      "command": "azure-mcp-server",
      "env": {
        "AZURE_SUBSCRIPTION_ID": "12345678-...",
        "AZURE_TENANT_ID": "your-tenant-id",
        "AZURE_CLIENT_ID": "your-managed-identity-client-id"
        // No AZURE_CLIENT_SECRET — uses DefaultAzureCredential
        // Works with `az login` for local dev
      }
    }
  }
}
```

The server uses `DefaultAzureCredential` — so it works with `az login` locally and Managed Identity in CI/CD.

### Claude Code + Azure MCP Workflow

```
# Example: Claude Code can now answer questions like:
"What Container Apps are running in rg-prod?"
→ Calls azure_resource_list(resource_group="rg-prod", type="Microsoft.App/containerApps")

"Show me the latest logs for my-api"
→ Calls azure_log_query(workspace="law-prod", query="ContainerAppConsoleLogs_CL ...")

"What's in the database-url secret?"
→ Calls azure_keyvault_secret_get(vault="kv-prod", name="database-url")

"Deploy the staging bicep template"
→ Calls azure_deployment_create(resource_group="rg-staging", template="./infra/main.bicep")
```

### Use Cases in Development Workflow

1. **Debugging in production**: Ask Claude to fetch logs, check app state, peek at Service Bus queues — all without switching to the portal
2. **Drift detection**: Compare Bicep template to actual deployed resource state
3. **Incident response**: Claude can query Log Analytics, check metrics, identify the failing component
4. **Deployment review**: Before running `az deployment group create`, have Claude review the what-if diff

---

## Part 10: Infrastructure as Code — Bicep and Terraform

### Bicep — Azure Native IaC

**Bicep compiles to ARM JSON** — think of it as ARM templates with a sane syntax. It's the recommended choice for Azure-only stacks.

**Simple Container App + PostgreSQL Bicep template:**

```bicep
// infra/main.bicep

@description('Environment name (dev, staging, prod)')
param environmentName string

@description('Location for all resources')
param location string = resourceGroup().location

@description('Container image to deploy')
param containerImage string

var prefix = 'myapp-${environmentName}'

// Log Analytics Workspace
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: '${prefix}-law'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: '${prefix}-kv'
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: tenant().tenantId
    enableRbacAuthorization: true  // RBAC, not legacy access policies
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
  }
}

// Container Apps Environment
resource containerAppsEnv 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: '${prefix}-cae'
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
  }
}

// PostgreSQL Flexible Server
resource postgresql 'Microsoft.DBforPostgreSQL/flexibleServers@2024-08-01' = {
  name: '${prefix}-db'
  location: location
  sku: {
    name: 'Standard_D2ds_v5'
    tier: 'GeneralPurpose'
  }
  properties: {
    administratorLogin: 'dbadmin'
    administratorLoginPassword: keyVault.getSecret('db-admin-password')
    version: '16'
    storage: {
      storageSizeGB: 32
    }
    backup: {
      backupRetentionDays: 7
      geoRedundantBackup: 'Disabled'
    }
    highAvailability: {
      mode: 'Disabled'  // Enable ZoneRedundant for prod
    }
    authConfig: {
      activeDirectoryAuth: 'Enabled'
      passwordAuth: 'Enabled'
    }
  }
}

// Container App
resource containerApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: '${prefix}-api'
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    managedEnvironmentId: containerAppsEnv.id
    configuration: {
      ingress: {
        external: true
        targetPort: 8000
        transport: 'auto'
      }
      secrets: [
        {
          name: 'database-url'
          keyVaultUrl: '${keyVault.properties.vaultUri}secrets/database-url'
          identity: 'system'
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'api'
          image: containerImage
          resources: {
            cpu: json('0.5')
            memory: '1.0Gi'
          }
          env: [
            {
              name: 'DATABASE_URL'
              secretRef: 'database-url'
            }
            {
              name: 'ENVIRONMENT'
              value: environmentName
            }
          ]
        }
      ]
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [
          {
            name: 'http-scaling'
            http: {
              metadata: {
                concurrentRequests: '50'
              }
            }
          }
        ]
      }
    }
  }
}

// Grant Container App identity access to Key Vault secrets
resource kvSecretsUserRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, containerApp.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '4633458b-17de-408a-b874-0445c86b69e6'  // Key Vault Secrets User
    )
    principalId: containerApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Outputs
output containerAppFqdn string = containerApp.properties.configuration.ingress.fqdn
output containerAppPrincipalId string = containerApp.identity.principalId
```

**Parameters file:**

```bicep
// infra/main.prod.bicepparam
using 'main.bicep'

param environmentName = 'prod'
param location = 'eastus'
param containerImage = 'myregistry.azurecr.io/my-api:latest'
```

**Deploy:**

```bash
# What-if first
az deployment group what-if \
  --resource-group rg-myapp-prod \
  --template-file infra/main.bicep \
  --parameters @infra/main.prod.bicepparam

# Apply
az deployment group create \
  --resource-group rg-myapp-prod \
  --template-file infra/main.bicep \
  --parameters @infra/main.prod.bicepparam
```

**Bicep linting:**

```bash
az bicep lint --file infra/main.bicep
# Or
bicep lint infra/main.bicep
```

### Terraform with azurerm v4+

For teams already standardized on Terraform or running multi-cloud:

```hcl
# infra/main.tf

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    azapi = {
      source  = "azure/azapi"
      version = "~> 2.0"  # For resources not yet in azurerm
    }
  }

  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "sttfstateeastus"
    container_name       = "tfstate"
    key                  = "myapp/prod/terraform.tfstate"
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = false
      recover_soft_deleted_key_vaults = true
    }
  }
  # Uses DefaultAzureCredential (az login or Managed Identity in CI)
}

# Container Apps Environment
resource "azurerm_container_app_environment" "main" {
  name                       = "${local.prefix}-cae"
  location                   = azurerm_resource_group.main.location
  resource_group_name        = azurerm_resource_group.main.name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
}

# PostgreSQL Flexible Server
resource "azurerm_postgresql_flexible_server" "main" {
  name                   = "${local.prefix}-db"
  resource_group_name    = azurerm_resource_group.main.name
  location               = azurerm_resource_group.main.location
  version                = "16"
  administrator_login    = "dbadmin"
  administrator_password = random_password.db.result

  storage_mb   = 32768
  sku_name     = "GP_Standard_D2ds_v5"

  high_availability {
    mode = "ZoneRedundant"
  }

  authentication {
    active_directory_auth_enabled = true
    password_auth_enabled         = true
    tenant_id                     = data.azurerm_client_config.current.tenant_id
  }
}

# Container App (use azapi for latest features)
resource "azapi_resource" "container_app" {
  type      = "Microsoft.App/containerApps@2024-03-01"
  name      = "${local.prefix}-api"
  location  = azurerm_resource_group.main.location
  parent_id = azurerm_resource_group.main.id

  identity {
    type = "SystemAssigned"
  }

  body = jsonencode({
    properties = {
      managedEnvironmentId = azurerm_container_app_environment.main.id
      configuration = {
        ingress = {
          external   = true
          targetPort = 8000
        }
      }
      template = {
        containers = [{
          name  = "api"
          image = var.container_image
          resources = {
            cpu    = 0.5
            memory = "1.0Gi"
          }
        }]
        scale = {
          minReplicas = 1
          maxReplicas = 10
        }
      }
    }
  })
}
```

**Remote state in Azure Blob Storage setup:**

```bash
az group create --name rg-terraform-state --location eastus
az storage account create \
  --name sttfstateeastus \
  --resource-group rg-terraform-state \
  --sku Standard_RAGRS \
  --kind StorageV2

az storage container create \
  --name tfstate \
  --account-name sttfstateeastus

# Initialize Terraform
terraform init
terraform plan -var-file=prod.tfvars
terraform apply -var-file=prod.tfvars
```

### Bicep vs Terraform Comparison

| Criterion | Bicep | Terraform (azurerm v4) |
|---|---|---|
| **Learning curve** | Lower (Azure concepts map directly) | Moderate (HCL + azurerm abstraction) |
| **Azure parity** | Immediate (compiles to ARM) | Delayed (azurerm team adds resources) |
| **Preview resources** | Works immediately | Use `azapi` provider |
| **Multi-cloud** | No (Azure only) | Yes |
| **State management** | No state file | State file (backend required) |
| **Drift detection** | Via what-if | `terraform plan` |
| **Modules** | Bicep Registry | Terraform Registry |
| **CI/CD integration** | `az deployment group create` | `terraform apply` |
| **Choose when** | Azure-only, team new to IaC | Multi-cloud, existing Terraform investment |

---

## Part 11: Observability — Azure Monitor, Application Insights, Log Analytics

### The Stack

```
Your App
  └── OpenTelemetry SDK
        ├── Traces → Application Insights (via OTLP exporter)
        ├── Metrics → Azure Monitor Metrics
        └── Logs → Log Analytics Workspace

Azure Monitor (umbrella service)
├── Application Insights (APM — traces, custom metrics, live metrics)
├── Log Analytics Workspace (all logs, KQL queries)
└── Alerts (based on metrics or KQL queries)
```

### Application Insights with OpenTelemetry

```python
# Install: pip install azure-monitor-opentelemetry
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.asyncpg import AsyncPGInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
import logging

def setup_observability(app) -> None:
    """Configure OpenTelemetry → Application Insights."""

    # Connection string from environment (Key Vault → Container App env var)
    connection_string = os.environ["APPLICATIONINSIGHTS_CONNECTION_STRING"]

    configure_azure_monitor(
        connection_string=connection_string,
        # cloud_role_name is what appears in Application Map
        cloud_role_name=os.environ.get("CONTAINER_APP_NAME", "api"),
    )

    # Auto-instrument FastAPI (traces all requests)
    FastAPIInstrumentor.instrument_app(app)

    # Auto-instrument database and cache clients
    AsyncPGInstrumentor().instrument()
    RedisInstrumentor().instrument()


# Custom spans for business logic
tracer = trace.get_tracer("myapp")

async def process_order(order_id: str, user_id: str) -> dict:
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("user.id", user_id)

        result = await do_processing(order_id)
        span.set_attribute("order.total", result["total"])
        return result


# Custom metrics
from opentelemetry import metrics

meter = metrics.get_meter("myapp")
order_counter = meter.create_counter(
    "orders_processed",
    description="Number of orders processed",
    unit="1",
)
llm_cost_histogram = meter.create_histogram(
    "llm_cost_per_request",
    description="LLM API cost per request in USD",
    unit="USD",
)

# Use them
order_counter.add(1, {"status": "success", "tier": "premium"})
llm_cost_histogram.record(0.002, {"model": "gpt-4o-mini", "endpoint": "/chat"})
```

### KQL — Kusto Query Language

KQL is how you query Log Analytics. The syntax is pipeline-based (like PowerShell or Spark):

```kusto
// Recent errors in Container App
ContainerAppConsoleLogs_CL
| where TimeGenerated > ago(1h)
| where ContainerAppName_s == "my-api"
| where Log_s contains "ERROR"
| project TimeGenerated, Log_s, RevisionName_s
| order by TimeGenerated desc
| limit 100

// Average response time by endpoint (last 24h)
requests
| where timestamp > ago(24h)
| summarize avg(duration), percentile(duration, 95), percentile(duration, 99), count()
  by name  // name = endpoint path
| order by percentile_duration_95 desc

// Failed requests rate
requests
| where timestamp > ago(1h)
| summarize
    total = count(),
    failed = countif(success == false)
  by bin(timestamp, 5m)
| extend failure_rate = toreal(failed) / toreal(total) * 100
| render timechart

// LLM token usage from custom metrics
customMetrics
| where name == "llm_cost_per_request"
| summarize
    total_cost = sum(value),
    avg_cost = avg(value),
    p95_cost = percentile(value, 95)
  by bin(timestamp, 1h), tostring(customDimensions.model)
| order by timestamp desc
```

### Alerts and Action Groups

```bash
# Create Action Group (who to notify)
az monitor action-group create \
  --name ag-oncall \
  --resource-group rg-monitoring \
  --short-name oncall \
  --action email oncall-email oncall@company.com

# Create metric alert (P99 latency > 5s)
az monitor metrics alert create \
  --name "api-p99-latency-high" \
  --resource-group rg-monitoring \
  --scopes $(az resource show --name appi-prod --resource-group rg-monitoring --resource-type "microsoft.insights/components" --query id -o tsv) \
  --condition "avg requests/duration > 5000" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action $(az monitor action-group show --name ag-oncall --resource-group rg-monitoring --query id -o tsv)

# Create log alert (error rate > 1%)
az monitor scheduled-query create \
  --name "api-error-rate-high" \
  --resource-group rg-monitoring \
  --scopes $(az resource show --name law-prod --resource-group rg-monitoring --resource-type "microsoft.operationalinsights/workspaces" --query id -o tsv) \
  --condition-query "requests | where timestamp > ago(5m) | summarize failed=countif(success==false), total=count() | where total > 10 | extend rate=toreal(failed)/toreal(total) | where rate > 0.01" \
  --condition-threshold 0 \
  --condition-operator GreaterThan \
  --window-duration PT5M \
  --evaluation-frequency PT1M \
  --severity 1 \
  --action-groups $(az monitor action-group show --name ag-oncall --resource-group rg-monitoring --query id -o tsv)
```

---

## Part 12: Architecture Patterns for FastAPI on Azure

### Pattern 1: Standard API (Most Common)

**Stack:** Container Apps + PostgreSQL Flexible Server + Azure Cache for Redis + Azure OpenAI

```
Internet → Azure Container Apps (my-api)
               ├── Azure Database for PostgreSQL Flexible Server (port 6432, PgBouncer)
               ├── Azure Cache for Redis (port 6380, TLS)
               └── Azure OpenAI Service (GPT-4o deployment)
           Key Vault → secrets injected via KV references
           Log Analytics ← telemetry (OpenTelemetry → Application Insights)
```

```python
# app/dependencies.py — dependency injection for Azure services

from functools import lru_cache
from typing import Annotated
from fastapi import Depends
import asyncpg
import redis.asyncio as aioredis
from openai import AsyncAzureOpenAI
from azure.identity.aio import DefaultAzureCredential
from azure.identity import get_bearer_token_provider

@lru_cache
def get_settings():
    from app.config import Settings
    return Settings()  # pydantic-settings reads env vars

async def get_db_pool(settings = Depends(get_settings)) -> asyncpg.Pool:
    # DATABASE_URL comes from Key Vault reference → env var
    # Format: postgresql://my-app@db.postgres.database.azure.com:6432/appdb?sslmode=require
    # For Managed Identity auth, token is the password — see Part 4
    return await asyncpg.create_pool(settings.DATABASE_URL, min_size=5, max_size=20)

async def get_redis(settings = Depends(get_settings)) -> aioredis.Redis:
    return await aioredis.from_url(
        settings.REDIS_URL,
        decode_responses=True,
        socket_connect_timeout=5,
    )

def get_openai_client(settings = Depends(get_settings)) -> AsyncAzureOpenAI:
    credential = DefaultAzureCredential()
    token_provider = get_bearer_token_provider(
        credential,
        "https://cognitiveservices.azure.com/.default"
    )
    return AsyncAzureOpenAI(
        azure_endpoint=settings.AZURE_OPENAI_ENDPOINT,
        azure_ad_token_provider=token_provider,
        api_version="2025-01-01-preview",
    )

# Type aliases for cleaner route signatures
DBPool = Annotated[asyncpg.Pool, Depends(get_db_pool)]
RedisClient = Annotated[aioredis.Redis, Depends(get_redis)]
OpenAIClient = Annotated[AsyncAzureOpenAI, Depends(get_openai_client)]
```

### Pattern 2: Event-Driven Worker

**Stack:** Container Apps + Cosmos DB NoSQL + Service Bus

```
Container App: api         → publishes to Service Bus topic "events"
Container App: worker      ← subscribes to Service Bus "events/my-subscription"
    (scales to 0, KEDA scales up when queue depth > 0)
    └── writes to Cosmos DB NoSQL
```

```python
# app/worker/main.py — Service Bus consumer that scales with KEDA

import asyncio
import logging
from azure.servicebus.aio import ServiceBusClient
from azure.identity.aio import DefaultAzureCredential
from app.handlers import handle_event
import json

SERVICEBUS_NAMESPACE = os.environ["SERVICEBUS_NAMESPACE"]
TOPIC_NAME = "events"
SUBSCRIPTION_NAME = "my-subscription"

async def main():
    credential = DefaultAzureCredential()

    async with ServiceBusClient(SERVICEBUS_NAMESPACE, credential) as client:
        async with client.get_subscription_receiver(
            TOPIC_NAME,
            SUBSCRIPTION_NAME,
            max_wait_time=30,
        ) as receiver:
            logging.info("Worker started, waiting for messages")

            async for message in receiver:
                try:
                    event = json.loads(str(message))
                    await handle_event(event)
                    await receiver.complete_message(message)
                    logging.info(f"Processed event: {event.get('type')}")
                except Exception as e:
                    logging.error(f"Failed to process message: {e}", exc_info=True)
                    await receiver.abandon_message(message)

if __name__ == "__main__":
    asyncio.run(main())
```

### Pattern 3: AI-Native RAG API

**Stack:** Container Apps + Azure OpenAI + PostgreSQL pgvector + Service Bus (async indexing)

```
User Request → Container App: api
    ├── Embed query → Azure OpenAI (text-embedding-3-small)
    ├── Vector search → PostgreSQL Flexible Server (pgvector HNSW)
    ├── Generate response → Azure OpenAI (GPT-4o, streaming)
    └── Stream SSE back to client

Background indexing:
Document Upload → Container App: api → Service Bus queue: "index-jobs"
    Container App: indexer (scale to 0) ← receives job
        ├── Chunk document
        ├── Embed chunks → Azure OpenAI (text-embedding-3-small)
        └── Store chunks + embeddings → PostgreSQL pgvector
```

```python
# app/rag.py — Full RAG pipeline on Azure

import asyncio
from openai import AsyncAzureOpenAI
import asyncpg
from app.dependencies import get_db_pool, get_openai_client
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

app = FastAPI()

@app.post("/ask")
async def ask(
    question: str,
    db: asyncpg.Pool = Depends(get_db_pool),
    openai: AsyncAzureOpenAI = Depends(get_openai_client),
) -> StreamingResponse:

    # Step 1: Embed the question
    embed_response = await openai.embeddings.create(
        model="text-embedding-3-small",
        input=question,
    )
    query_vector = embed_response.data[0].embedding

    # Step 2: Vector search in PostgreSQL
    async with db.acquire() as conn:
        rows = await conn.fetch(
            """
            SELECT content, metadata,
                   1 - (embedding <=> $1::vector) AS similarity
            FROM documents
            WHERE 1 - (embedding <=> $1::vector) > 0.7
            ORDER BY embedding <=> $1::vector
            LIMIT 5
            """,
            query_vector,
        )

    context = "\n\n".join(
        f"[Source: {row['metadata'].get('source', 'unknown')}]\n{row['content']}"
        for row in rows
    )

    # Step 3: Generate response with streaming
    messages = [
        {
            "role": "system",
            "content": (
                "You are a helpful assistant. Answer based on the provided context. "
                "If the answer is not in the context, say so.\n\n"
                f"Context:\n{context}"
            )
        },
        {"role": "user", "content": question},
    ]

    async def stream_response():
        async with openai.chat.completions.stream(
            model="gpt-4o",
            messages=messages,
            max_tokens=2048,
        ) as stream:
            async for chunk in stream:
                if chunk.choices and chunk.choices[0].delta.content:
                    yield f"data: {json.dumps({'content': chunk.choices[0].delta.content})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        stream_response(),
        media_type="text/event-stream",
        headers={"X-Accel-Buffering": "no"},
    )
```

### Zero-Downtime Deployments

Container Apps does zero-downtime deployments natively via **revision management**:

```bash
# Deploy new version with 0% initial traffic
az containerapp update \
  --name my-api \
  --resource-group rg-prod \
  --image myregistry.azurecr.io/my-api:v2.0.0 \
  --revision-suffix v2-0-0 \
  --traffic-weight latest=0  # New revision gets 0% traffic initially

# Validate new revision via direct URL
NEW_REVISION_URL=$(az containerapp revision show \
  --name my-api \
  --resource-group rg-prod \
  --revision my-api--v2-0-0 \
  --query "properties.fqdn" -o tsv)

curl https://$NEW_REVISION_URL/health

# Gradual rollout: send 10% traffic
az containerapp ingress traffic set \
  --name my-api \
  --resource-group rg-prod \
  --revision-weight my-api--v2-0-0=10 my-api--v1-9-0=90

# After validation: full cutover
az containerapp ingress traffic set \
  --name my-api \
  --resource-group rg-prod \
  --revision-weight my-api--v2-0-0=100

# Rollback instantly (no deployment needed — just shift traffic)
az containerapp ingress traffic set \
  --name my-api \
  --resource-group rg-prod \
  --revision-weight my-api--v1-9-0=100
```

### Cost Optimization

```bash
# Container Apps consumption plan:
# You pay only for CPU/memory when replicas are active
# Idle replicas after scale-down = $0

# For PostgreSQL: use Burstable tier for dev, General Purpose for prod
az postgres flexible-server create \
  --sku-name Standard_B2ms \  # Burstable — dev/test only
  --tier Burstable

# Stop dev server outside business hours (saves ~70%)
az postgres flexible-server stop --name db-dev --resource-group rg-dev
az postgres flexible-server start --name db-dev --resource-group rg-dev

# Azure OpenAI: use PTU for predictable high-volume, pay-as-you-go for variable
# GPT-4o-mini is 30x cheaper than GPT-4o — use it for classification/extraction

# Container App: allow scale to 0 for workers
az containerapp update \
  --name my-worker \
  --resource-group rg-prod \
  --min-replicas 0  # Zero cost when idle
```

---

## Quick Reference: Connection Strings and Endpoints

```bash
# PostgreSQL Flexible Server (via PgBouncer)
postgresql://{db-user}@{server-name}.postgres.database.azure.com:6432/{database}?sslmode=require

# Azure Cache for Redis (SSL)
rediss://{server-name}.redis.cache.windows.net:6380

# Azure Service Bus
{namespace}.servicebus.windows.net

# Azure Blob Storage
https://{account}.blob.core.windows.net

# Azure OpenAI
https://{resource-name}.openai.azure.com/

# Azure Key Vault
https://{vault-name}.vault.azure.net

# Cosmos DB
https://{account-name}.documents.azure.com:443/

# Application Insights (connection string from portal)
InstrumentationKey=...;IngestionEndpoint=https://eastus-8.in.applicationinsights.azure.com/
```

---

*Last updated: April 2026 | Companion rules: `~/.claude/rules/azure-patterns.md`*
