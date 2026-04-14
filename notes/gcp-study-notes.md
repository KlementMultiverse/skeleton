# GCP Complete — AI-Native Backend Engineer Guide (April 2026)

> **Audience:** Senior AI-native backend engineers deploying FastAPI + Vertex AI workloads on GCP.
> **Perspective:** Forward-looking, April 2026 patterns only. Deprecated paths are noted once and dropped.
> **Companion rules file:** `~/.claude/rules/gcp-patterns.md`

---

## Part 1: GCP Mental Model

### The Core Abstraction: Projects

If you're coming from AWS, the single most important mental shift is this:

**GCP project ≈ AWS account.**

In AWS, you isolate workloads by creating separate AWS accounts (one per environment, per team, per product). In GCP, you create separate **projects**. A project is the atomic unit of billing, IAM, and quota. Every GCP resource lives inside exactly one project. If you want prod, staging, and dev isolated, you create three projects.

```
AWS mental model:             GCP mental model:
Organization                  Cloud Organization
  └─ Management Account         └─ Organization (domain: company.com)
       └─ Member Accounts            └─ Folders (optional grouping)
            ├─ prod-account               ├─ production/
            ├─ staging-account            │    ├─ project: myapp-prod
            └─ dev-account                ├─ staging/
                                          │    ├─ project: myapp-staging
                                          └─ dev/
                                               └─ project: myapp-dev
```

### Resource Hierarchy

```
Cloud Organization (company.com)
  └─ Folders (optional, for grouping projects)
       ├─ Production
       │    └─ Projects: myapp-prod-001, myapp-prod-002
       ├─ Non-Production
       │    ├─ Projects: myapp-staging-001
       │    └─ Projects: myapp-dev-001
       └─ Shared
            └─ Projects: networking-shared, security-shared
```

**Why folders matter:** IAM policies inherit down the hierarchy. A policy applied at the Organization level applies to every folder and project. Applied at a folder, it cascades to all projects in that folder. This is additive — you cannot deny at a lower level what's granted above. Design your hierarchy before you have 50 projects.

### Billing Accounts

A billing account is separate from a project. One billing account can fund many projects. An organization can have multiple billing accounts (e.g., one per business unit, one per customer if you're a reseller).

```
Billing Account: billing-corp-central
  ├─ project: myapp-prod-001      (monthly cost: $X)
  ├─ project: myapp-staging-001   (monthly cost: $Y)
  └─ project: shared-networking   (monthly cost: $Z)
```

You link a project to a billing account at creation time. Unlinking it suspends resources after a grace period.

### IAM: Who Can Do What

GCP IAM has three concepts: **principals**, **roles**, and **policies**.

- **Principal** — who: a user (email), service account, group, or domain
- **Role** — what: a named collection of permissions (`roles/run.developer` grants 23 specific permissions)
- **Policy** — binding a principal to a role on a resource

```
Policy on project myapp-prod:
  member: user:klement@company.com
  role: roles/run.developer
  → Can deploy to Cloud Run in this project
```

**Predefined roles** are Google-managed collections (`roles/storage.objectAdmin`). **Custom roles** let you define exactly which permissions you need. Always prefer predefined roles until they're too broad, then create custom.

**Key role categories:**
- `roles/viewer` — read-only across most resources
- `roles/editor` — CRUD on most resources (avoid granting this broadly)
- `roles/owner` — full control including billing (never grant programmatically)
- Service-specific: `roles/run.invoker`, `roles/bigquery.dataEditor`, `roles/secretmanager.secretAccessor`

### Service Accounts

Service accounts are identities for non-human actors (your Cloud Run service, your Cloud Build pipeline, your Terraform runner). They are both a principal (can be granted roles) and a resource (can be impersonated).

```
Service Account: myapp-backend@myapp-prod-001.iam.gserviceaccount.com
  Roles granted to it:
    - roles/secretmanager.secretAccessor  (can read secrets)
    - roles/alloydb.client               (can connect to AlloyDB)
    - roles/pubsub.publisher             (can publish messages)
```

**April 2026 critical rule:** Never download service account key JSON files. Use Workload Identity Federation instead. Key files are a long-lived credential that can be exfiltrated. WIF is short-lived and scoped to the CI system's identity.

---

## Part 2: gcloud CLI (April 2026)

### Installation

```bash
# macOS with Homebrew
brew install --cask google-cloud-sdk

# Linux (curl installer)
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Verify
gcloud version
# Google Cloud SDK 500.0.0  (April 2026 current series)
```

### First-Time Setup

```bash
# Initialize — walks you through login + project selection
gcloud init

# Or separately:
gcloud auth login                    # browser-based OAuth for human users
gcloud config set project myapp-prod-001
gcloud config set compute/region us-central1
```

### Application Default Credentials (ADC)

ADC is the mechanism by which GCP client libraries authenticate. The library checks a credential chain:

1. `GOOGLE_APPLICATION_CREDENTIALS` env var (explicit key file — avoid in 2026)
2. `gcloud auth application-default login` credentials (local dev)
3. Attached service account (Cloud Run, GCE, etc.)
4. Workload Identity (GKE, Cloud Run with WIF)

```bash
# For local development — this is the right pattern
gcloud auth application-default login

# Now Python code using google-cloud-* libraries authenticates automatically
# No GOOGLE_APPLICATION_CREDENTIALS needed
```

When you call `gcloud auth application-default login`, it stores a credential at `~/.config/gcloud/application_default_credentials.json`. Your Python code using `google.auth.default()` or any `google-cloud-*` library picks this up automatically.

### Named Configurations

Switch between projects without re-running `gcloud init`:

```bash
# Create configurations for each environment
gcloud config configurations create prod
gcloud config set project myapp-prod-001
gcloud config set account klement@company.com

gcloud config configurations create staging
gcloud config set project myapp-staging-001
gcloud config set account klement@company.com

# Switch
gcloud config configurations activate prod
gcloud config configurations activate staging

# See current
gcloud config configurations list
gcloud config list
```

### Key Commands Reference

```bash
# Cloud Run
gcloud run deploy SERVICE_NAME \
  --image gcr.io/PROJECT/IMAGE:TAG \
  --region us-central1 \
  --platform managed \
  --service-account SA_EMAIL \
  --min-instances 1 \
  --max-instances 100 \
  --concurrency 80 \
  --cpu 2 \
  --memory 2Gi \
  --set-secrets "DB_PASSWORD=myapp-db-password:latest" \
  --vpc-connector my-vpc-connector \
  --ingress all

gcloud run services list --region us-central1
gcloud run services describe SERVICE_NAME --region us-central1
gcloud run services update-traffic SERVICE_NAME \
  --to-latest \
  --region us-central1

# Cloud Build
gcloud builds submit --tag gcr.io/PROJECT/IMAGE:TAG .
gcloud builds list --limit 10

# IAM
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member "serviceAccount:SA_EMAIL" \
  --role "roles/run.invoker"

gcloud iam service-accounts create my-sa \
  --display-name "My Service Account" \
  --project PROJECT_ID

# Secrets
gcloud secrets create my-secret --replication-policy automatic
echo -n "mysecretvalue" | gcloud secrets versions add my-secret --data-file=-
gcloud secrets versions access latest --secret my-secret

# Output formatting
gcloud run services list --format json
gcloud run services list --format "table(name,region,status)"
gcloud run services list --filter "region:us-central1"
gcloud projects list --filter "labels.env=prod"
```

### Service Account Impersonation (Not Key Files)

```bash
# Grant your user account permission to impersonate the service account
gcloud iam service-accounts add-iam-policy-binding \
  myapp-backend@myapp-prod.iam.gserviceaccount.com \
  --member "user:klement@company.com" \
  --role "roles/iam.serviceAccountTokenCreator"

# Now impersonate it (ADC will use the SA's identity)
gcloud auth application-default login \
  --impersonate-service-account myapp-backend@myapp-prod.iam.gserviceaccount.com

# Verify
gcloud auth application-default print-access-token
```

This is how you test your backend code locally with exactly the same permissions as production — no key files needed, no risk of credential leakage.

### Cloud Shell

Cloud Shell gives you a browser-based terminal with gcloud pre-installed, authenticated as your Google account, and persistent home directory storage. Zero installation, works from any machine.

Access: console.cloud.google.com → activate Cloud Shell (terminal icon, top right).

Limits: ephemeral VM (shuts down after 20 min idle), 5GB persistent `/home`, not suitable for long builds.

---

## Part 3: Compute — Cloud Run (Primary)

### Cloud Run Mental Model

Cloud Run is the April 2026 primary compute for backend services. Think of it as "managed containers with auto-scaling to zero." You provide a container image; GCP handles everything else — load balancing, TLS termination, scaling, patching.

```
Your Container Image
  → pushed to Artifact Registry
  → Cloud Run Service created/updated
  → Receives HTTPS endpoint: https://SERVICE-HASH-uc.a.run.app
  → Scales from 0 to N instances based on concurrent requests
  → Each instance handles up to `concurrency` requests simultaneously
```

### Gen 2 Runtime (Use This — Always)

Cloud Run Gen 2 (the default since 2024) runs on a full Linux kernel environment. Key differences from Gen 1:

- Full network access (not simulated)
- Supports background threads and processes after request completion (with CPU always-on)
- Larger instances (up to 32 vCPU, 128 GiB RAM)
- Supports shared memory
- Better gVisor isolation

```bash
# Gen 2 is the default — explicitly specify if you need to be sure
gcloud run deploy SERVICE \
  --execution-environment gen2 \
  ...
```

### CPU Allocation Modes

**Request-based (default):** CPU is throttled to near-zero between requests. Cheapest option. Suitable when all work happens during request handling.

**Always-on:** CPU runs continuously even between requests. More expensive but enables background tasks, WebSocket keepalives, scheduled work within the instance.

```bash
# Always-on CPU (needed for background tasks, WebSockets, etc.)
gcloud run deploy SERVICE \
  --cpu-throttling=false \
  ...

# Or in service.yaml:
# metadata:
#   annotations:
#     run.googleapis.com/cpu-throttling: "false"
```

### Concurrency

Each Cloud Run instance handles multiple requests simultaneously (unlike Lambda which is one-per-instance). Default is 80, max is 1000.

For FastAPI + asyncio: set concurrency to 80-200 depending on your async workload.
For CPU-bound workloads: set concurrency to match your thread count.

```bash
gcloud run deploy SERVICE --concurrency 100
```

### Minimum Instances (Cold Start Prevention)

Cold starts are real — a Python FastAPI container can take 2-4 seconds to start. For production APIs, set `min-instances 1` to always keep at least one warm instance.

```bash
gcloud run deploy SERVICE --min-instances 1 --max-instances 50
```

Cost implication: min-instances are billed even when idle (at a reduced rate). For production services with SLAs, it's worth it.

### Startup CPU Boost

When a new instance starts (cold start), give it extra CPU for the initialization phase:

```bash
gcloud run deploy SERVICE --cpu-boost
```

This temporarily allocates extra CPU during instance startup, cutting cold start time by 30-50% for CPU-bound startup tasks (loading ML models, warming caches).

### Cloud Run Jobs vs Services

**Service** — responds to HTTP requests, scales on traffic, has a persistent URL.
**Job** — runs to completion (exit code 0 = success), no HTTP endpoint. For batch processing, DB migrations, scheduled tasks.

```bash
# Create a Job
gcloud run jobs create my-migration-job \
  --image gcr.io/PROJECT/IMAGE:TAG \
  --region us-central1 \
  --service-account SA_EMAIL \
  --set-secrets "DB_URL=db-url:latest"

# Execute it
gcloud run jobs execute my-migration-job --region us-central1

# Schedule it (via Cloud Scheduler)
gcloud scheduler jobs create http nightly-cleanup \
  --schedule "0 2 * * *" \
  --uri "https://us-central1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/PROJECT_ID/jobs/my-cleanup-job:run" \
  --oauth-service-account-email SA_EMAIL
```

### VPC Connector (Serverless VPC Access)

Cloud Run instances are not inside your VPC by default. To reach private resources (AlloyDB, Memorystore Valkey, private GCE instances), you need a VPC connector:

```bash
# Create the connector (do this once per region)
gcloud compute networks vpc-access connectors create my-connector \
  --region us-central1 \
  --network default \
  --range 10.8.0.0/28

# Attach to Cloud Run service
gcloud run deploy SERVICE \
  --vpc-connector my-connector \
  --vpc-egress private-ranges-only \
  ...
```

`--vpc-egress private-ranges-only` routes only RFC1918 traffic through the VPC. `all-traffic` routes everything through the VPC (use with Cloud NAT for outbound internet).

### FastAPI + Cloud Run Complete Workflow

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install dependencies first (Docker layer caching)
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && \
    poetry config virtualenvs.create false && \
    poetry install --only main --no-interaction

COPY . .

# Cloud Run sets PORT env var (default 8080)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080", "--workers", "1"]
```

```bash
# Build and push
gcloud builds submit \
  --tag us-central1-docker.pkg.dev/myapp-prod/myapp/backend:$(git rev-parse --short HEAD) \
  .

# Deploy
gcloud run deploy myapp-backend \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/backend:$(git rev-parse --short HEAD) \
  --region us-central1 \
  --service-account myapp-backend@myapp-prod.iam.gserviceaccount.com \
  --min-instances 1 \
  --max-instances 50 \
  --concurrency 100 \
  --cpu 2 \
  --memory 2Gi \
  --cpu-boost \
  --set-secrets "DATABASE_URL=alloydb-url:latest,REDIS_URL=valkey-url:latest" \
  --vpc-connector my-vpc-connector \
  --vpc-egress private-ranges-only \
  --allow-unauthenticated
```

### Multi-Region with Global Load Balancer

```bash
# Deploy to multiple regions
for region in us-central1 europe-west1 asia-east1; do
  gcloud run deploy myapp-backend \
    --region $region \
    --image us-central1-docker.pkg.dev/myapp-prod/myapp/backend:latest \
    --no-traffic  # don't send traffic yet
done

# Create HTTPS load balancer with NEGs (Network Endpoint Groups)
# This is typically done via Terraform (see Part 10)
```

### Decision: Cloud Run vs App Engine vs Cloud Functions

| Use Case | Cloud Run | App Engine | Cloud Functions |
|---|---|---|---|
| FastAPI REST API | **Yes** | Possible (Gen 2) | No |
| Background async worker | **Yes** (Jobs/Services) | No | No |
| HTTP triggered function | Yes | No | **Yes** |
| WebSockets | **Yes** (always-on CPU) | No | No |
| ML model serving | **Yes** (GPU support) | No | No |
| Legacy Python 2/3 app | No | **Yes** (standard) | No |

**Decision rule:** Default to Cloud Run for everything new. App Engine only if you're migrating an existing App Engine app. Cloud Functions only for very simple event-triggered code (< 100 lines, single HTTP handler).

---

## Part 4: Database Selection (April 2026)

### The April 2026 Database Landscape

```
New AI/RAG workload? → AlloyDB for PostgreSQL (pgvector + SCANN built-in)
Simple relational needs? → Cloud SQL for PostgreSQL (less overhead, simpler ops)
Document/realtime/mobile? → Firestore (Native mode ONLY — Datastore mode is deprecated)
Analytics/ML in-database? → BigQuery
Caching/sessions? → Memorystore for Valkey
Vector search at massive scale? → Vertex AI Vector Search (dedicated)
```

### AlloyDB for PostgreSQL — The AI-Native Choice

AlloyDB is Google's fully-managed PostgreSQL-compatible database, built on a disaggregated storage architecture. For April 2026 AI workloads, it's the recommended choice over Cloud SQL because of its built-in AI capabilities.

**Why AlloyDB over Cloud SQL for AI workloads:**

1. **AlloyDB AI** — pgvector support with SCANN (Scale-Accurate Near Neighbor) index, which is significantly faster than HNSW for large vector datasets (> 1M vectors)
2. **Columnar Engine** — built-in columnar store for analytics queries (10-100x faster for aggregate queries vs row store)
3. **Instant failover** — sub-second failover, compared to Cloud SQL's 30-60 second failover
4. **Autoscaling read pools** — read replicas scale automatically based on CPU utilization
5. **AlloyDB Omni** — run the same AlloyDB engine on-prem or other clouds for hybrid scenarios

**When Cloud SQL is enough:**
- Smaller datasets (< 100GB)
- No vector/AI needs
- Budget-constrained (AlloyDB is ~2x Cloud SQL price per compute)
- Team unfamiliar with AlloyDB operational quirks

**AlloyDB Architecture:**

```
Primary Instance (writes)
  └─ Disaggregated Storage (distributed across zones — automatic replication)
       └─ Read Pool (autoscaling read instances share same storage)
            ├─ Read Pool Instance 1
            ├─ Read Pool Instance 2
            └─ Read Pool Instance N (scales automatically)
```

**AlloyDB AI — pgvector with SCANN:**

```sql
-- Enable pgvector extension
CREATE EXTENSION vector;

-- Create table with vector column
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  content TEXT NOT NULL,
  embedding vector(1536),  -- OpenAI ada-002 or Vertex AI text-embedding-004
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Create SCANN index (AlloyDB-specific, much faster than HNSW at scale)
CREATE INDEX ON documents
USING scann (embedding cosine)
WITH (num_leaves = 500);  -- tune based on dataset size

-- Similarity search
SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

SCANN vs HNSW on AlloyDB at 10M vectors:
- HNSW build time: ~4 hours
- SCANN build time: ~20 minutes
- Query throughput: SCANN 3x higher at same recall

**Connecting to AlloyDB from Cloud Run:**

AlloyDB uses the AlloyDB Auth Proxy (same concept as Cloud SQL Auth Proxy) for secure connections without IP allowlisting:

```python
# app/database.py
import asyncpg
import asyncio
from google.cloud.alloydb.connector import AsyncConnector

async def create_alloydb_pool():
    """Create asyncpg connection pool via AlloyDB Auth Proxy."""
    connector = AsyncConnector()

    async def getconn():
        conn = await connector.connect(
            # Format: projects/PROJECT/locations/REGION/clusters/CLUSTER/instances/INSTANCE
            "projects/myapp-prod/locations/us-central1/clusters/myapp-cluster/instances/myapp-primary",
            "asyncpg",
            user="myapp_user",
            password=os.environ["DB_PASSWORD"],
            db="myapp",
        )
        return conn

    pool = await asyncpg.create_pool(
        dsn=None,
        connect=getconn,
        min_size=5,
        max_size=20,
        max_inactive_connection_lifetime=300,
    )
    return pool, connector
```

```bash
# Grant Cloud Run SA access to AlloyDB
gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-backend@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/alloydb.client"
```

### Cloud SQL for PostgreSQL — The Simpler Choice

When you don't need AlloyDB's AI features, Cloud SQL is simpler to operate:

```bash
# Create Cloud SQL instance
gcloud sql instances create myapp-db \
  --database-version POSTGRES_16 \
  --tier db-custom-2-8192 \
  --region us-central1 \
  --availability-type REGIONAL \  # HA with automatic failover
  --backup-start-time 03:00 \
  --enable-bin-log

# Create database and user
gcloud sql databases create myapp --instance myapp-db
gcloud sql users create myapp_user --instance myapp-db --password=$(openssl rand -base64 32)
```

**Cloud SQL Auth Proxy (same pattern as AlloyDB):**

```python
from google.cloud.sql.connector import Connector, IPTypes
import asyncpg

async def create_pool():
    connector = Connector()

    async def getconn():
        conn = await connector.connect_async(
            "myapp-prod:us-central1:myapp-db",
            "asyncpg",
            user="myapp_user",
            password=os.environ["DB_PASSWORD"],
            db="myapp",
            ip_type=IPTypes.PRIVATE,  # Use private IP via VPC connector
        )
        return conn

    return await asyncpg.create_pool(connect=getconn, min_size=5, max_size=20)
```

### Firestore — Native Mode (Datastore Mode is Dead)

**Firestore in Datastore mode is deprecated for new projects as of 2024.** Do not use it. Use **Firestore Native mode** only.

Firestore Native mode is a document database:

```
Collection: users
  Document: user-123
    Fields:
      name: "Klement"
      email: "k@example.com"
      preferences: {theme: "dark", language: "en"}
  Subcollection: sessions
    Document: session-abc
      Fields: ...
```

**When Firestore over AlloyDB:**
- Mobile/offline sync (Firestore has native SDK for iOS/Android/web with offline support)
- Real-time listeners (WebSocket-based change subscriptions)
- Schemaless rapid prototyping
- Globally distributed reads without read replicas
- No complex relational queries needed

**Python Firestore:**

```python
from google.cloud import firestore

db = firestore.AsyncClient()

# Write
await db.collection("users").document("user-123").set({
    "name": "Klement",
    "email": "k@example.com",
    "created_at": firestore.SERVER_TIMESTAMP,
})

# Read
doc = await db.collection("users").document("user-123").get()
if doc.exists:
    user = doc.to_dict()

# Query
users = db.collection("users").where("active", "==", True).order_by("created_at").limit(10)
async for doc in users.stream():
    print(doc.to_dict())

# Real-time listener (Cloud Run with always-on CPU)
def on_snapshot(col_snapshot, changes, read_time):
    for change in changes:
        if change.type.name == "ADDED":
            print(f"New doc: {change.document.to_dict()}")

db.collection("events").on_snapshot(on_snapshot)
```

### BigQuery — Serverless Analytics + ML

BigQuery is GCP's serverless data warehouse. In April 2026, it's not just for analytics — AI engineers use it for:

1. **BigQuery Vector Search** (GA in April 2026) — similarity search on embedding columns
2. **BigQuery ML** — train and serve ML models directly in SQL
3. **Long-term trace/log storage** — route Cloud Logging to BigQuery for retention
4. **Embeddings at scale** — store and search millions of embeddings cost-effectively

```python
from google.cloud import bigquery

client = bigquery.Client()

# Query with BigQuery Vector Search (April 2026)
query = """
SELECT
  base.id,
  base.content,
  distance
FROM VECTOR_SEARCH(
  TABLE myapp_prod.documents,
  'embedding',
  (SELECT embedding FROM myapp_prod.queries WHERE id = @query_id),
  top_k => 10,
  distance_type => 'COSINE'
)
"""

job_config = bigquery.QueryJobConfig(
    query_parameters=[
        bigquery.ScalarQueryParameter("query_id", "STRING", query_id)
    ]
)
results = client.query(query, job_config=job_config).result()
```

**BigQuery ML example:**

```sql
-- Train a classification model entirely in SQL
CREATE OR REPLACE MODEL myapp_prod.churn_model
OPTIONS(model_type='LOGISTIC_REG', input_label_cols=['churned'])
AS SELECT
  days_since_last_login,
  total_api_calls_30d,
  subscription_tier,
  churned
FROM myapp_prod.user_features;

-- Predict
SELECT * FROM ML.PREDICT(
  MODEL myapp_prod.churn_model,
  (SELECT * FROM myapp_prod.user_features WHERE user_id = 'u123')
);
```

**Streaming inserts vs batch:**
- Streaming inserts: data visible immediately, higher cost per row, good for real-time dashboards
- Batch loads (GCS → BQ): cheaper, higher throughput, data available after job completes (minutes)
- Storage Write API (recommended): exactly-once semantics, high throughput, lower cost than streaming inserts

### Memorystore for Valkey — April 2026 Standard

Redis OSS was replaced by Valkey (a Redis fork maintained by the Linux Foundation) in Google Cloud Memorystore. The API is identical to Redis — same commands, same client libraries.

```bash
# Create Memorystore Valkey instance
gcloud redis instances create myapp-valkey \
  --size 1 \  # GB
  --region us-central1 \
  --redis-version redis_7_2 \  # Valkey is compatible with Redis 7.2
  --network default \
  --tier standard  # For HA; basic for dev

# Get connection details
gcloud redis instances describe myapp-valkey --region us-central1
# Note the host IP and port (6379)
```

**Serverless Memorystore (April 2026):** No capacity planning needed, scales automatically:

```bash
gcloud redis clusters create myapp-valkey-serverless \
  --region us-central1 \
  --network default \
  --replica-count 1
```

**Python connection (same as Redis):**

```python
import redis.asyncio as redis

pool = redis.ConnectionPool.from_url(
    f"redis://{os.environ['VALKEY_HOST']}:6379",
    max_connections=20,
    decode_responses=True,
)
client = redis.Redis(connection_pool=pool)

# Standard Redis commands work identically
await client.setex("session:abc", 3600, json.dumps(session_data))
data = await client.get("session:abc")
```

---

## Part 5: Vertex AI and Gemini (April 2026)

### Gemini Model Selection (April 2026)

```
Gemini 2.0 Flash       → Cost-effective default. Use for most tasks.
                          1M context window, fast, cheap.
                          Model ID: gemini-2.0-flash-001

Gemini 2.5 Pro         → Complex reasoning, long documents, code generation.
                          2M context window, slow, expensive.
                          Model ID: gemini-2.5-pro-001

Gemini 2.0 Flash-Lite  → Ultra-cheap, lower quality. For classification, filtering.
                          Model ID: gemini-2.0-flash-lite-001
```

**Decision rule:** Start with Flash. Upgrade to Pro only when Flash fails your quality bar on specific tasks. The cost difference is ~10x.

### Vertex AI SDK Setup

```bash
pip install google-cloud-aiplatform
```

```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part, SafetySetting, HarmCategory

# Initialize (uses ADC automatically)
vertexai.init(project="myapp-prod", location="us-central1")
```

### Basic Generation

```python
from vertexai.generative_models import GenerativeModel

model = GenerativeModel(
    model_name="gemini-2.0-flash-001",
    system_instruction="You are a helpful assistant for a software company.",
)

response = model.generate_content(
    "Explain the difference between AlloyDB and Cloud SQL in 3 bullet points.",
    generation_config={
        "max_output_tokens": 1024,
        "temperature": 0.1,
        "top_p": 0.95,
    },
)
print(response.text)
```

### Streaming Generation (Production Pattern)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import vertexai
from vertexai.generative_models import GenerativeModel
import asyncio

app = FastAPI()
vertexai.init(project="myapp-prod", location="us-central1")

async def gemini_stream(prompt: str):
    """Async generator for Gemini streaming."""
    model = GenerativeModel("gemini-2.0-flash-001")

    # generate_content with stream=True returns a generator
    # Wrap in executor since Vertex AI SDK is sync
    loop = asyncio.get_event_loop()

    def generate():
        return model.generate_content(
            prompt,
            stream=True,
            generation_config={"max_output_tokens": 2048, "temperature": 0.7},
        )

    response_stream = await loop.run_in_executor(None, generate)

    for chunk in response_stream:
        if chunk.text:
            yield f"data: {chunk.text}\n\n"

    yield "data: [DONE]\n\n"

@app.get("/stream")
async def stream_endpoint(prompt: str):
    return StreamingResponse(
        gemini_stream(prompt),
        media_type="text/event-stream",
        headers={"X-Accel-Buffering": "no"},
    )
```

### Async Vertex AI (April 2026)

The Vertex AI SDK added native async support. Use it instead of run_in_executor:

```python
from vertexai.generative_models import GenerativeModel

async def generate_async(prompt: str) -> str:
    model = GenerativeModel("gemini-2.0-flash-001")
    response = await model.generate_content_async(
        prompt,
        generation_config={"max_output_tokens": 1024, "temperature": 0.1},
    )
    return response.text

async def stream_async(prompt: str):
    model = GenerativeModel("gemini-2.0-flash-001")
    async for chunk in await model.generate_content_async(
        prompt,
        stream=True,
        generation_config={"max_output_tokens": 2048},
    ):
        if chunk.text:
            yield chunk.text
```

### Multimodal: Text + Images + Documents

```python
from vertexai.generative_models import GenerativeModel, Part

model = GenerativeModel("gemini-2.0-flash-001")

# Image from GCS
image_part = Part.from_uri(
    "gs://myapp-uploads/invoice-123.pdf",
    mime_type="application/pdf"
)

response = model.generate_content([
    image_part,
    "Extract all line items, quantities, and prices from this invoice. Return as JSON.",
])

# Image from bytes
with open("image.jpg", "rb") as f:
    image_bytes = f.read()

image_part = Part.from_data(image_bytes, mime_type="image/jpeg")
response = model.generate_content([image_part, "Describe this image."])
```

### Safety Settings

```python
from vertexai.generative_models import SafetySetting, HarmCategory, HarmBlockThreshold

safety_settings = [
    SafetySetting(
        category=HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
        threshold=HarmBlockThreshold.BLOCK_ONLY_HIGH,
    ),
    SafetySetting(
        category=HarmCategory.HARM_CATEGORY_HARASSMENT,
        threshold=HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
    ),
]

response = model.generate_content(
    prompt,
    safety_settings=safety_settings,
)
```

### Grounding with Google Search (April 2026)

Ground Gemini responses in real-time web search results:

```python
from vertexai.generative_models import GenerativeModel, Tool, grounding

model = GenerativeModel("gemini-2.0-flash-001")

# Enable Google Search grounding
google_search_tool = Tool.from_google_search_retrieval(
    grounding.GoogleSearchRetrieval()
)

response = model.generate_content(
    "What is the current GCP pricing for Cloud Run in us-central1?",
    tools=[google_search_tool],
    generation_config={"temperature": 0},  # Low temp for factual grounded responses
)

print(response.text)
# Response includes citations from web search
for attribution in response.candidates[0].grounding_metadata.search_entry_point:
    print(attribution)
```

### Vertex AI Vector Search (Formerly Matching Engine)

For RAG workloads with more than ~5M vectors, AlloyDB pgvector may not be sufficient. Vertex AI Vector Search is a dedicated ANN (approximate nearest neighbor) service:

```python
from google.cloud import aiplatform

# Deploy an index (pre-built from embeddings stored in GCS)
aiplatform.init(project="myapp-prod", location="us-central1")

# Create index from embeddings
index = aiplatform.MatchingEngineIndex.create_tree_ah_index(
    display_name="document-embeddings",
    contents_delta_uri="gs://myapp-embeddings/",
    dimensions=1536,
    approximate_neighbors_count=150,
)

# Create index endpoint (online serving)
index_endpoint = aiplatform.MatchingEngineIndexEndpoint.create(
    display_name="document-search",
    public_endpoint_enabled=True,
)

index_endpoint.deploy_index(
    index=index,
    deployed_index_id="document_embeddings_v1",
)

# Query
response = index_endpoint.find_neighbors(
    deployed_index_id="document_embeddings_v1",
    queries=[query_embedding],  # list of float
    num_neighbors=10,
)
for neighbor in response[0]:
    print(f"ID: {neighbor.id}, Distance: {neighbor.distance}")
```

**AlloyDB pgvector vs Vertex AI Vector Search:**

| Factor | AlloyDB pgvector + SCANN | Vertex AI Vector Search |
|---|---|---|
| Dataset size | < 10M vectors | 10M–1B+ vectors |
| Joins with relational data | Native SQL | Extra round trip |
| Operational complexity | Low (same DB) | High (separate service) |
| Cost | DB compute + storage | Per-query pricing |
| Latency | ~10-50ms | ~5-20ms |

**Recommendation:** Start with AlloyDB pgvector. Move to Vertex AI Vector Search only when you hit AlloyDB's limits.

### Vertex AI Reasoning Engine (April 2026)

Reasoning Engine is a managed environment for running LangChain/LangGraph agents on Vertex AI. You define an agent class, deploy it, and Vertex AI handles scaling and execution:

```python
import vertexai
from vertexai.preview import reasoning_engines

vertexai.init(project="myapp-prod", location="us-central1", staging_bucket="gs://myapp-staging")

class MyAgent(reasoning_engines.Queryable):
    def set_up(self):
        from vertexai.generative_models import GenerativeModel
        self.model = GenerativeModel("gemini-2.0-flash-001")
        # Set up tools, chain, graph here

    def query(self, *, user_message: str) -> str:
        response = self.model.generate_content(user_message)
        return response.text

# Deploy
agent = reasoning_engines.ReasoningEngine.create(
    MyAgent(),
    requirements=["google-cloud-aiplatform[langchain]"],
    display_name="my-agent",
)

# Query the deployed agent
response = agent.query(user_message="What is AlloyDB?")
```

### Model Garden — Third-Party Models on Vertex AI

Vertex AI Model Garden provides access to Claude (Anthropic), Llama, Mistral, and other models through a unified API:

```python
import anthropic

# Claude on Vertex AI via Anthropic SDK
client = anthropic.AnthropicVertex(
    project_id="myapp-prod",
    region="us-east5",  # Claude available in us-east5 as of April 2026
)

message = client.messages.create(
    model="claude-sonnet-4-5@20251101",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Explain Workload Identity Federation."}],
)
print(message.content[0].text)
```

Using Claude on Vertex AI means billing goes through GCP (not Anthropic directly) and the model is deployed within your GCP project boundary — useful for compliance.

---

## Part 6: Messaging — Pub/Sub, Cloud Tasks, Eventarc

### Pub/Sub — The GCP Messaging Backbone

Pub/Sub is the primary async messaging service on GCP. It's a publish-subscribe system: publishers send messages to topics; subscribers receive them through subscriptions.

```
Publisher (Cloud Run service)
  → publishes message to Topic: "user-events"

Subscription 1: "email-worker" (push to Cloud Run /webhook/email)
Subscription 2: "analytics-worker" (pull from Cloud Run analytics service)
Subscription 3: "bigquery-sub" (direct to BigQuery table)
```

**At-least-once delivery:** Pub/Sub guarantees each message is delivered at least once. Your consumers must be idempotent. Use a message attribute or database record to deduplicate.

**Ordering:** Not guaranteed by default. Enable message ordering per topic + subscription and use an ordering key for ordered delivery.

### Publish a Message

```python
from google.cloud import pubsub_v1
import json

publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path("myapp-prod", "user-events")

message = {
    "event": "user.registered",
    "user_id": "user-123",
    "timestamp": "2026-04-14T10:00:00Z",
}

future = publisher.publish(
    topic_path,
    data=json.dumps(message).encode("utf-8"),
    # Attributes for filtering
    event_type="user.registered",
    tenant_id="tenant-abc",
)
message_id = future.result()  # Blocks until acknowledgment
```

**Async publishing (high throughput):**

```python
from google.cloud import pubsub_v1
from concurrent.futures import Future
from typing import Callable

publisher = pubsub_v1.PublisherClient(
    batch_settings=pubsub_v1.types.BatchSettings(
        max_messages=100,
        max_bytes=1024 * 1024,  # 1MB
        max_latency=0.01,  # 10ms — tune for throughput vs latency
    )
)

futures: list[Future] = []
for event in events:
    future = publisher.publish(topic_path, data=json.dumps(event).encode())
    futures.append(future)

# Wait for all to publish
for future in futures:
    future.result()
```

### Push Subscriptions (Preferred for Cloud Run)

Push subscriptions have Pub/Sub push messages to your Cloud Run endpoint. No polling needed, auto-scales with message volume:

```python
# FastAPI handler for Pub/Sub push
from fastapi import FastAPI, Request, HTTPException
import base64
import json

app = FastAPI()

@app.post("/pubsub/user-events")
async def handle_user_event(request: Request):
    body = await request.json()

    # Pub/Sub wraps the message in an envelope
    if "message" not in body:
        raise HTTPException(400, "Invalid Pub/Sub message format")

    message = body["message"]
    data = json.loads(base64.b64decode(message["data"]).decode("utf-8"))

    event_type = message.get("attributes", {}).get("event_type")

    if event_type == "user.registered":
        await handle_user_registered(data)
    elif event_type == "user.deleted":
        await handle_user_deleted(data)
    else:
        # Unknown event — acknowledge to avoid retry storm
        pass

    # Return 200/204 to acknowledge. Return 4xx/5xx to nack (message will be redelivered)
    return {"status": "ok"}
```

```bash
# Create push subscription
gcloud pubsub subscriptions create email-worker \
  --topic user-events \
  --push-endpoint https://myapp-backend-xxx-uc.a.run.app/pubsub/user-events \
  --push-auth-service-account myapp-backend@myapp-prod.iam.gserviceaccount.com \
  --ack-deadline 60 \
  --message-retention-duration 7d
```

### Dead Letter Topics

When a message fails to be processed after N attempts, route it to a dead-letter topic:

```bash
# Create DLT
gcloud pubsub topics create user-events-dead-letter

# Create subscription with DLT
gcloud pubsub subscriptions create email-worker \
  --topic user-events \
  --dead-letter-topic user-events-dead-letter \
  --max-delivery-attempts 5 \
  --push-endpoint https://...
```

### BigQuery Subscription (Direct Streaming to BQ)

Route Pub/Sub messages directly to BigQuery without a consumer service:

```bash
gcloud pubsub subscriptions create events-to-bq \
  --topic user-events \
  --bigquery-table myapp-prod:events.user_events \
  --use-topic-schema \
  --drop-unknown-fields
```

### Cloud Tasks — Targeted HTTP Queue

Cloud Tasks is different from Pub/Sub. Use it when you need:
- Exactly one HTTP target per task (no fan-out)
- Deduplication (same task ID = not enqueued twice)
- Per-task scheduling (delay until a future time)
- Per-task retry configuration

```python
from google.cloud import tasks_v2
from datetime import datetime, timedelta

client = tasks_v2.CloudTasksClient()
queue_path = client.queue_path("myapp-prod", "us-central1", "email-queue")

task = {
    "http_request": {
        "http_method": tasks_v2.HttpMethod.POST,
        "url": "https://myapp-backend-xxx-uc.a.run.app/tasks/send-email",
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"user_id": "user-123", "template": "welcome"}).encode(),
        "oidc_token": {
            "service_account_email": "myapp-backend@myapp-prod.iam.gserviceaccount.com",
        },
    },
    "name": f"{queue_path}/tasks/send-welcome-user-123",  # Deduplication key
    "schedule_time": {
        "seconds": int((datetime.utcnow() + timedelta(minutes=5)).timestamp())
    },
}

response = client.create_task(request={"parent": queue_path, "task": task})
```

**Pub/Sub vs Cloud Tasks decision:**

| Need | Use |
|---|---|
| Fan-out (multiple consumers per event) | Pub/Sub |
| Multiple services react to same event | Pub/Sub |
| Single HTTP endpoint delivery | Cloud Tasks |
| Deduplication by task ID | Cloud Tasks |
| Schedule task for future time | Cloud Tasks |
| High throughput (> 10k/s) | Pub/Sub |

### Eventarc — Native GCP Event Routing

Eventarc routes events from GCP services directly to Cloud Run using the Cloud Events standard. No polling, no custom pub/sub setup:

```bash
# Trigger Cloud Run when a GCS object is created
gcloud eventarc triggers create process-upload \
  --destination-run-service myapp-backend \
  --destination-run-region us-central1 \
  --event-filters "type=google.cloud.storage.object.v1.finalized" \
  --event-filters "bucket=myapp-uploads" \
  --service-account myapp-backend@myapp-prod.iam.gserviceaccount.com
```

```python
# FastAPI handler for Eventarc events
from cloudevents.http import from_http, CloudEvent
from fastapi import Request

@app.post("/events/gcs-upload")
async def handle_gcs_upload(request: Request):
    body = await request.body()
    headers = dict(request.headers)

    event: CloudEvent = from_http(headers, body)
    data = event.data

    bucket = data["bucket"]
    name = data["name"]
    size = data["size"]

    await process_upload(bucket=bucket, name=name, size=int(size))
    return {"status": "processed"}
```

---

## Part 7: Storage — Cloud Storage

### GCS Storage Classes

Cloud Storage uses storage classes for cost optimization based on access frequency:

| Class | Access | Cost | Minimum Storage |
|---|---|---|---|
| Standard | Frequent | High | None |
| Nearline | Monthly | Medium | 30 days |
| Coldline | Quarterly | Low | 90 days |
| Archive | Yearly | Very low | 365 days |

Set lifecycle rules to automatically transition objects:

```bash
# Create bucket with lifecycle rules
gcloud storage buckets create gs://myapp-uploads \
  --location us-central1 \
  --uniform-bucket-level-access \
  --lifecycle-file lifecycle.json
```

```json
// lifecycle.json
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
      "condition": {"age": 90}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365}
    }
  ]
}
```

### Uniform Bucket-Level Access

Always use uniform bucket-level access (not per-object ACLs). Uniform access means IAM controls all access, no legacy ACLs:

```bash
# Enable uniform access (should be on by default for new buckets)
gcloud storage buckets update gs://myapp-uploads \
  --uniform-bucket-level-access
```

### Signed URLs (V4)

Generate signed URLs for temporary access to private objects without requiring a GCP account:

```python
from google.cloud import storage
from datetime import timedelta

def generate_upload_url(bucket_name: str, object_name: str, expiry_minutes: int = 15) -> str:
    """Generate a signed URL for direct client upload."""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(object_name)

    url = blob.generate_signed_url(
        version="v4",
        expiration=timedelta(minutes=expiry_minutes),
        method="PUT",
        content_type="application/octet-stream",
    )
    return url

def generate_download_url(bucket_name: str, object_name: str, expiry_hours: int = 1) -> str:
    """Generate a signed URL for temporary download access."""
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(object_name)

    url = blob.generate_signed_url(
        version="v4",
        expiration=timedelta(hours=expiry_hours),
        method="GET",
    )
    return url
```

Note: V4 signed URLs have a maximum expiry of 7 days. For longer-lived access, use IAM or public access.

### GCS as Data Lake

Use BigQuery external tables to query GCS data directly without loading it:

```sql
-- Create external table from GCS Parquet files
CREATE EXTERNAL TABLE myapp_prod.raw_events
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://myapp-data-lake/events/*.parquet']
);

-- Query directly
SELECT event_type, COUNT(*) as count
FROM myapp_prod.raw_events
WHERE date = '2026-04-14'
GROUP BY event_type;
```

---

## Part 8: Security — IAM, Secret Manager, Workload Identity

### Workload Identity Federation — The April 2026 Standard

**Never download service account key JSON files.** Instead, use Workload Identity Federation (WIF) to let external identities (GitHub Actions, other clouds) impersonate GCP service accounts.

How it works:
```
GitHub Actions OIDC token (proves "I am workflow X in repo Y")
  → WIF Pool validates the OIDC token against GitHub's JWKS endpoint
  → WIF maps the token to GCP service account
  → Short-lived GCP access token issued (valid 1 hour)
  → No JSON key file ever created
```

**Setup:**

```bash
# 1. Create WIF pool
gcloud iam workload-identity-pools create github-pool \
  --project myapp-prod \
  --location global \
  --display-name "GitHub Actions Pool"

# 2. Create OIDC provider in the pool
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --project myapp-prod \
  --location global \
  --workload-identity-pool github-pool \
  --display-name "GitHub Provider" \
  --attribute-mapping "google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri "https://token.actions.githubusercontent.com"

# 3. Get the WIF provider resource name
gcloud iam workload-identity-pools providers describe github-provider \
  --project myapp-prod \
  --location global \
  --workload-identity-pool github-pool \
  --format "value(name)"
# → projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider

# 4. Grant service account impersonation to the GitHub repo
gcloud iam service-accounts add-iam-policy-binding \
  deploy-sa@myapp-prod.iam.gserviceaccount.com \
  --project myapp-prod \
  --role roles/iam.workloadIdentityUser \
  --member "principalSet://iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/github-pool/attribute.repository/myorg/myrepo"
```

**GitHub Actions workflow:**

```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloud Run

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write  # Required for WIF

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider
          service_account: deploy-sa@myapp-prod.iam.gserviceaccount.com

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Build and Deploy
        run: |
          gcloud builds submit --tag us-central1-docker.pkg.dev/myapp-prod/myapp/backend:${{ github.sha }}
          gcloud run deploy myapp-backend \
            --image us-central1-docker.pkg.dev/myapp-prod/myapp/backend:${{ github.sha }} \
            --region us-central1
```

### Secret Manager

GCP's Secret Manager stores secrets with versioning, rotation, and IAM-controlled access:

```bash
# Create a secret
gcloud secrets create alloydb-password \
  --replication-policy automatic \
  --project myapp-prod

# Add a version (the actual secret value)
echo -n "$(openssl rand -base64 32)" | \
  gcloud secrets versions add alloydb-password --data-file=- --project myapp-prod

# Grant access to Cloud Run SA
gcloud secrets add-iam-policy-binding alloydb-password \
  --project myapp-prod \
  --member "serviceAccount:myapp-backend@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/secretmanager.secretAccessor"
```

**Access in Python:**

```python
from google.cloud import secretmanager

def get_secret(secret_id: str, project_id: str = "myapp-prod", version: str = "latest") -> str:
    """Access a secret version."""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version}"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("utf-8")

# Use at startup, not in request handlers (avoid per-request Secret Manager calls)
DB_PASSWORD = get_secret("alloydb-password")
```

**Mounting secrets as environment variables (preferred for Cloud Run):**

```bash
gcloud run deploy myapp-backend \
  --set-secrets "DB_PASSWORD=alloydb-password:latest,VALKEY_URL=valkey-url:latest"
```

This injects the secret value as an environment variable at instance start — the application code just reads `os.environ["DB_PASSWORD"]`, no Secret Manager SDK needed in the request path.

### Secret Rotation with Pub/Sub

Automate secret rotation notifications:

```bash
# Add rotation notification to a secret
gcloud secrets update alloydb-password \
  --add-topics projects/myapp-prod/topics/secret-rotation \
  --rotation-period 2592000s \  # 30 days
  --next-rotation-time 2026-05-14T00:00:00Z
```

Your Cloud Run service subscribes to `secret-rotation` and handles rotation by:
1. Generating a new password in the DB
2. Adding a new secret version
3. Updating Cloud Run to use the new version
4. Destroying the old version

### VPC Service Controls

For sensitive workloads, VPC Service Controls create an API-level perimeter — preventing data exfiltration even if credentials are compromised:

```bash
# Create an access policy (org-level, done once)
gcloud access-context-manager policies create \
  --organization ORG_ID \
  --title "myapp-perimeter-policy"

# Create a service perimeter
gcloud access-context-manager perimeters create myapp-perimeter \
  --policy POLICY_ID \
  --title "MyApp Production Perimeter" \
  --resources "projects/123456789" \
  --restricted-services bigquery.googleapis.com,storage.googleapis.com
```

Resources inside the perimeter can only be accessed by identities in the perimeter. A compromised credential outside the perimeter cannot exfiltrate BigQuery data.

### Organization Policy Constraints

Enforce guardrails across all projects in your organization:

```bash
# Prevent service account key creation org-wide
gcloud org-policies set-policy \
  --organization=ORG_ID \
  constraints/iam.disableServiceAccountKeyCreation.yaml

# Require OS Login for all VMs
gcloud org-policies set-policy \
  --organization=ORG_ID \
  constraints/compute.requireOsLogin.yaml

# Restrict which regions can be used
gcloud org-policies set-policy \
  --folder=FOLDER_ID \
  constraints/gcp.resourceLocations.yaml
```

---

## Part 9: GCP MCP Servers (April 2026)

### Available GCP MCP Servers

As of April 2026, two primary MCP server packages provide GCP tooling:

**`google-cloud-mcp`** — Official Google MCP server with native GCP service tools:
- Cloud Storage (read, write, list objects)
- BigQuery (query execution, table listing)
- Pub/Sub (publish messages, list topics)
- Secret Manager (read secrets)
- Cloud Run (list services, describe services)

**`@google-cloud/mcp-tools`** (Node.js) — gcloud CLI-based operations, wraps gcloud commands.

### Using google-cloud-mcp with Claude Code

```json
// .mcp.json (project-level MCP config)
{
  "mcpServers": {
    "google-cloud": {
      "command": "python",
      "args": ["-m", "google_cloud_mcp"],
      "env": {
        "GOOGLE_CLOUD_PROJECT": "myapp-prod",
        "GOOGLE_APPLICATION_CREDENTIALS": ""  // Uses ADC (gcloud auth app-default login)
      }
    }
  }
}
```

### Claude Code + GCP MCP Workflow

With the GCP MCP server running, you can ask Claude Code natural language questions about your GCP resources:

```
"List all Cloud Run services in us-central1 and show their current traffic splits"
"Query BigQuery for the top 10 slowest API endpoints in the last 24 hours"
"What secrets are defined in Secret Manager for project myapp-prod?"
"Read the latest 100 log entries for service myapp-backend"
```

### Use Cases

**Development workflow:**
- Query BigQuery analytics without leaving the editor
- Read and write test data to GCS
- Check Pub/Sub topic/subscription configuration
- Read secrets to understand configuration (without displaying values)
- Deploy to Cloud Run staging with natural language

**Debugging:**
- Query Cloud Logging for recent errors
- Check Cloud Run service configuration
- Inspect Pub/Sub subscription backlog
- Review AlloyDB performance insights

```bash
# Install google-cloud-mcp
pip install google-cloud-mcp

# Configure ADC first
gcloud auth application-default login

# Test it works
python -m google_cloud_mcp --test
```

---

## Part 10: Infrastructure as Code — Terraform + Cloud Foundation Toolkit

### Why Terraform for GCP

Deployment Manager (GCP's native IaC) is legacy as of 2026. Use Terraform or Pulumi. Terraform is the industry standard for GCP infrastructure.

```bash
# Install Terraform
brew install terraform  # macOS
# or
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

### Provider Setup

```hcl
# versions.tf
terraform {
  required_version = ">= 1.7"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 6.0"
    }
  }

  backend "gcs" {
    bucket = "myapp-terraform-state"
    prefix = "terraform/state"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

### GCS Remote State Backend

```bash
# Create state bucket (do this once manually)
gcloud storage buckets create gs://myapp-terraform-state \
  --location us-central1 \
  --uniform-bucket-level-access

# Enable versioning on state bucket (for state recovery)
gcloud storage buckets update gs://myapp-terraform-state \
  --versioning
```

### Cloud Run with Terraform

```hcl
# cloud_run.tf
resource "google_cloud_run_v2_service" "backend" {
  name     = "myapp-backend"
  location = var.region
  project  = var.project_id

  ingress = "INGRESS_TRAFFIC_ALL"

  template {
    service_account = google_service_account.backend.email

    scaling {
      min_instance_count = 1
      max_instance_count = 50
    }

    containers {
      image = "us-central1-docker.pkg.dev/${var.project_id}/myapp/backend:${var.image_tag}"

      resources {
        limits = {
          cpu    = "2"
          memory = "2Gi"
        }
        cpu_idle          = false  # Always-on CPU
        startup_cpu_boost = true
      }

      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_password.secret_id
            version = "latest"
          }
        }
      }

      startup_probe {
        http_get {
          path = "/health"
          port = 8080
        }
        initial_delay_seconds = 5
        period_seconds        = 5
        failure_threshold     = 5
      }
    }

    vpc_access {
      connector = google_vpc_access_connector.connector.id
      egress    = "PRIVATE_RANGES_ONLY"
    }
  }

  traffic {
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
    percent = 100
  }
}

# Allow unauthenticated invocations (public API)
resource "google_cloud_run_v2_service_iam_member" "public_invoker" {
  project  = var.project_id
  location = var.region
  name     = google_cloud_run_v2_service.backend.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}
```

### IAM: member vs binding (Critical Difference)

```hcl
# CORRECT for shared projects — additive, never removes other bindings
resource "google_project_iam_member" "backend_secret_accessor" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_service_account.backend.email}"
}

# DANGEROUS for shared projects — REPLACES all existing members with this set
# Only safe when you own the entire role on the project
resource "google_project_iam_binding" "secret_accessor" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  members = [
    "serviceAccount:${google_service_account.backend.email}",
  ]
  # This will REMOVE any other service accounts that had this role!
}
```

**Rule:** Use `google_project_iam_member` in almost all cases. Use `google_project_iam_binding` only in new projects where you control all IAM bindings for that role.

### Cloud Foundation Toolkit Modules

Google's Cloud Foundation Toolkit (CFT) provides opinionated Terraform modules:

```hcl
# Use CFT modules for common patterns
module "project_factory" {
  source  = "terraform-google-modules/project-factory/google"
  version = "~> 15.0"

  name              = "myapp-prod"
  org_id            = var.org_id
  billing_account   = var.billing_account
  folder_id         = var.folder_id

  activate_apis = [
    "run.googleapis.com",
    "alloydb.googleapis.com",
    "secretmanager.googleapis.com",
    "pubsub.googleapis.com",
  ]
}

module "alloydb" {
  source  = "terraform-google-modules/alloydb/google"
  version = "~> 3.0"

  project_id    = var.project_id
  cluster_id    = "myapp-cluster"
  location      = var.region
  network_id    = module.vpc.network_id

  primary_instance = {
    instance_id   = "myapp-primary"
    machine_type  = "db-perf-optimized-N-2"
    database_flags = {
      "google_columnar_engine.enabled" = "on"
    }
  }
}
```

### Terragrunt for DRY Infrastructure

When managing multiple environments (prod, staging, dev), use Terragrunt to avoid duplicating Terraform code:

```
infrastructure/
  modules/
    cloud-run/
    alloydb/
    pubsub/
  live/
    prod/
      cloud-run/
        terragrunt.hcl  → uses ../../../modules/cloud-run
      alloydb/
        terragrunt.hcl  → uses ../../../modules/alloydb
    staging/
      cloud-run/
        terragrunt.hcl  → same module, different vars
```

```hcl
# live/prod/cloud-run/terragrunt.hcl
terraform {
  source = "../../../modules/cloud-run"
}

inputs = {
  project_id  = "myapp-prod"
  region      = "us-central1"
  min_instances = 1
  image_tag   = "latest"
}
```

---

## Part 11: Observability — Cloud Logging, Monitoring, Trace

### Cloud Logging — Structured Logging

Always use structured (JSON) logging. Cloud Logging parses JSON log entries and makes fields searchable in the console:

```python
import logging
import google.cloud.logging
from google.cloud.logging.handlers import CloudLoggingHandler

def setup_logging():
    """Configure structured Cloud Logging."""
    if os.environ.get("K_SERVICE"):
        # Running on Cloud Run — use Cloud Logging
        client = google.cloud.logging.Client()
        handler = CloudLoggingHandler(client)
        logging.getLogger().addHandler(handler)
        logging.getLogger().setLevel(logging.INFO)
    else:
        # Local development — use standard stderr
        logging.basicConfig(
            level=logging.DEBUG,
            format="%(asctime)s %(levelname)s %(name)s %(message)s",
        )

# In Cloud Run, print to stdout as JSON — automatically ingested by Cloud Logging
import json, sys

def log_structured(severity: str, message: str, **kwargs):
    """Write structured log entry to stdout for Cloud Logging ingestion."""
    entry = {
        "severity": severity,
        "message": message,
        **kwargs,
    }
    print(json.dumps(entry), file=sys.stdout, flush=True)

log_structured(
    "INFO",
    "Request processed",
    user_id="user-123",
    latency_ms=142,
    endpoint="/api/v1/documents",
)
```

**Log-based metrics:** Create metrics from log entries for alerting:

```bash
gcloud logging metrics create api-errors \
  --description "Count of API 5xx errors" \
  --log-filter 'resource.type="cloud_run_revision" AND httpRequest.status>=500'
```

**Log Router to BigQuery (long-term retention):**

```bash
gcloud logging sinks create bigquery-sink \
  bigquery.googleapis.com/projects/myapp-prod/datasets/logs \
  --log-filter 'resource.type="cloud_run_revision"' \
  --use-partitioned-tables
```

### Cloud Monitoring — Metrics, Alerting, SLOs

**Uptime check:**

```bash
gcloud monitoring uptime check-configs create myapp-backend \
  --display-name "MyApp Backend Health" \
  --monitored-resource-type uptime_url \
  --http-check-path /health \
  --hostname myapp-backend-xxx-uc.a.run.app \
  --period 60
```

**Native SLO management:**

```bash
# Create a request-based SLO for error rate
gcloud monitoring services slos create \
  --service myapp-backend-service \
  --slo-id availability-slo \
  --display-name "99.9% Availability" \
  --good-total-ratio-threshold 0.999 \
  --good-total-filter 'metric.labels.response_code_class!="5xx"' \
  --total-filter 'metric.type="run.googleapis.com/request_count"' \
  --rolling-period-days 28
```

**Python custom metrics:**

```python
from google.cloud import monitoring_v3
import time

def write_custom_metric(
    project_id: str,
    metric_name: str,
    value: float,
    labels: dict[str, str] | None = None,
) -> None:
    """Write a custom metric to Cloud Monitoring."""
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{project_id}"

    series = monitoring_v3.TimeSeries()
    series.metric.type = f"custom.googleapis.com/{metric_name}"
    if labels:
        series.metric.labels.update(labels)
    series.resource.type = "global"

    now = time.time()
    interval = monitoring_v3.TimeInterval(
        end_time={"seconds": int(now), "nanos": int((now - int(now)) * 10**9)}
    )
    point = monitoring_v3.Point(
        interval=interval,
        value={"double_value": value},
    )
    series.points = [point]

    client.create_time_series(name=project_name, time_series=[series])
```

### Cloud Trace — Distributed Tracing

```python
# Install
# pip install opentelemetry-sdk opentelemetry-exporter-gcp-trace google-cloud-trace

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor

def setup_tracing(project_id: str, sample_rate: float = 0.1):
    """Configure OpenTelemetry with Cloud Trace exporter."""
    provider = TracerProvider(
        sampler=TraceIdRatioBased(sample_rate),
    )
    provider.add_span_processor(
        BatchSpanProcessor(CloudTraceSpanExporter(project_id=project_id))
    )
    trace.set_tracer_provider(provider)

# Use in FastAPI middleware
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

app = FastAPI()
FastAPIInstrumentor.instrument_app(app)
setup_tracing("myapp-prod", sample_rate=0.1)
```

---

## Part 12: Architecture Patterns for FastAPI on GCP

### Pattern 1: Cloud Run + AlloyDB + Memorystore Valkey (Recommended)

The standard April 2026 pattern for most FastAPI backends:

```
Internet → Cloud Load Balancer (HTTPS) → Cloud Run (FastAPI)
                                              ├─ AlloyDB (via Auth Proxy) — primary DB + vectors
                                              ├─ Memorystore Valkey — session cache, rate limiting
                                              └─ Secret Manager — credentials

Cloud Run → Pub/Sub → Cloud Run Worker — async event processing
Cloud Run → GCS — file storage
Cloud Run → Vertex AI Gemini — LLM calls
```

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
import asyncpg
import redis.asyncio as redis
from google.cloud.alloydb.connector import AsyncConnector

db_pool: asyncpg.Pool | None = None
cache: redis.Redis | None = None
alloydb_connector: AsyncConnector | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global db_pool, cache, alloydb_connector

    # AlloyDB
    alloydb_connector = AsyncConnector()
    async def getconn():
        return await alloydb_connector.connect(
            os.environ["ALLOYDB_INSTANCE"],
            "asyncpg",
            user=os.environ["DB_USER"],
            password=os.environ["DB_PASSWORD"],
            db=os.environ["DB_NAME"],
        )
    db_pool = await asyncpg.create_pool(
        connect=getconn, min_size=5, max_size=20
    )

    # Valkey
    cache = redis.Redis.from_url(
        os.environ["VALKEY_URL"],
        max_connections=20,
        decode_responses=True,
    )

    yield  # Application runs here

    # Cleanup
    await db_pool.close()
    await cache.aclose()
    await alloydb_connector.close()

app = FastAPI(lifespan=lifespan)
```

### Pattern 3: AI-Native — Cloud Run + Vertex AI + AlloyDB AI + Pub/Sub

For RAG applications:

```
User Request → Cloud Run (FastAPI)
                    ├─ Embed query → Vertex AI text-embedding-004
                    ├─ Vector search → AlloyDB pgvector (SCANN index)
                    ├─ Retrieve context → AlloyDB
                    ├─ Generate response → Vertex AI Gemini 2.0 Flash
                    └─ Log trace → Pub/Sub → BigQuery (async)

Background:
Pub/Sub "document-ingested" → Cloud Run Worker
  ├─ Generate embedding → Vertex AI
  └─ Store in AlloyDB → pgvector column
```

```python
# app/rag.py
import vertexai
from vertexai.generative_models import GenerativeModel
from vertexai.language_models import TextEmbeddingModel
import asyncpg

vertexai.init(project=os.environ["GCP_PROJECT"], location="us-central1")

embed_model = TextEmbeddingModel.from_pretrained("text-embedding-004")
gen_model = GenerativeModel("gemini-2.0-flash-001")

async def rag_query(question: str, db: asyncpg.Pool) -> str:
    """RAG pipeline using AlloyDB pgvector + Gemini 2.0 Flash."""
    # 1. Embed the question
    embeddings = embed_model.get_embeddings([question])
    query_embedding = embeddings[0].values

    # 2. Vector search in AlloyDB
    rows = await db.fetch("""
        SELECT content, 1 - (embedding <=> $1::vector) AS similarity
        FROM documents
        WHERE 1 - (embedding <=> $1::vector) > 0.7
        ORDER BY embedding <=> $1::vector
        LIMIT 5
    """, query_embedding)

    if not rows:
        context = "No relevant documents found."
    else:
        context = "\n\n".join([row["content"] for row in rows])

    # 3. Generate response with Gemini 2.0 Flash
    prompt = f"""Answer the question based on the following context.

Context:
{context}

Question: {question}

Answer:"""

    response = await gen_model.generate_content_async(
        prompt,
        generation_config={"max_output_tokens": 1024, "temperature": 0.1},
    )
    return response.text
```

### Zero-Downtime Deployments with Traffic Splitting

```bash
# Deploy new version without sending traffic
gcloud run deploy myapp-backend \
  --image NEW_IMAGE \
  --no-traffic \
  --tag canary

# Split traffic: 10% to new version
gcloud run services update-traffic myapp-backend \
  --to-tags canary=10

# After validation, send all traffic
gcloud run services update-traffic myapp-backend \
  --to-latest

# Or rollback
gcloud run services update-traffic myapp-backend \
  --to-revisions PREVIOUS_REVISION=100
```

### Connection Pooling Best Practices

```python
# app/deps.py
from typing import AsyncGenerator
import asyncpg
from fastapi import Depends

async def get_db() -> AsyncGenerator[asyncpg.Connection, None]:
    """FastAPI dependency for database connections."""
    async with db_pool.acquire() as conn:
        yield conn

@app.get("/users/{user_id}")
async def get_user(user_id: str, db: asyncpg.Connection = Depends(get_db)):
    row = await db.fetchrow(
        "SELECT * FROM users WHERE id = $1 AND deleted_at IS NULL",
        user_id
    )
    if not row:
        raise HTTPException(404)
    return dict(row)
```

Pool sizing formula for Cloud Run:
```
pool_max_size = (alloydb_max_connections * 0.8) / num_cloud_run_instances
```

For AlloyDB with max 1000 connections and 10 Cloud Run instances:
`pool_max_size = (1000 * 0.8) / 10 = 80`

### Cost Optimization

**Committed Use Discounts (CUDs):** For AlloyDB and Memorystore instances you'll run continuously, commit to 1-year or 3-year use for 30-70% discount.

**Cloud Run min-instances vs cold starts:** Profile your cold start time. If it's under 1 second (small Python app), `min-instances 0` and accept occasional cold starts. If it's over 2 seconds (ML model loading), set `min-instances 1`.

**Request-based vs always-on CPU:** Always-on CPU costs roughly 2x more. Only enable if you have genuine background work (WebSockets, scheduled tasks within instance).

**Gemini cost optimization:** Gemini 2.0 Flash is ~10x cheaper than Gemini 2.5 Pro. Profile your tasks — most classification, summarization, and simple generation tasks work fine on Flash.

---

## Quick Reference Card

```bash
# Most-used commands in daily GCP work

# Deploy Cloud Run service
gcloud run deploy SERVICE --image IMAGE --region us-central1

# View logs
gcloud run services logs read SERVICE --region us-central1 --limit 50

# Access a secret
gcloud secrets versions access latest --secret SECRET_NAME

# List active Cloud Run services
gcloud run services list --region us-central1

# BigQuery quick query
bq query --use_legacy_sql=false 'SELECT COUNT(*) FROM myapp_prod.users'

# Pub/Sub: publish a test message
gcloud pubsub topics publish TOPIC --message '{"test": true}'

# Set up ADC for local dev
gcloud auth application-default login

# Switch project config
gcloud config configurations activate PROFILE_NAME

# Impersonate a service account
gcloud auth application-default login \
  --impersonate-service-account SA@PROJECT.iam.gserviceaccount.com
```

---

*Generated: April 2026. Patterns reflect current GCP best practices as of this date.*
*See companion: `~/.claude/rules/gcp-patterns.md` for enforcement rules.*
