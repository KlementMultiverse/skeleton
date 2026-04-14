# AWS Complete — Study Notes (April 2026)

> Stack: Python 3.12, boto3, AWS CDK v2, FastAPI. Last verified: April 2026.
> AI connection: AWS is the infrastructure layer under every backend you build. Bedrock is the
> managed LLM runtime. Everything else — compute, storage, database, messaging — supports it.

---

## The One Thing to Remember

AWS is a menu, not a prescription. The skill is **picking the right service for the context**,
not memorizing every feature. Every section ends with a decision table — use that when unsure.

---

## Part 1: AWS Mental Model — The Building Blocks

### The City Analogy

Think of AWS as a city you build from parts:

- **Region** = which city (us-east-1 = Northern Virginia, eu-west-1 = Dublin)
- **Availability Zone (AZ)** = which district in that city (physically separated data centers, km apart)
- **VPC** = your private walled compound inside the city
- **Subnets** = rooms inside your compound (public = windows to the street, private = interior)
- **EC2/Lambda/Fargate** = the workers who live in the compound
- **S3/RDS/DynamoDB** = the warehouse, the filing room, the spreadsheet cabinet
- **IAM** = the keycard system — who gets in, what they're allowed to touch

The mental model: **you never own infrastructure, you rent capability.** AWS manages the hardware; you manage what runs on it.

---

### Services Taxonomy

| Category | Services | When You Need It |
|---|---|---|
| Compute | Lambda, App Runner, ECS Fargate, EKS | Running your application code |
| Storage | S3, EBS, EFS | Files, disks, shared file systems |
| Database | Aurora, DynamoDB, ElastiCache Valkey | Relational, key-value, cache |
| Messaging | SQS, SNS, EventBridge, Kinesis | Decoupling services, events, streams |
| AI/ML | Bedrock, SageMaker | LLM APIs, model training |
| Security | IAM, Secrets Manager, KMS, GuardDuty | Auth, secrets, encryption, threat detection |
| Networking | VPC, Route 53, CloudFront, ALB | DNS, CDN, load balancing, isolation |
| Observability | CloudWatch, X-Ray, ADOT | Logs, metrics, traces |
| IaC | CDK v2, CloudFormation | Defining infrastructure as code |

---

### Shared Responsibility Model

**Analogy:** You rent a furnished apartment. The landlord maintains the building structure, plumbing, and electricity infrastructure. You are responsible for locking your door, not causing damage, and managing what's inside.


AWS is responsible for:              You are responsible for:
- Physical data centers              - OS configuration (EC2)
- Hardware failure                   - Application security
- Hypervisor security               - IAM policies and least privilege
- Network infrastructure            - Data encryption at rest/in transit
- Managed service patching          - Secrets management
  (RDS engine, Lambda runtime)       - VPC and security group rules


**Key insight for FastAPI devs:** With Lambda/Fargate/App Runner, AWS patches the OS and container runtime. You only patch your application's Python dependencies and container image. With EC2, you patch everything. This is the main reason to prefer managed services.

---

### Region vs AZ vs VPC Design Mental Model


us-east-1 (Region)
├── us-east-1a (AZ)
│   ├── Public subnet (10.0.1.0/24)  — ALB, NAT Gateway
│   └── Private subnet (10.0.11.0/24) — App, DB
├── us-east-1b (AZ)
│   ├── Public subnet (10.0.2.0/24)  — ALB
│   └── Private subnet (10.0.12.0/24) — App, DB replica
└── us-east-1c (AZ) — optional third AZ for HA


**Rules:**
- Production: minimum 2 AZs for everything (ALB, RDS, ECS service)
- Isolated subnets (no route to internet, not even NAT): for databases
- Private subnets: for app servers (outbound via NAT Gateway only)
- Public subnets: only for things needing direct internet access (load balancers)

**April 2026 default VPC layout for FastAPI on ECS/App Runner:**

VPC: 10.0.0.0/16
├── Public subnets (2 AZs): ALB, NAT Gateways
├── Private subnets (2 AZs): ECS tasks, App Runner VPC connector
└── Isolated subnets (2 AZs): Aurora cluster, ElastiCache Valkey


---

### IAM Identity Center vs IAM Users (April 2026 Standard)

**Analogy:** IAM users are permanent magnetic keycards handed to employees — they never expire, and if lost you must hunt them down manually. IAM Identity Center is a corporate badge system linked to your HR directory — access disappears when someone leaves, and time-limited session tokens replace permanent keys.

**April 2026: IAM Identity Center is the standard for human access. IAM users are legacy.**

| Scenario | Use |
|---|---|
| Developer accessing AWS console or CLI | IAM Identity Center (SSO) |
| CI/CD system (GitHub Actions) | OIDC role assumption — no permanent keys |
| Service-to-service (Lambda calling S3) | IAM role attached to the service |
| Legacy system that can't use SSO | Last resort: IAM user with MFA enforced |

Never create IAM users for humans in a new setup. Connect IAM Identity Center to your IdP (Okta, Google Workspace, Entra ID) or use the AWS Organizations built-in directory.

---

## Part 2: CLI and Developer Tools (April 2026)

### AWS CLI v2

**Analogy:** The CLI is your remote control for the entire AWS data center. Instead of clicking through the console, you express the operation in a single command.

**Installation:**
bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Verify
aws --version   # aws-cli/2.x.x Python/3.x.x


**SSO Configuration (April 2026 standard — not `aws configure`):**
bash
# Initial setup — interactive
aws configure sso
# Prompts: SSO session name, SSO start URL, SSO region, account ID, role, profile name

# Login (opens browser)
aws sso login --profile dev

# Set default for shell session
export AWS_PROFILE=dev
aws s3 ls   # uses dev profile


**Named profile in `~/.aws/config`:**
ini
[profile dev]
sso_session = my-company
sso_account_id = 123456789012
sso_role_name = DeveloperAccess
region = us-east-1
output = json

[sso-session my-company]
sso_start_url = https://my-company.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access


**Output formats:**
bash
--output json    # programmatic processing (default)
--output yaml    # human-readable, copy to CDK
--output table   # quick visual scan in terminal
--output text    # piping to awk/cut/grep


**Identity check — always run first when debugging permissions:**
bash
aws sts get-caller-identity
# {
#   "Account": "123456789012",
#   "Arn": "arn:aws:sts::123456789012:assumed-role/DeveloperAccess/klement"
# }


**JMESPath `--query` — the most useful CLI skill:**
bash
# List running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].[InstanceId,InstanceType,Tags[?Key=='Name'].Value|[0]]" \
  --output table

# Get a specific SSM parameter value
aws ssm get-parameter --name /myapp/db-url --query "Parameter.Value" --output text

# List RDS clusters and their status
aws rds describe-db-clusters \
  --query "DBClusters[*].[DBClusterIdentifier,Status,EngineVersion]" \
  --output table

# ECR login — pipe password directly to docker
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com


**Tailing Lambda/ECS logs live:**
bash
aws logs tail /aws/lambda/my-function --follow
aws logs tail /aws/lambda/my-function --follow --format short
aws logs tail /ecs/my-service --follow --filter-pattern "ERROR"


**Useful patterns for AI/backend devs:**
bash
# Invoke Lambda and see output immediately
aws lambda invoke \
  --function-name my-function \
  --payload '{"key": "value"}' \
  --cli-binary-format raw-in-base64-out \
  output.json && cat output.json

# Force ECS service redeployment (pull new container)
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --force-new-deployment

# Get a secret value (for debugging — don't use in scripts)
aws secretsmanager get-secret-value \
  --secret-id /myapp/prod/db-password \
  --query SecretString --output text

# List ECR images sorted by push time (last 5)
aws ecr describe-images \
  --repository-name my-app \
  --query 'sort_by(imageDetails, &imagePushedAt)[-5:].[imageTags[0],imagePushedAt]' \
  --output table

# Check SQS queue depth
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --attribute-names ApproximateNumberOfMessages

# CloudFormation stack outputs (after CDK deploy)
aws cloudformation describe-stacks \
  --stack-name MyStack \
  --query "Stacks[0].Outputs"


---

### AWS CDK v2 CLI

**Analogy:** CDK is infrastructure as Python. You write Python classes. CDK compiles them to CloudFormation JSON. CloudFormation deploys the real resources. The CDK CLI is how you drive that pipeline.

**One-time setup per account/region:**
bash
# Install CDK CLI (Node.js tool)
npm install -g aws-cdk

# Install Python CDK libraries
pip install aws-cdk-lib constructs

# Bootstrap — creates CDK toolkit stack (run once per account/region)
cdk bootstrap aws://123456789012/us-east-1


**Project initialization:**
bash
mkdir my-infra && cd my-infra
cdk init app --language python
source .venv/bin/activate
pip install -r requirements.txt


**Core workflow:**
bash
cdk synth           # Compile CDK → CloudFormation (no deploy — use for validation)
cdk diff            # Show what WILL change vs current deployed state
cdk deploy          # Deploy all stacks
cdk deploy MyStack  # Deploy specific stack
cdk destroy         # Tear down (asks confirmation)
cdk ls              # List all stacks


**Development workflow:**
bash
cdk watch           # Hot reload — watches file changes, auto-deploys (dev only)
cdk deploy --context environment=staging   # pass context values
cdk deploy --require-approval never        # auto-approve IAM changes (careful in prod)
cdk synth --no-staging > template.yaml     # inspect generated CloudFormation


**CDK Pipelines — self-mutating CI/CD:**

"Self-mutating" means: when you modify the pipeline definition in CDK code, the pipeline updates itself on the next run without you manually updating it. It's a pipeline that can redeploy itself.

python
# stacks/pipeline_stack.py
import aws_cdk as cdk
from aws_cdk import pipelines
from constructs import Construct

class PipelineStack(cdk.Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        pipeline = pipelines.CodePipeline(self, "Pipeline",
            pipeline_name="MyAppPipeline",
            synth=pipelines.ShellStep("Synth",
                input=pipelines.CodePipelineSource.git_hub(
                    "KlementMultiverse/my-repo", "main",
                    authentication=cdk.SecretValue.secrets_manager("github-token"),
                ),
                commands=[
                    "npm install -g aws-cdk",
                    "pip install -r requirements.txt",
                    "cdk synth",
                ],
            ),
        )

        # Staging stage
        staging = pipeline.add_stage(
            MyAppStage(self, "Staging",
                env=cdk.Environment(account="123456789012", region="us-east-1")
            )
        )
        # Run integration tests after staging deploy
        staging.add_post(pipelines.ShellStep("IntegrationTest",
            commands=["pytest tests/integration/ -v"],
            env_from_cfn_outputs={"API_URL": staging.app_url},
        ))

        # Production with manual approval gate
        pipeline.add_stage(
            MyAppStage(self, "Production",
                env=cdk.Environment(account="987654321098", region="us-east-1")
            ),
            pre=[pipelines.ManualApprovalStep("ApproveProduction")],
        )


---

### AWS SAM CLI (Lambda development)

**Analogy:** SAM is a local Lambda simulator. Test your Lambda function on your laptop before deploying to AWS.

bash
pip install aws-sam-cli

sam init                              # new Lambda project
sam local invoke MyFunction --event events/test.json   # run locally
sam local start-api                   # local API Gateway + Lambda on :3000
sam build                             # build deployment package
sam deploy --guided                   # interactive first deploy
sam logs -n MyFunction --tail         # tail deployed Lambda logs


---

## Part 3: Compute — Selection Guide

### The Decision Tree


Is the task event-driven and < 15 minutes?
  └── Lambda
       ├── Simple HTTP? → Function URL (skip API Gateway)
       └── Needs RDS? → ALWAYS add RDS Proxy between Lambda and RDS

Long-running HTTP server (FastAPI/Django)?
  ├── Simplest path to production? → App Runner
  └── Need ECS Exec, blue/green, sidecars? → ECS Fargate

Need Kubernetes CRDs, operators, Helm charts?
  └── EKS — high operational cost, only when genuinely needed


---

### Lambda — Event-Driven Compute

**Analogy:** A vending machine. You insert a coin (an event), you get a result. It doesn't run all day — it activates only when you interact. Lambda works the same: no events = zero cost, zero infrastructure to manage.

**When Lambda is right:**
- S3 upload processing, SQS consumers, DynamoDB Streams
- Scheduled jobs (EventBridge Scheduler)
- Webhooks, lightweight HTTP endpoints
- ETL with unpredictable bursty traffic

**When Lambda is wrong:**
- Processes > 15 minutes (hard limit)
- Persistent WebSocket servers
- High-concurrency where cold starts degrade UX
- Anything needing local disk > 10 GB

**Function URLs — direct HTTPS without API Gateway:**
python
# CDK v2: give a Lambda a public HTTPS endpoint
from aws_cdk import aws_lambda as lambda_

fn = lambda_.Function(self, "MyFn",
    runtime=lambda_.Runtime.PYTHON_3_12,
    handler="index.handler",
    code=lambda_.Code.from_asset("lambda"),
    architecture=lambda_.Architecture.ARM_64,  # Graviton2 — 20% cheaper
)

fn_url = fn.add_function_url(
    auth_type=lambda_.FunctionUrlAuthType.NONE,  # or AWS_IAM for private
    cors=lambda_.FunctionUrlCorsOptions(
        allowed_origins=["https://myapp.com"],
        allowed_methods=[lambda_.HttpMethod.POST],
    ),
)


**Lambda + RDS Proxy — the mandatory pattern:**

Lambda scales to thousands of concurrent invocations, each opening a new DB connection. PostgreSQL has a finite connection limit (~500 for small instances). RDS Proxy is a managed connection pooler sitting between Lambda and RDS.


Lambda invocations (0 → 1000 concurrent)
    ↓
RDS Proxy (pool of 20–50 connections, persistent)
    ↓
Aurora PostgreSQL (sees only 20–50 connections regardless of Lambda scale)


**Lambda SnapStart (April 2026 — Python 3.12+):**

SnapStart snapshots the fully-initialized Lambda execution environment. On cold start, it restores from the snapshot instead of re-importing packages from scratch. For Python 3.12+, this dramatically cuts initialization time caused by slow package imports.

python
# CDK: enable SnapStart (requires published versions)
fn = lambda_.Function(self, "MyFn",
    runtime=lambda_.Runtime.PYTHON_3_12,
    snap_start=lambda_.SnapStartConf.ON_PUBLISHED_VERSIONS,
)
version = fn.current_version  # SnapStart operates on versions


**SnapStart constraints (April 2026):**
- Python 3.12+ only (not Node.js, not Ruby, not container images)
- Incompatible with: ephemeral storage > 512 MB, EFS, provisioned concurrency
- Requires publishing a function version
- Best for: functions with heavy import overhead (ML libraries, large SDKs)

**Graviton2 ARM64 for Lambda — free 20% savings:**
python
# Always use ARM_64 for new Python Lambda functions
fn = lambda_.Function(self, "MyFn",
    architecture=lambda_.Architecture.ARM_64,  # 20% cheaper per GB-second
    runtime=lambda_.Runtime.PYTHON_3_12,
    ...
)


**Cold start mitigation hierarchy (cheapest → most expensive):**
1. ARM64 (slight benefit, free)
2. SnapStart for Python 3.12+ (eliminates import-time cold start)
3. Keep packages small — fewer imports = faster init
4. Provisioned Concurrency (guaranteed warm instances — ongoing cost)

**Lambda Powertools — structured observability:**
python
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger()
tracer = Tracer()
metrics = Metrics(namespace="MyApp", service="api")

@logger.inject_lambda_context(log_event=True)
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def handler(event: dict, context: LambdaContext) -> dict:
    logger.info("Processing request", user_id=event.get("user_id"))
    metrics.add_metric(name="ProcessedItems", unit=MetricUnit.Count, value=1)
    return {"statusCode": 200, "body": "OK"}


---

### App Runner — Simplest Production Path

**Analogy:** Heroku for AWS. Push a Docker image (or GitHub repo), App Runner handles servers, load balancing, TLS, auto-scaling, and health checks. No Kubernetes. No ALB config. No task definitions. Just: here's my container, please run it.

**When App Runner is right:**
- First production deployment
- Small-to-medium FastAPI/Django apps
- Team without container orchestration experience
- Moving fast without operational overhead

**When to graduate to ECS Fargate:**
- Need ECS Exec for interactive debugging
- Need blue/green deployments with CodeDeploy
- Need sidecar containers (log shippers, proxies)
- Need fine-grained scaling policies

**Key concepts:**

Service
├── Source: ECR image or GitHub repo (auto-build on push)
├── Instance: vCPU + memory (0.25–4 vCPU, 0.5–12 GB)
├── Auto-scaling: min/max instances, concurrency trigger
├── Health check: HTTP path + interval + thresholds
└── VPC Connector: access to private VPC resources (RDS, Valkey)


**CDK: App Runner service from ECR:**
python
from aws_cdk import aws_apprunner_alpha as apprunner
from aws_cdk import aws_ecr as ecr

repo = ecr.Repository.from_repository_name(self, "Repo", "my-fastapi-app")

service = apprunner.Service(self, "Service",
    source=apprunner.Source.from_ecr(
        repository=repo,
        tag="latest",
        image_configuration=apprunner.ImageConfiguration(
            port=8000,
            environment_variables={"ENVIRONMENT": "production"},
            environment_secrets={
                "DATABASE_URL": apprunner.Secret.from_secrets_manager(db_secret, "url"),
                "CACHE_URL": apprunner.Secret.from_secrets_manager(cache_secret, "url"),
            },
        ),
    ),
    cpu=apprunner.Cpu.ONE_VCPU,
    memory=apprunner.Memory.TWO_GB,
    auto_scaling_configuration=apprunner.AutoScalingConfiguration(
        self, "Scaling",
        min_size=1,
        max_size=10,
        max_concurrency=100,
    ),
    vpc_connector=vpc_connector,
)


**VPC Connector — mandatory for private database access:**
python
vpc_connector = apprunner.VpcConnector(self, "VpcConnector",
    vpc=vpc,
    vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS),
    security_groups=[app_sg],
)


---

### ECS Fargate — Standard Production Container Platform

**Analogy:** App Runner gives you a managed apartment. ECS Fargate gives you a managed building — you design each floor (task definition), decide how many to run (desired count), and AWS manages the physical structure. More control without managing individual servers.

**Key concepts:**

Cluster = logical grouping of services
└── Service = "always keep N copies of this task running"
    └── Task = one running instance (like a pod in Kubernetes)
        └── Container(s) = Docker container(s) inside the task
Task Definition = the blueprint (image, CPU, memory, env, ports)


**Graviton ARM64 on ECS Fargate — April 2026 default:**

For ECS Fargate, ARM64 tasks cost approximately 20% less than equivalent x86 tasks with comparable or better performance for Python workloads. Always use ARM64 for new Fargate services.

python
from aws_cdk import aws_ecs as ecs, aws_ec2 as ec2, Duration

cluster = ecs.Cluster(self, "Cluster", vpc=vpc)

task_def = ecs.FargateTaskDefinition(self, "TaskDef",
    cpu=512,
    memory_limit_mib=1024,
    runtime_platform=ecs.RuntimePlatform(
        cpu_architecture=ecs.CpuArchitecture.ARM64,  # ~20% cheaper
        operating_system_family=ecs.OperatingSystemFamily.LINUX,
    ),
)

container = task_def.add_container("App",
    image=ecs.ContainerImage.from_ecr_repository(repo, tag="latest"),
    port_mappings=[ecs.PortMapping(container_port=8000)],
    logging=ecs.LogDrivers.aws_logs(
        stream_prefix="app",
        log_group=log_group,
    ),
    environment={"ENVIRONMENT": "production"},
    secrets={
        "DATABASE_URL": ecs.Secret.from_secrets_manager(db_secret, "url"),
    },
    health_check=ecs.HealthCheck(
        command=["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
        interval=Duration.seconds(30),
        timeout=Duration.seconds(5),
    ),
)

service = ecs.FargateService(self, "Service",
    cluster=cluster,
    task_definition=task_def,
    desired_count=2,
    vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS),
    security_groups=[app_sg],
    enable_execute_command=True,  # enables ECS Exec for debugging
    circuit_breaker=ecs.DeploymentCircuitBreaker(
        enable=True,
        rollback=True,  # auto-rollback if health checks fail
    ),
)


**Service Connect — modern service-to-service networking (replaces App Mesh):**
python
service = ecs.FargateService(self, "ApiService",
    ...
    service_connect_configuration=ecs.ServiceConnectProps(
        namespace="myapp.local",
        services=[
            ecs.ServiceConnectService(
                port_mapping_name="http",
                port=8000,
                dns_name="api",  # other services reach this as api:8000
            )
        ],
    ),
)


**ECS Exec — drop into a running container (no bastion host needed):**
bash
aws ecs execute-command \
  --cluster my-cluster \
  --task arn:aws:ecs:us-east-1:123456789012:task/my-cluster/abc123 \
  --container App \
  --interactive \
  --command "/bin/sh"

# Prerequisites: enable_execute_command=True in CDK
# Task role needs: ssmmessages:CreateControlChannel, ssmmessages:OpenControlChannel, etc.


**Auto-scaling:**
python
scaling = service.auto_scale_task_count(min_capacity=2, max_capacity=20)

scaling.scale_on_cpu_utilization("CpuScaling",
    target_utilization_percent=70,
    scale_in_cooldown=Duration.seconds(60),
    scale_out_cooldown=Duration.seconds(30),
)


---

### When NOT to Use EKS

Use EKS only when you genuinely need:
- Custom resource definitions (CRDs) and operators
- Advanced scheduling (GPU topology, node affinity rules)
- Existing Kubernetes Helm charts you cannot migrate
- Service mesh features beyond Service Connect

For a new FastAPI backend: ECS Fargate covers 90% of needs with far less operational overhead. EKS adds significant complexity in cluster upgrades, node group management, and networking troubleshooting.

---

## Part 4: Storage — S3 and Beyond

### S3 Mental Model

**Analogy:** A global filing cabinet with infinite drawers. Each drawer is a bucket (region-specific). Each file is an object (key + value + metadata). You never run out of space. Objects are immutable — replace, not edit in place.

**Storage classes and lifecycle:**

| Class | Use Case | Retrieval | Cost vs Standard |
|---|---|---|---|
| Standard | Hot data (< 30 days) | Instant | Baseline |
| Standard-IA | Warm data (30–90 days) | Instant | ~40% less |
| Glacier Instant | Archives accessed monthly | Instant | ~68% less |
| Glacier Flexible | Archives accessed yearly | 1–12 hours | ~80% less |
| Glacier Deep Archive | Compliance archives | 12–48 hours | ~90% less |
| Intelligent-Tiering | Unknown access pattern | Varies | Small monitoring fee |

**CDK lifecycle rules:**
python
from aws_cdk import aws_s3 as s3, Duration

bucket = s3.Bucket(self, "DataBucket",
    versioned=True,
    lifecycle_rules=[
        s3.LifecycleRule(
            transitions=[
                s3.Transition(
                    storage_class=s3.StorageClass.INFREQUENT_ACCESS,
                    transition_after=Duration.days(30),
                ),
                s3.Transition(
                    storage_class=s3.StorageClass.GLACIER_INSTANT_RETRIEVAL,
                    transition_after=Duration.days(90),
                ),
            ],
            expiration=Duration.days(365),
        )
    ],
)


**Presigned URLs — the right way to handle file uploads/downloads:**

Never expose S3 credentials to clients. Never proxy file data through your app server. Presigned URLs: the client gets a time-limited signed URL and uploads/downloads directly to/from S3.

python
import boto3

s3 = boto3.client("s3", region_name="us-east-1")

# PUT presigned URL for direct upload from browser
upload_url = s3.generate_presigned_url(
    "put_object",
    Params={
        "Bucket": "my-bucket",
        "Key": f"uploads/{user_id}/{filename}",
        "ContentType": "image/jpeg",
    },
    ExpiresIn=300,  # 5 minutes — keep short
)

# GET presigned URL for download
download_url = s3.generate_presigned_url(
    "get_object",
    Params={
        "Bucket": "my-bucket",
        "Key": f"uploads/{user_id}/{filename}",
        "ResponseContentDisposition": f"attachment; filename={filename}",
    },
    ExpiresIn=3600,
)


**FastAPI endpoint pattern:**
python
from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel
from uuid import uuid4

router = APIRouter()

class UploadRequest(BaseModel):
    filename: str
    content_type: str
    file_size: int

class UploadResponse(BaseModel):
    upload_url: str
    key: str

@router.post("/files/upload-url", response_model=UploadResponse)
async def get_upload_url(
    body: UploadRequest,
    current_user: User = Depends(get_current_user),
) -> UploadResponse:
    if body.file_size > 50 * 1024 * 1024:
        raise HTTPException(400, "File exceeds 50 MB limit")
    if body.content_type not in {"image/jpeg", "image/png", "application/pdf"}:
        raise HTTPException(400, "Unsupported content type")

    key = f"users/{current_user.id}/{uuid4()}/{body.filename}"
    url = s3_client.generate_presigned_url(
        "put_object",
        Params={"Bucket": BUCKET, "Key": key, "ContentType": body.content_type},
        ExpiresIn=300,
    )
    return UploadResponse(upload_url=url, key=key)


**S3 Event Notifications → Lambda:**
python
from aws_cdk import aws_s3_notifications as s3n

bucket.add_event_notification(
    s3.EventType.OBJECT_CREATED,
    s3n.LambdaDestination(process_fn),
    s3.NotificationKeyFilter(prefix="uploads/", suffix=".pdf"),
)


**EBS vs EFS vs S3 decision:**

| | EBS | EFS | S3 |
|---|---|---|---|
| Type | Block (disk) | Shared file system | Object storage |
| Access | Single EC2 instance | Multiple EC2/Fargate | HTTP API / presigned URLs |
| Use case | Database volumes, OS | Shared ML weights, shared uploads | Everything else |
| Fargate | Not usable | Mountable | Via boto3 SDK |

For FastAPI on Fargate: use S3 for files. Use EFS only when multiple tasks need a shared writable file system (e.g., shared model weights directory).

---

## Part 5: Database Selection Guide (April 2026)

### The Decision Framework


Relational data, complex queries, joins?
  └── Aurora PostgreSQL (serverless v2 for scale-to-zero)
      └── Need vector similarity for RAG? → add pgvector

Pure key-value, known access patterns, global scale?
  └── DynamoDB — read the "when NOT to use" section first

Cache, sessions, pub/sub, rate limiting?
  └── ElastiCache Valkey (April 2026 default)

Full-text search, log analytics at scale?
  └── OpenSearch (or start with pg_trgm if Aurora can handle it)


---

### Aurora PostgreSQL — Production Default

**Analogy:** Aurora is PostgreSQL with a racing car engine. Same SQL you already know, same psycopg2/asyncpg drivers, but storage grows automatically, scales across 6 replicas under the hood, and Serverless v2 scales down to near-zero when idle.

**Aurora Serverless v2 — April 2026 default for new projects:**

Serverless v2 scales in fine-grained 0.5 ACU increments (1 ACU ≈ 2 GB RAM). Scaling happens in seconds without connection drops. Aurora Serverless v1 was deprecated in 2025 — v2 is the only option.

python
from aws_cdk import aws_rds as rds, aws_ec2 as ec2, RemovalPolicy

cluster = rds.DatabaseCluster(self, "Database",
    engine=rds.DatabaseClusterEngine.aurora_postgres(
        version=rds.AuroraPostgresEngineVersion.VER_16_4
    ),
    writer=rds.ClusterInstance.serverless_v2("writer"),
    readers=[
        rds.ClusterInstance.serverless_v2("reader", scale_with_writer=True),
    ],
    serverless_v2_min_capacity=0.5,   # minimum ACU (never fully pauses)
    serverless_v2_max_capacity=16,    # scale ceiling
    vpc=vpc,
    vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_ISOLATED),
    security_groups=[db_sg],
    credentials=rds.Credentials.from_generated_secret("postgres"),
    removal_policy=RemovalPolicy.SNAPSHOT,
)


**RDS Proxy — mandatory for Lambda, recommended for App Runner:**
python
proxy = rds.DatabaseProxy(self, "Proxy",
    proxy_target=rds.ProxyTarget.from_cluster(cluster),
    secrets=[cluster.secret],
    vpc=vpc,
    vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS),
    security_groups=[proxy_sg],
    iam_auth=True,           # IAM-based auth for Lambda (no password needed)
    require_tls=True,
    idle_client_timeout=Duration.minutes(15),
    max_connections_percent=80,
)


**pgvector on Aurora PostgreSQL — for RAG workloads:**
sql
-- Enable pgvector (Aurora PostgreSQL 15.2+ supports it natively)
CREATE EXTENSION IF NOT EXISTS vector;

-- Document chunks table
CREATE TABLE documents (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content    TEXT NOT NULL,
    metadata   JSONB DEFAULT '{}',
    embedding  vector(1536),  -- 1536 for Titan V2, 1024 for Cohere Embed V3
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW index for approximate nearest neighbor (faster query time than IVFFlat)
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);


**Aurora Global Database — multi-region:**

Primary: us-east-1 (read + write)
  ↓ < 1 second replication lag
Secondary: eu-west-1 (read only)
Secondary: ap-southeast-1 (read only)

Failover: promote secondary to primary in ~1 minute
RPO: near-zero. RTO: ~1 minute.


---

### DynamoDB — When to Use (and When NOT To)

**Analogy:** DynamoDB is a massive indexed filing cabinet. You can find any file instantly if you know the exact drawer label. But you cannot ask "give me all files from 2024 where the amount is between $100 and $500" — that requires scanning every drawer. Perfect for "get this exact thing," terrible for "find things matching these criteria."

**Use DynamoDB when:**
- Access patterns are known upfront and simple (get by ID, list by user ID)
- Scale is truly massive (millions of rows, thousands of TPS)
- Multi-region active-active writes (Global Tables)
- TTL-based expiry is needed (sessions, rate limit counters, temp tokens)

**Do NOT use DynamoDB when:**
- You need complex queries (joins, aggregations, GROUP BY, HAVING)
- Access patterns are evolving — you'll need a new GSI every sprint
- The team is comfortable with SQL (use Aurora, easier to change later)
- You're building an MVP (Aurora Serverless v2 is just as cheap at small scale)

**Single-table design — the key DynamoDB concept:**

One table stores ALL entity types.
PK (partition key) = entity type + ID
SK (sort key) = relationship + secondary identifier

Example:
PK: USER#user123   SK: PROFILE        → user profile record
PK: USER#user123   SK: ORDER#2024-001 → user's order
PK: USER#user123   SK: ORDER#2024-002 → user's order
PK: ORDER#2024-001 SK: ITEM#1         → order line item

Query "all orders for user123":
  → Query(PK="USER#user123", SK begins_with "ORDER#")


**CDK: DynamoDB table with GSI:**
python
from aws_cdk import aws_dynamodb as dynamodb

table = dynamodb.Table(self, "Table",
    partition_key=dynamodb.Attribute(name="PK", type=dynamodb.AttributeType.STRING),
    sort_key=dynamodb.Attribute(name="SK", type=dynamodb.AttributeType.STRING),
    billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,  # on-demand, no capacity planning
    removal_policy=RemovalPolicy.RETAIN,
    point_in_time_recovery=True,
    table_class=dynamodb.TableClass.STANDARD,
    stream=dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
)

table.add_global_secondary_index(
    index_name="GSI1",
    partition_key=dynamodb.Attribute(name="GSI1PK", type=dynamodb.AttributeType.STRING),
    sort_key=dynamodb.Attribute(name="GSI1SK", type=dynamodb.AttributeType.STRING),
)


**boto3 DynamoDB patterns:**
python
import boto3, time
from boto3.dynamodb.conditions import Key, Attr

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("MyTable")

# Get single item
response = table.get_item(Key={"PK": "USER#123", "SK": "PROFILE"})
user = response.get("Item")

# Query (efficient — uses index, not scan)
response = table.query(
    KeyConditionExpression=Key("PK").eq("USER#123") & Key("SK").begins_with("ORDER#"),
    Limit=20,
    ScanIndexForward=False,  # descending (newest first)
)
orders = response["Items"]

# Conditional put (prevent overwrites)
table.put_item(
    Item={"PK": "USER#123", "SK": "PROFILE", "email": "user@example.com"},
    ConditionExpression=Attr("PK").not_exists(),
)

# Atomic counter increment
table.update_item(
    Key={"PK": "USER#123", "SK": "COUNTERS"},
    UpdateExpression="ADD request_count :inc",
    ExpressionAttributeValues={":inc": 1},
)

# TTL — automatic deletion at epoch timestamp
table.put_item(Item={
    "PK": "SESSION#abc",
    "SK": "DATA",
    "data": "...",
    "ttl": int(time.time()) + 3600,  # auto-deleted after 1 hour
})


---

### ElastiCache Valkey — April 2026 Standard

**Analogy:** Valkey is the refrigerator next to the kitchen. The database (Aurora) is the warehouse in the basement — comprehensive but slow to reach. Valkey keeps what you need right now at arm's reach. A query that takes 10ms from Aurora takes < 1ms from Valkey.

**Valkey vs Redis OSS — April 2026 reality:**

In March 2024, Redis Ltd changed the license from BSD-3-Clause to a dual proprietary/SSPL model. The open-source community forked Redis 7.2.4 as Valkey under the Apache 2.0 license. AWS made Valkey the **default engine for new ElastiCache and MemoryDB instances**. Google Memorystore and Akamai followed.


Wire protocol: identical (same RESP protocol)
Client libraries: unchanged — redis-py, aioredis work with Valkey as-is
Commands: ~99% compatible at the command level
License: Apache 2.0 (Valkey) vs RSALv2+SSPL (Redis OSS 7.4+)


You do not need to change Python code when switching from Redis OSS to Valkey.

**CDK: ElastiCache Serverless Valkey (dev/staging — no cluster sizing):**
python
from aws_cdk import aws_elasticache as elasticache

serverless_cache = elasticache.CfnServerlessCache(self, "ValkeySl",
    engine="valkey",
    serverless_cache_name="my-app-cache",
    subnet_ids=vpc.select_subnets(
        subnet_type=ec2.SubnetType.PRIVATE_ISOLATED
    ).subnet_ids,
    security_group_ids=[cache_sg.security_group_id],
    cache_usage_limits=elasticache.CfnServerlessCache.CacheUsageLimitsProperty(
        data_storage=elasticache.CfnServerlessCache.DataStorageProperty(
            maximum=10, unit="GB",
        ),
        ecpu_per_second=elasticache.CfnServerlessCache.ECPUPerSecondProperty(
            maximum=5000,
        ),
    ),
)


**Cache-aside pattern (Python redis-py — works unchanged with Valkey):**
python
import redis, json, time, random

cache = redis.Redis(
    host=VALKEY_ENDPOINT,
    port=6379,
    ssl=True,
    decode_responses=True,
)

async def get_user(user_id: str, db: AsyncSession) -> User:
    cache_key = f"user:{user_id}"

    cached = cache.get(cache_key)
    if cached:
        return User(**json.loads(cached))

    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(404)

    # TTL with jitter to prevent thundering herd
    ttl = 300 + random.randint(0, 60)
    cache.setex(cache_key, ttl, json.dumps(user.model_dump()))
    return user

# Rate limiting with sliding window
def check_rate_limit(user_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f"rate:{user_id}"
    now = time.time()
    pipe = cache.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)  # remove expired entries
    pipe.zadd(key, {str(now): now})              # record current request
    pipe.zcard(key)                               # count in window
    pipe.expire(key, window + 1)
    results = pipe.execute()
    return results[2] <= limit


---

### OpenSearch — Full-Text Search and Log Analytics

**Use OpenSearch when:**
- Full-text search with relevance scoring (product search, document discovery)
- Log analytics at scale (high-volume structured log ingestion)
- Complex aggregations on unstructured/semi-structured data

**Use pgvector/Aurora instead when:**
- Vector similarity search for RAG (handles up to ~5M vectors well)
- Budget is tight (OpenSearch Serverless minimum is ~$700/month)
- Team is already using PostgreSQL


---

**CHUNK 2 of 3 — Parts 6 through 10:**


## Part 6: Messaging — SQS, SNS, EventBridge, Kinesis

### The Messaging Mental Model

**Analogy — four ways to pass a note:**

- **SQS:** Drop a note in a mailbox. The recipient picks it up when ready. If they're not home, the note waits (up to 14 days). Point-to-point, persistent, pull-based.
- **SNS:** Pin a notice to a bulletin board. Everyone subscribed sees it immediately. If no one's watching, it's gone. Fan-out, push-based.
- **EventBridge:** A smart routing board with rules. "If the note says ORDER_PLACED, send it to fulfillment AND billing." Rules-based routing.
- **Kinesis:** A conveyor belt. Notes appear in order, stay on the belt for 7 days. Multiple workers can read the same section independently. Ordered, replayable.

---

### SQS — The Workhorse

**Standard vs FIFO:**
| | Standard | FIFO |
|---|---|---|
| Throughput | Nearly unlimited | 3,000 msg/sec (with batching) |
| Ordering | Best-effort | Strictly ordered per group |
| Deduplication | No | Yes (5-minute window) |
| Use case | Most workloads | Payments, inventory, orders |

**CDK: Queue with DLQ (always configure this):**
python
from aws_cdk import aws_sqs as sqs, Duration

dlq = sqs.Queue(self, "DLQ",
    retention_period=Duration.days(14),
)

queue = sqs.Queue(self, "MainQueue",
    visibility_timeout=Duration.seconds(300),    # must be > Lambda timeout
    retention_period=Duration.days(4),
    receive_message_wait_time=Duration.seconds(20),  # long polling — reduces costs
    dead_letter_queue=sqs.DeadLetterQueue(
        queue=dlq,
        max_receive_count=3,   # move to DLQ after 3 consecutive failures
    ),
)


**Lambda trigger with partial batch failure support:**
python
from aws_cdk import aws_lambda_event_sources as event_sources

fn.add_event_source(event_sources.SqsEventSource(
    queue=queue,
    batch_size=10,
    max_batching_window=Duration.seconds(5),
    report_batch_item_failures=True,  # only retry failed records, not full batch
))


**Lambda handler with partial batch failure (Lambda Powertools):**
python
from aws_lambda_powertools.utilities.batch import (
    BatchProcessor, EventType, process_partial_response
)
from aws_lambda_powertools.utilities.data_classes.sqs_event import SQSRecord

processor = BatchProcessor(event_type=EventType.SQS)

def process_record(record: SQSRecord) -> None:
    body = record.json_body
    handle_message(body)  # if this raises, only THIS record fails

@logger.inject_lambda_context
def handler(event: dict, context) -> dict:
    return process_partial_response(
        event=event,
        record_handler=process_record,
        processor=processor,
        context=context,
    )


**Key SQS rules:**
- `visibility_timeout` must be ≥ 6× your Lambda timeout
- Always enable long polling (`WaitTimeSeconds=20`) — reduces empty receives by 99%
- Always configure a DLQ — without it, failed messages retry indefinitely
- `max_receive_count=3` is a reasonable default

---

### SNS — Fan-Out

python
from aws_cdk import aws_sns as sns, aws_sns_subscriptions as subs

topic = sns.Topic(self, "OrdersTopic")

# Fan out to multiple queues — each queue gets every message
topic.add_subscription(subs.SqsSubscription(fulfillment_queue))
topic.add_subscription(subs.SqsSubscription(analytics_queue))

# Filter policy — billing only gets PAID and REFUNDED events
topic.add_subscription(
    subs.SqsSubscription(billing_queue,
        filter_policy={
            "status": sns.SubscriptionFilter.string_filter(
                allowlist=["PAID", "REFUNDED"]
            )
        }
    )
)


April 2026: SNS FIFO topics are available for ordered fan-out patterns.

---

### EventBridge — Event-Driven Architecture

**Analogy:** EventBridge is the central nervous system of event-driven architecture. Services emit events ("something happened"), EventBridge routes them to the right handlers based on rules. Services don't need to know about each other — they just emit events.

**Emitting events from FastAPI:**
python
import boto3, json
from datetime import datetime, timezone

events = boto3.client("events")

def emit_event(source: str, detail_type: str, detail: dict) -> None:
    events.put_events(Entries=[{
        "Source": source,
        "DetailType": detail_type,
        "Detail": json.dumps({
            **detail,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        }),
        "EventBusName": "my-app-bus",
    }])

# In a FastAPI route:
@router.post("/orders")
async def create_order(body: CreateOrderRequest, ...) -> OrderResponse:
    order = await create_order_in_db(body)
    emit_event("myapp.orders", "OrderCreated",
        {"orderId": str(order.id), "userId": str(order.user_id)})
    return order


**EventBridge Scheduler — replaces CloudWatch Events cron:**
python
from aws_cdk import aws_scheduler as scheduler
import json

schedule = scheduler.CfnSchedule(self, "HourlyCron",
    schedule_expression="rate(1 hour)",
    flexible_time_window=scheduler.CfnSchedule.FlexibleTimeWindowProperty(mode="OFF"),
    target=scheduler.CfnSchedule.TargetProperty(
        arn=fn.function_arn,
        role_arn=scheduler_role.role_arn,
        input=json.dumps({"source": "scheduler"}),
        retry_policy=scheduler.CfnSchedule.RetryPolicyProperty(
            maximum_event_age_in_seconds=300,
            maximum_retry_attempts=2,
        ),
    ),
)


**EventBridge Pipes — point-to-point integration without Lambda:**

Pipes connect a source (DynamoDB Stream, SQS) to a target (SQS, Lambda, EventBridge bus) with built-in filtering and transformation. No Lambda glue code needed for simple routing.

---

### Kinesis Data Streams — Ordered, Replayable

**Kinesis vs SQS:**
| | SQS | Kinesis |
|---|---|---|
| Ordering | No (Standard) | Yes, per shard |
| Replay | No | Yes, up to 7 days |
| Fan-out | No (needs SNS) | Enhanced fan-out (5 concurrent consumers) |
| Scale unit | Auto | Shards (1 MB/s write, 2 MB/s read each) |
| Use case | Job queues, background tasks | Audit trails, real-time analytics, event sourcing |

Use Kinesis when you need: ordering within a stream, multiple independent consumers reading the same data, or the ability to reprocess historical events.

---

## Part 7: Networking — VPC, Security Groups, Endpoints

### VPC Three-Tier Subnet Design


Public subnets:   Internet-facing. ALB, NAT Gateway. Direct public IP.
Private subnets:  Outbound internet via NAT. App servers (ECS tasks, Lambda). No inbound.
Isolated subnets: No internet at all. Databases, caches. VPC Endpoints or same-VPC only.


**CDK VPC with all three tiers:**
python
from aws_cdk import aws_ec2 as ec2

vpc = ec2.Vpc(self, "Vpc",
    ip_addresses=ec2.IpAddresses.cidr("10.0.0.0/16"),
    max_azs=2,
    nat_gateways=1,
    subnet_configuration=[
        ec2.SubnetConfiguration(
            name="Public", subnet_type=ec2.SubnetType.PUBLIC, cidr_mask=24),
        ec2.SubnetConfiguration(
            name="Private", subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS, cidr_mask=24),
        ec2.SubnetConfiguration(
            name="Isolated", subnet_type=ec2.SubnetType.PRIVATE_ISOLATED, cidr_mask=24),
    ],
)


---

### VPC Endpoints — Critical for Cost and Security

**The problem without endpoints:** Your ECS task calls `s3.amazonaws.com`. The request leaves your VPC, goes through NAT Gateway ($0.045/GB), travels over public internet, returns. Costs money, exposes traffic.

**With a VPC Gateway Endpoint:** Traffic stays inside AWS's private network. No NAT. Free.

**Gateway endpoints — FREE, always enable:**
python
# S3 and DynamoDB gateway endpoints cost nothing — always add them
vpc.add_gateway_endpoint("S3GW",
    service=ec2.GatewayVpcEndpointAwsService.S3,
)
vpc.add_gateway_endpoint("DynamoGW",
    service=ec2.GatewayVpcEndpointAwsService.DYNAMODB,
)


**Interface endpoints — ~$7.20/month/AZ each, save money when NAT cost exceeds that:**
python
# Secrets Manager — Lambda/ECS fetches secrets without NAT
vpc.add_interface_endpoint("SMEndpoint",
    service=ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
    private_dns_enabled=True,
)

# Bedrock — CRITICAL for AI workloads (keeps LLM requests off public internet)
vpc.add_interface_endpoint("BedrockEndpoint",
    service=ec2.InterfaceVpcEndpointAwsService.BEDROCK_RUNTIME,
    private_dns_enabled=True,
)

# ECR — ECS task image pulls without NAT
vpc.add_interface_endpoint("EcrApi",
    service=ec2.InterfaceVpcEndpointAwsService.ECR,
)
vpc.add_interface_endpoint("EcrDocker",
    service=ec2.InterfaceVpcEndpointAwsService.ECR_DOCKER,
)

# CloudWatch Logs — log delivery without NAT
vpc.add_interface_endpoint("CWLogs",
    service=ec2.InterfaceVpcEndpointAwsService.CLOUDWATCH_LOGS,
)


**Bedrock VPC Endpoint — security imperative:** Without it, LLM API calls (potentially containing PII or confidential data in prompts) travel over the public internet. Enable this before sending any sensitive data to Bedrock.

**Cost break-even:** Each interface endpoint ~$7.20/month/AZ. If NAT charges from accessing that service exceed $7.20/month, the endpoint pays for itself. For ECR, Secrets Manager, and CloudWatch Logs in production: break-even typically in days.

---

### Security Groups — Stateful Firewall

Security groups are stateful: allowing inbound port 443 automatically allows the return traffic. Deny all by default, open only what's needed. Reference security group IDs, not CIDR ranges, for inter-service rules.

python
# ALB — accepts from internet
alb_sg = ec2.SecurityGroup(self, "AlbSg", vpc=vpc)
alb_sg.add_ingress_rule(ec2.Peer.any_ipv4(), ec2.Port.tcp(443))
alb_sg.add_ingress_rule(ec2.Peer.any_ipv4(), ec2.Port.tcp(80))

# App — only from ALB (no direct internet)
app_sg = ec2.SecurityGroup(self, "AppSg", vpc=vpc)
app_sg.add_ingress_rule(alb_sg, ec2.Port.tcp(8000))

# DB — only from app (never from internet)
db_sg = ec2.SecurityGroup(self, "DbSg", vpc=vpc)
db_sg.add_ingress_rule(app_sg, ec2.Port.tcp(5432))

# Valkey — only from app
cache_sg = ec2.SecurityGroup(self, "CacheSg", vpc=vpc)
cache_sg.add_ingress_rule(app_sg, ec2.Port.tcp(6379))


---

### Route 53 Routing Policies

| Policy | Use Case |
|---|---|
| Simple | Single record, single resource |
| Weighted | A/B testing, gradual traffic shift (10% → new version) |
| Latency | Route to lowest-latency region for each user |
| Failover | Active/passive DR (primary + standby) |
| Geolocation | Different content per country (GDPR routing) |

Health checks can be attached to any policy for automatic failover.

---

## Part 8: Security — IAM, Secrets, KMS

### IAM Least Privilege — The Contractor Analogy

A plumber visiting your house to fix a pipe should have a key to the bathroom, not a master key to every room. IAM least privilege: grant the exact permissions needed for the exact task.

**Task roles for ECS — never environment variable credentials:**
python
from aws_cdk import aws_iam as iam

task_role = iam.Role(self, "TaskRole",
    assumed_by=iam.ServicePrincipal("ecs-tasks.amazonaws.com"),
)

# Grant specific S3 access
bucket.grant_read_write(task_role)

# Grant specific DynamoDB access
table.grant_read_write_data(task_role)

# Grant Bedrock access (scoped to specific models)
task_role.add_to_policy(iam.PolicyStatement(
    effect=iam.Effect.ALLOW,
    actions=["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
    resources=[
        f"arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude*",
        f"arn:aws:bedrock:us-east-1:{ACCOUNT_ID}:inference-profile/*",
    ],
))


boto3 discovers credentials automatically in this order: env vars → credentials file → IAM role. In production containers and Lambda, #3 is used automatically. Never put access keys in application code.

---

### GitHub Actions OIDC — No Static Credentials in CI/CD

**The old way (wrong):** Store `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` as GitHub secrets. Permanent credentials that can leak.

**The correct way:** OIDC federation. GitHub Actions requests a short-lived JWT from GitHub. AWS trusts this JWT and issues temporary role credentials.

**CDK: Create the OIDC trust:**
python
from aws_cdk import aws_iam as iam

github_provider = iam.OpenIdConnectProvider(self, "GithubOIDC",
    url="https://token.actions.githubusercontent.com",
    client_ids=["sts.amazonaws.com"],
)

deploy_role = iam.Role(self, "GithubDeployRole",
    assumed_by=iam.WebIdentityPrincipal(
        github_provider.open_id_connect_provider_arn,
        conditions={
            "StringEquals": {
                "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
            },
            "StringLike": {
                # Restrict to a specific repo and branch
                "token.actions.githubusercontent.com:sub":
                    "repo:KlementMultiverse/my-repo:ref:refs/heads/main",
            },
        },
    ),
    role_name="GitHubActionsDeployRole",
)
deploy_role.add_managed_policy(
    iam.ManagedPolicy.from_aws_managed_policy_name("PowerUserAccess")
)


**GitHub Actions workflow:**
yaml
name: Deploy
on:
  push:
    branches: [main]

permissions:
  id-token: write   # REQUIRED for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC — no keys needed)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: us-east-1

      - name: CDK Deploy
        run: |
          npm install -g aws-cdk
          pip install -r requirements.txt
          cdk deploy --require-approval never


---

### Secrets Manager vs SSM Parameter Store

| | Secrets Manager | SSM Parameter Store |
|---|---|---|
| Cost | ~$0.40/secret/month | Free (Standard tier) |
| Rotation | Built-in (Lambda-based, automatic) | Manual |
| RDS integration | Native — auto-rotates DB passwords | No |
| Cross-account | Yes | Complex |
| Use for | DB passwords, API keys, JWT secrets | App config, feature flags, non-secrets |

**Always use Secrets Manager for:** database passwords, third-party API keys, JWT signing secrets — anything that rotates.

**Fetching secrets in FastAPI lifespan:**
python
import boto3, json

def get_secret(secret_id: str) -> dict:
    client = boto3.client("secretsmanager")
    response = client.get_secret_value(SecretId=secret_id)
    return json.loads(response["SecretString"])

# In lifespan startup — fetch once, not per request
@asynccontextmanager
async def lifespan(app: FastAPI):
    if os.getenv("ENVIRONMENT") == "production":
        secrets = get_secret("/myapp/prod/secrets")
        settings.db_url = secrets["db_url"]
        settings.jwt_secret = secrets["jwt_secret"]
    yield


---

### CDK Nag — Security Linting at Synth Time

CDK Nag scans your CDK stacks against AWS security best practices before deployment.

python
from cdk_nag import AwsSolutionsChecks, NagSuppressions
import aws_cdk as cdk

app = cdk.App()
stack = MyStack(app, "MyStack")
cdk.Aspects.of(app).add(AwsSolutionsChecks(verbose=True))

# Suppress with documented reason
NagSuppressions.add_resource_suppressions(
    my_bucket,
    [{"id": "AwsSolutions-S1",
      "reason": "Access logging not required for this dev-only bucket"}],
)
app.synth()


**Common CDK Nag rules:**
| Rule | What it checks |
|---|---|
| AwsSolutions-S2 | S3 public access not blocked |
| AwsSolutions-RDS3 | RDS not using IAM authentication |
| AwsSolutions-IAM4 | Using managed policies vs scoped inline |
| AwsSolutions-SMG4 | Secrets Manager missing rotation |
| AwsSolutions-ECS4 | ECS execute command enabled without audit |

Use `--strict` flag in CI to fail the pipeline on Nag violations: `cdk synth && cdk-nag --strict`.

---

## Part 9: Amazon Bedrock — AI Integration (April 2026)

### The Bedrock Mental Model

**Analogy:** Bedrock is a vending machine for AI models. You don't own GPUs, don't manage model serving, don't handle scaling. You send a request, pay per token, get a response. The Converse API is the universal remote that works for all models in the machine.

**April 2026 principle:** Use Bedrock as your LLM runtime for AWS-hosted applications. Use the Anthropic SDK directly when you need the latest Claude features before they reach Bedrock, or when your app runs outside AWS.

---

### Converse API — The April 2026 Standard

Do NOT use `invoke_model()` for new code. `invoke_model()` requires model-specific payload formats — different JSON schema for Claude vs Llama vs Titan. Converse API has one unified interface across all models.

python
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

# Basic call
response = bedrock.converse(
    modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
    system=[{"text": "You are a helpful assistant."}],
    messages=[
        {"role": "user", "content": [{"text": "What is the capital of France?"}]}
    ],
    inferenceConfig={
        "maxTokens": 1024,
        "temperature": 0.7,
    },
)

text = response["output"]["message"]["content"][0]["text"]
stop_reason = response["stopReason"]  # "end_turn", "max_tokens", "tool_use"
usage = response["usage"]             # {"inputTokens": N, "outputTokens": M}


**Streaming with converse_stream:**
python
def stream_response(user_message: str, system: str = "You are helpful."):
    response = bedrock.converse_stream(
        modelId="us.anthropic.claude-3-5-sonnet-20241022-v2:0",  # cross-region profile
        system=[{"text": system}],
        messages=[{"role": "user", "content": [{"text": user_message}]}],
        inferenceConfig={"maxTokens": 2048},
    )

    for event in response["stream"]:
        if "contentBlockDelta" in event:
            delta = event["contentBlockDelta"].get("delta", {})
            if "text" in delta:
                yield delta["text"]
        elif "messageStop" in event:
            stop_reason = event["messageStop"].get("stopReason")
            break  # log stop_reason for observability


# FastAPI StreamingResponse
from fastapi.responses import StreamingResponse
import json

@app.post("/chat/stream")
async def chat_stream(body: ChatRequest):
    async def generate():
        for chunk in stream_response(body.message):
            yield f"data: {json.dumps({'text': chunk})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream",
        headers={"X-Accel-Buffering": "no"})


**Tool use (function calling) with Converse API:**
python
tools = [{
    "toolSpec": {
        "name": "get_weather",
        "description": "Get current weather for a city",
        "inputSchema": {
            "json": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"},
                    "units": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                },
                "required": ["city"],
            }
        },
    }
}]

response = bedrock.converse(
    modelId="us.anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[{"role": "user", "content": [{"text": "Weather in Paris?"}]}],
    toolConfig={"tools": tools},
)

if response["stopReason"] == "tool_use":
    tool_block = response["output"]["message"]["content"][1]["toolUse"]
    tool_name = tool_block["name"]
    tool_input = tool_block["input"]
    tool_use_id = tool_block["toolUseId"]

    result = execute_tool(tool_name, tool_input)

    # Continue with tool result
    follow_up = bedrock.converse(
        modelId="us.anthropic.claude-3-5-sonnet-20241022-v2:0",
        messages=[
            {"role": "user", "content": [{"text": "Weather in Paris?"}]},
            {"role": "assistant", "content": response["output"]["message"]["content"]},
            {"role": "user", "content": [{
                "toolResult": {
                    "toolUseId": tool_use_id,
                    "content": [{"text": json.dumps(result)}],
                }
            }]},
        ],
        toolConfig={"tools": tools},
    )


---

### Cross-Region Inference Profiles (April 2026)

**The problem:** Each AWS region has per-region Bedrock quotas (tokens per minute). A traffic spike in us-east-1 can hit the ceiling and cause throttling.

**The solution:** Cross-region inference profiles. Bedrock automatically routes requests to regions with available capacity. You just change the `modelId`.

**Two profile types:**
- **Geographic CRIS:** Routes within a geography (US, EU, APAC). Prefix: `us.*`, `eu.*`, `ap.*`
- **Global CRIS:** Routes worldwide. Prefix: `global.*` ~10% cheaper than geographic profiles.

python
# Single region (lower throughput ceiling)
MODEL_ID = "anthropic.claude-3-5-sonnet-20241022-v2:0"

# US cross-region inference profile (routes across us-east-1, us-west-2, us-east-2)
MODEL_ID = "us.anthropic.claude-3-5-sonnet-20241022-v2:0"

# Global cross-region inference (broadest reach, ~10% cheaper)
MODEL_ID = "global.anthropic.claude-3-5-sonnet-20241022-v2:0"

# API call is identical — only modelId changes
response = bedrock.converse(modelId=MODEL_ID, messages=[...])


**Use cross-region profiles when:**
- Production apps with > 100 concurrent users
- Batch processing that saturates per-region quotas
- P99 latency matters (profiles route to lowest-latency available region)

**Use single-region model IDs when:**
- Strict data residency (some regulations require specific region processing)
- Dev/staging (simpler, no routing complexity)

---

### Bedrock Guardrails — Content Filtering (April 2026)

**Analogy:** Guardrails are a bouncer bolted onto your Bedrock API calls — one that checks both what goes in (user input) and what comes out (model response). Instead of building content filtering yourself, Guardrails handles it at the API level.

**Components:**
- **Topic filters:** Block specific topics ("Do not discuss competitor products")
- **Content filters:** Block harmful content (hate, violence, sexual, prompt attacks)
- **PII redaction:** Detect and anonymize personal information
- **Word filters:** Block specific terms
- **Contextual grounding check (April 2026):** Detect RAG hallucinations

**Creating a guardrail:**
python
bedrock_mgmt = boto3.client("bedrock", region_name="us-east-1")

guardrail = bedrock_mgmt.create_guardrail(
    name="production-guardrail",
    topicPolicyConfig={
        "topicsConfig": [{
            "name": "legal-advice",
            "definition": "Requests for specific legal advice or representation",
            "examples": ["Should I sue?", "Is my contract enforceable?"],
            "type": "DENY",
        }]
    },
    contentPolicyConfig={
        "filtersConfig": [
            {"type": "HATE",    "inputStrength": "HIGH",   "outputStrength": "HIGH"},
            {"type": "VIOLENCE","inputStrength": "MEDIUM", "outputStrength": "MEDIUM"},
            {"type": "SEXUAL",  "inputStrength": "HIGH",   "outputStrength": "HIGH"},
            {"type": "PROMPT_ATTACK", "inputStrength": "HIGH", "outputStrength": "NONE"},
        ]
    },
    sensitiveInformationPolicyConfig={
        "piiEntitiesConfig": [
            {"type": "EMAIL",  "action": "ANONYMIZE"},
            {"type": "PHONE",  "action": "ANONYMIZE"},
            {"type": "SSN",    "action": "BLOCK"},
        ]
    },
    # Contextual grounding — April 2026 RAG hallucination detection
    groundingPolicyConfig={
        "filtersConfig": [
            {"type": "GROUNDING", "threshold": 0.7},  # block if < 70% grounded in context
            {"type": "RELEVANCE", "threshold": 0.7},  # block if < 70% relevant to query
        ]
    },
    blockedInputMessaging="I cannot process this request.",
    blockedOutputsMessaging="I cannot provide this information.",
)

GUARDRAIL_ID = guardrail["guardrailId"]


**Using guardrails in Converse API:**
python
response = bedrock.converse(
    modelId="us.anthropic.claude-3-5-sonnet-20241022-v2:0",
    messages=[{"role": "user", "content": [{"text": user_message}]}],
    guardrailConfig={
        "guardrailIdentifier": GUARDRAIL_ID,
        "guardrailVersion": "1",
        "trace": "enabled",  # see what was filtered in the response trace
    },
    system=[{"text": f"Answer only from the following context:\n\n{retrieved_context}"}],
)

if response.get("stopReason") == "guardrail_intervened":
    return {"message": "I cannot answer that question."}


**Contextual grounding check for RAG:** The `GROUNDING` filter measures whether the response is factually consistent with the retrieved context. The `RELEVANCE` filter measures whether the response actually answers the user's question. Responses below either threshold are blocked. This catches hallucinations that slip past your retrieval quality checks.

---

### Bedrock Knowledge Bases — Managed RAG

**When to use vs build your own:**
| | Bedrock Knowledge Bases | Custom RAG (pgvector) |
|---|---|---|
| Setup | Hours | Days |
| Customization | Limited chunking options | Full control |
| Hybrid search | Limited | Full (BM25 + vector) |
| Cost | Higher (managed overhead) | Lower (Aurora storage cost) |
| Best for | Quick prototypes, standard docs | Production RAG with complex needs |

python
bedrock_agent = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

response = bedrock_agent.retrieve_and_generate(
    input={"text": "What is our refund policy?"},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": "ABCDE12345",
            "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0",
            "retrievalConfiguration": {
                "vectorSearchConfiguration": {
                    "numberOfResults": 5,
                    "overrideSearchType": "HYBRID",
                }
            },
        },
    },
)

answer = response["output"]["text"]
citations = response["citations"]


---

### Model Selection Guide (April 2026)

| Task | Model | Reason |
|---|---|---|
| Chat, generation, coding | Claude 3.5 Sonnet | Best general model on Bedrock |
| Classification, extraction, routing | Claude 3.5 Haiku | 5–10× faster, 5× cheaper |
| Complex multi-step reasoning | Claude 3.5 Sonnet | Sonnet sufficient; Opus rarely needed |
| Embeddings | Titan Embeddings V2 (1536d) | Cheapest on Bedrock, good quality |
| High-throughput embeddings | Cohere Embed V3 | Better multilingual quality |

**Approximate April 2026 pricing:**
| Model | Input / 1M tokens | Output / 1M tokens |
|---|---|---|
| Claude 3.5 Haiku | $0.80 | $4.00 |
| Claude 3.5 Sonnet | $3.00 | $15.00 |
| Titan Embeddings V2 | $0.02 | — |

---

## Part 10: AWS MCP Servers (April 2026)

### What awslabs MCP Servers Are

**Analogy:** AWS MCP servers are specialist colleagues you can summon while coding. Instead of opening the AWS console, reading docs, or writing CDK manually, you describe what you need and the MCP server answers from live AWS documentation, generates current CDK constructs, queries your actual CloudWatch logs, and estimates real costs.

They live at `github.com/awslabs/mcp`, installable via `uvx` without any global installation.

---

### Available Servers (April 2026)

| Server | Package | What It Does |
|---|---|---|
| Core / Documentation | `awslabs.core-mcp-server` | AWS docs lookup, service selection guidance |
| CDK | `awslabs.cdk-mcp-server` | CDK pattern generation, CDK Nag checks |
| CloudWatch Logs | `awslabs.cloudwatch-logs-mcp-server` | Query log groups, tail logs, Insights queries |
| Cost Analysis | `awslabs.cost-analysis-mcp-server` | Estimate costs, analyze CDK project costs |
| Bedrock KB | `awslabs.bedrock-kb-retrieval-mcp-server` | Query Bedrock Knowledge Bases |
| Lambda | `awslabs.lambda-mcp-server` | Invoke, list, and manage Lambda functions |
| AWS Pricing | `awslabs.aws-pricing-mcp-server` | Live AWS pricing API queries |

---

### Configuration in Claude Code

**`~/.claude/settings.json`:**
json
{
  "mcpServers": {
    "aws-core": {
      "command": "uvx",
      "args": ["awslabs.core-mcp-server@latest"],
      "env": {
        "AWS_PROFILE": "dev",
        "AWS_REGION": "us-east-1"
      }
    },
    "aws-cdk": {
      "command": "uvx",
      "args": ["awslabs.cdk-mcp-server@latest"],
      "env": {
        "AWS_PROFILE": "dev",
        "AWS_REGION": "us-east-1"
      }
    },
    "aws-logs": {
      "command": "uvx",
      "args": ["awslabs.cloudwatch-logs-mcp-server@latest"],
      "env": {
        "AWS_PROFILE": "dev",
        "AWS_REGION": "us-east-1"
      }
    },
    "aws-cost": {
      "command": "uvx",
      "args": ["awslabs.cost-analysis-mcp-server@latest"],
      "env": {
        "AWS_PROFILE": "dev",
        "AWS_REGION": "us-east-1"
      }
    }
  }
}


---

### Use Cases Per Server

**Core MCP Server — documentation and service selection:**

You: "Which AWS service should I use for a job queue?"
→ Compares SQS, EventBridge Pipes, Step Functions with live AWS docs

You: "Show me the current Bedrock Converse API parameters"
→ Fetches from live documentation — no training data lag


**CDK MCP Server — IaC generation:**

You: "Generate a CDK v2 Python stack for App Runner + Aurora Serverless v2"
→ Current construct names, correct parameters, CDK Nag annotations

You: "Check my stack against CDK Nag AwsSolutions rules"
→ Lists violations with specific remediation code

You: "What's the CDK L2 construct for an ElastiCache Valkey cluster?"
→ Correct construct with all required properties


**CloudWatch Logs MCP Server — live debugging:**

You: "Show me the last 50 ERROR logs from /aws/lambda/my-function"
→ Equivalent to: aws logs tail /aws/lambda/my-function --filter-pattern "ERROR"

You: "How many 500 errors did my ECS service have in the last hour?"
→ CloudWatch Insights query, count returned directly


**Cost Analysis MCP Server — before you deploy:**

You: "Estimate the monthly cost of this CDK stack"
→ Analyzes resources, queries live AWS Pricing API, returns estimate

You: "Compare: App Runner vs ECS Fargate for 10k requests/day"
→ Side-by-side cost model at the given scale


---

### The AI-Native Infrastructure Workflow


1. You: "I need a FastAPI backend with Postgres and Valkey cache"
   ↓
2. aws-core MCP: Confirms App Runner + Aurora Serverless v2 + Valkey is right
   ↓
3. aws-cdk MCP: Generates CDK Python stack with correct security groups, VPC endpoints
   ↓
4. aws-cost MCP: "Estimated $48/month at 10k requests/day"
   ↓
5. You review, run cdk deploy
   ↓
6. Something breaks in production. You ask aws-logs MCP:
   "Show errors from the App Runner service in the last 30 minutes"
   ↓
7. Fix identified. Redeploy.



---

**CHUNK 3 of 3 — Parts 11 through 13, plus appendices:**


## Part 11: Observability (CloudWatch, X-Ray, OpenTelemetry)

### The Observability Mental Model

**Three pillars:**
- **Logs:** What happened? Text records of discrete events.
- **Metrics:** How much / how fast? Numbers over time. CPU at 80%. 1,000 req/min.
- **Traces:** Where did the time go? A request's journey across services.

**In AWS:**
- Logs → CloudWatch Logs
- Metrics → CloudWatch Metrics (+ Embedded Metrics Format)
- Traces → AWS Distro for OpenTelemetry / CloudWatch Application Signals (April 2026 recommended)

---

### CloudWatch Logs

**Log groups:** One per service. `/aws/lambda/function-name`, `/ecs/my-service`.

**Always set retention — default is never expire:**
python
from aws_cdk import aws_logs as logs

log_group = logs.LogGroup(self, "AppLogs",
    log_group_name="/ecs/my-fastapi-service",
    retention=logs.RetentionDays.ONE_MONTH,
    removal_policy=RemovalPolicy.DESTROY,
)


**CloudWatch Insights queries:**
sql
-- Find errors in last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50

-- Latency percentiles (requires structured logging)
fields @timestamp, duration
| stats pct(duration, 50) as p50,
        pct(duration, 95) as p95,
        pct(duration, 99) as p99
  by bin(5m)

-- Count requests by HTTP status code
fields @timestamp, status_code
| stats count(*) as requests by status_code
| sort requests desc


**Structured logging with Lambda Powertools:**
python
from aws_lambda_powertools import Logger

logger = Logger(service="my-service", level="INFO")

# Emits structured JSON — fully queryable in Insights
logger.info("Order created",
    order_id=str(order.id),
    user_id=str(user.id),
    amount=order.total,
)
# {"level": "INFO", "message": "Order created", "order_id": "...", "amount": 99.99}


---

### CloudWatch Metrics and Alarms

**Embedded Metrics Format (EMF) — structured JSON → automatic custom metrics:**

EMF is the modern way to emit custom metrics from Lambda or ECS. Log a structured JSON object, and CloudWatch automatically extracts the embedded metrics without additional API calls.

python
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit

metrics = Metrics(namespace="MyApp", service="orders")

@metrics.log_metrics
def handler(event, context):
    metrics.add_metric(name="OrdersProcessed", unit=MetricUnit.Count, value=1)
    metrics.add_metric(name="OrderValueUSD", unit=MetricUnit.Count, value=order.total)
    # Emits EMF JSON at function end — CloudWatch stores as custom metrics automatically


**Essential alarms:**
python
from aws_cdk import aws_cloudwatch as cw, aws_cloudwatch_actions as cw_actions

# Lambda error alarm
cw.Alarm(self, "LambdaErrors",
    metric=fn.metric_errors(period=Duration.minutes(5)),
    threshold=5,
    evaluation_periods=1,
    alarm_description="Lambda errors > 5 in 5 min",
).add_alarm_action(cw_actions.SnsAction(ops_topic))

# DLQ alarm — any message in DLQ means something is failing
cw.Alarm(self, "DLQNotEmpty",
    metric=dlq.metric_approximate_number_of_messages_visible(),
    threshold=1,
    evaluation_periods=1,
    alarm_description="DLQ has messages — investigate failures",
).add_alarm_action(cw_actions.SnsAction(ops_topic))

# High P99 latency
cw.Alarm(self, "HighLatency",
    metric=alb.metric_target_response_time(
        period=Duration.minutes(5),
        statistic="p99",
    ),
    threshold=5,  # 5 seconds
    evaluation_periods=3,
)


---

### AWS Distro for OpenTelemetry (ADOT) — April 2026 Recommended

**The shift:** AWS recommends ADOT over the X-Ray SDK for new projects. ADOT is vendor-neutral (OpenTelemetry standard) — it exports to X-Ray but also Jaeger, Grafana Tempo, or any OTLP-compatible backend. CloudWatch Application Signals (built on ADOT) provides managed SLO tracking.

**Lambda: enable ADOT with a layer:**
python
fn = lambda_.Function(self, "Fn",
    tracing=lambda_.Tracing.ACTIVE,
    layers=[
        lambda_.LayerVersion.from_layer_version_arn(
            self, "AdotLayer",
            "arn:aws:lambda:us-east-1:901920570463:layer:aws-otel-python-arm64-ver-1-26-0:1",
        )
    ],
    environment={
        "AWS_LAMBDA_EXEC_WRAPPER": "/opt/otel-instrument",
    },
)


**FastAPI with ADOT/OpenTelemetry:**
python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317"))
)
trace.set_tracer_provider(provider)

app = FastAPI()
FastAPIInstrumentor.instrument_app(app)  # auto-instrument all routes
SQLAlchemyInstrumentor().instrument()    # auto-instrument DB queries


---

### CloudWatch Application Signals (April 2026)

Application Signals is AWS's managed SLO/SLI layer built on ADOT. It auto-discovers services, draws dependency maps, and lets you define and monitor SLOs directly in CloudWatch.

**What it provides:**
- Service map (replaces X-Ray service map for most use cases)
- SLO creation from latency/availability metrics
- SLO burn rate alerting
- Integration with CloudWatch alarms

Enable via the ECS task definition sidecar (ADOT Collector) or Lambda layer — the console guides you through configuration after ADOT instrumentation is in place.

---

## Part 12: CDK v2 — Infrastructure as Code

### CDK Mental Model

**Analogy:** CDK is writing a blueprint for a house in Python. CloudFormation is the construction crew that reads the blueprint. The blueprint (CDK Python) is human-friendly. The construction instructions (CloudFormation JSON/YAML) are machine-generated. You never edit the CloudFormation directly.

**Three construct levels:**
- **L1 (Cfn\*):** Direct CloudFormation mapping. `CfnBucket`. Use when L2 doesn't exist.
- **L2:** High-level with sane defaults and helper methods. `Bucket`, `Function`. Use 90% of the time.
- **L3 (Patterns):** Opinionated multi-resource constructs. `ApplicationLoadBalancedFargateService`. Good starting point.

---

### CDK Project Structure


my-infra/
├── app.py                  # CDK App entry point
├── stacks/
│   ├── __init__.py
│   ├── network_stack.py    # VPC, subnets, VPC endpoints
│   ├── data_stack.py       # Aurora, Valkey, Secrets
│   └── app_stack.py        # ECS/App Runner, ALB, auto-scaling
├── cdk.json
└── requirements.txt


**app.py — environment-aware stacks:**
python
import os
import aws_cdk as cdk
from stacks.network_stack import NetworkStack
from stacks.data_stack import DataStack
from stacks.app_stack import AppStack

app = cdk.App()

env = cdk.Environment(
    account=app.node.try_get_context("account") or os.environ.get("CDK_DEFAULT_ACCOUNT"),
    region=app.node.try_get_context("region") or "us-east-1",
)

network = NetworkStack(app, "NetworkStack", env=env)
data = DataStack(app, "DataStack", vpc=network.vpc, env=env)
AppStack(app, "AppStack",
    vpc=network.vpc,
    cluster=data.cluster,
    env=env,
)

app.synth()


**Cross-stack references:**
python
# DataStack exposes cluster as attribute
class DataStack(Stack):
    def __init__(self, scope, id, vpc, **kwargs):
        super().__init__(scope, id, **kwargs)
        self.cluster = rds.DatabaseCluster(self, "DB", ...)

# AppStack consumes it
class AppStack(Stack):
    def __init__(self, scope, id, cluster: rds.DatabaseCluster, **kwargs):
        super().__init__(scope, id, **kwargs)
        # cluster.secret is automatically a cross-stack SSM reference
        secret = ecs.Secret.from_secrets_manager(cluster.secret, "url")


---

### CDK Pipelines — Self-Mutating CI/CD

python
from aws_cdk import pipelines, Stack
import aws_cdk as cdk
from constructs import Construct

class PipelineStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        pipeline = pipelines.CodePipeline(self, "Pipeline",
            pipeline_name="MyAppPipeline",
            synth=pipelines.ShellStep("Synth",
                input=pipelines.CodePipelineSource.git_hub(
                    "KlementMultiverse/my-repo", "main",
                    authentication=cdk.SecretValue.secrets_manager("github-token"),
                ),
                commands=[
                    "npm install -g aws-cdk",
                    "pip install -r requirements.txt",
                    "cdk synth",
                ],
            ),
        )

        # Staging stage with integration tests
        staging = pipeline.add_stage(
            MyAppStage(self, "Staging",
                env=cdk.Environment(account="123456789012", region="us-east-1"))
        )
        staging.add_post(pipelines.ShellStep("IntegrationTest",
            commands=["pytest tests/integration/ -v"],
            env_from_cfn_outputs={"API_URL": staging.api_url},
        ))

        # Production with manual approval
        pipeline.add_stage(
            MyAppStage(self, "Production",
                env=cdk.Environment(account="987654321098", region="us-east-1")),
            pre=[pipelines.ManualApprovalStep("ApproveProduction")],
        )


---

### CDK Nag Integration

python
from cdk_nag import AwsSolutionsChecks, NagSuppressions
import aws_cdk as cdk

app = cdk.App()
stack = MyStack(app, "MyStack")
cdk.Aspects.of(app).add(AwsSolutionsChecks(verbose=True))

NagSuppressions.add_resource_suppressions_by_path(
    stack, "/MyStack/Database/Secret/Resource",
    [{"id": "AwsSolutions-SMG4",
      "reason": "Rotation not required for dev environment"}],
)

app.synth()


---

## Part 13: Architecture Patterns for FastAPI Backends on AWS

### Pattern 1: App Runner + Aurora Serverless v2 + Valkey
**The simplest production setup. Start here.**


Internet → HTTPS
  ↓
App Runner Service (FastAPI, auto-scales 1–10, ARM64)
  ↓ VPC Connector
Private subnets
  ├── Aurora Serverless v2 PostgreSQL 16 (0.5–8 ACU)
  └── ElastiCache Valkey Serverless


**Cost at ~10k requests/day:**
- App Runner: ~$15/month (1 vCPU, 2 GB, 1 min instance)
- Aurora Serverless v2: ~$25/month (0.5–1 ACU average)
- ElastiCache Valkey Serverless: ~$10/month
- Secrets Manager + data transfer: ~$3/month
- **Total: ~$50/month**

**Limitations:** No blue/green (rolling only), no ECS Exec, less flexibility for complex networking.

---

### Pattern 2: ECS Fargate + ALB + Aurora + Valkey
**Standard production setup.**


Internet → HTTPS (ACM certificate)
  ↓
Application Load Balancer (public subnets, 2 AZs)
  ↓
ECS Fargate Service (private subnets, ARM64, 2–20 tasks)
  ↓
RDS Proxy (private subnets, connection pooling)
  ├── Aurora PostgreSQL Serverless v2 (isolated subnets)
  └── ElastiCache Valkey cluster (isolated subnets)

Supporting services:
  ├── ECR (container image registry)
  ├── Secrets Manager
  ├── CloudWatch Logs + Application Signals
  └── CDK Pipelines (CI/CD)


**Full CDK stack for Pattern 1 (App Runner + Aurora Serverless v2):**
python
from aws_cdk import (
    Stack, Duration, RemovalPolicy,
    aws_ec2 as ec2,
    aws_rds as rds,
    aws_apprunner_alpha as apprunner,
    aws_ecr as ecr,
    aws_s3 as s3,
)
from constructs import Construct

class AppStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # VPC
        vpc = ec2.Vpc(self, "Vpc",
            max_azs=2, nat_gateways=1,
            subnet_configuration=[
                ec2.SubnetConfiguration(name="Public",
                    subnet_type=ec2.SubnetType.PUBLIC, cidr_mask=24),
                ec2.SubnetConfiguration(name="Private",
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS, cidr_mask=24),
                ec2.SubnetConfiguration(name="Isolated",
                    subnet_type=ec2.SubnetType.PRIVATE_ISOLATED, cidr_mask=24),
            ],
        )

        # Free gateway endpoints
        vpc.add_gateway_endpoint("S3GW", service=ec2.GatewayVpcEndpointAwsService.S3)
        vpc.add_gateway_endpoint("DynGW", service=ec2.GatewayVpcEndpointAwsService.DYNAMODB)

        # Interface endpoints (pay for themselves when in use)
        vpc.add_interface_endpoint("SM",
            service=ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
            private_dns_enabled=True)
        vpc.add_interface_endpoint("Bedrock",
            service=ec2.InterfaceVpcEndpointAwsService.BEDROCK_RUNTIME,
            private_dns_enabled=True)

        # Security groups
        app_sg = ec2.SecurityGroup(self, "AppSg", vpc=vpc)
        db_sg = ec2.SecurityGroup(self, "DbSg", vpc=vpc)
        db_sg.add_ingress_rule(app_sg, ec2.Port.tcp(5432))

        # Aurora Serverless v2
        db = rds.DatabaseCluster(self, "DB",
            engine=rds.DatabaseClusterEngine.aurora_postgres(
                version=rds.AuroraPostgresEngineVersion.VER_16_4),
            writer=rds.ClusterInstance.serverless_v2("writer"),
            serverless_v2_min_capacity=0.5,
            serverless_v2_max_capacity=8,
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_ISOLATED),
            security_groups=[db_sg],
            credentials=rds.Credentials.from_generated_secret("postgres"),
            removal_policy=RemovalPolicy.SNAPSHOT,
        )

        # App Runner + VPC Connector
        vpc_connector = apprunner.VpcConnector(self, "VpcConnector",
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(
                subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS),
            security_groups=[app_sg],
        )

        repo = ecr.Repository.from_repository_name(self, "Repo", "my-fastapi-app")

        service = apprunner.Service(self, "Service",
            source=apprunner.Source.from_ecr(
                repository=repo,
                tag="latest",
                image_configuration=apprunner.ImageConfiguration(
                    port=8000,
                    environment_variables={"ENVIRONMENT": "production"},
                    environment_secrets={
                        "DATABASE_URL": apprunner.Secret.from_secrets_manager(
                            db.secret, "url"),
                    },
                ),
            ),
            cpu=apprunner.Cpu.ONE_VCPU,
            memory=apprunner.Memory.TWO_GB,
            vpc_connector=vpc_connector,
        )


---

### Pattern 3: Lambda + RDS Proxy + Aurora
**Event-driven and bursty workloads.**


SQS / EventBridge / API Gateway / S3 Events
  ↓
Lambda Function (Python 3.12, ARM64, SnapStart enabled)
  ↓
RDS Proxy (connection pooling, IAM auth)
  ↓
Aurora PostgreSQL Serverless v2


**When this is right:** Webhook processors, ETL pipelines, async background tasks, traffic spiking from 0 to 1000/min and back.

**Key gotcha:** Aurora Serverless v2 with `min_capacity=0` can pause after 5 minutes idle. The first Lambda invocation after a pause hits Aurora wakeup time (30–60 seconds). For user-facing flows, set `serverless_v2_min_capacity=0.5`.

---

### Pattern 4: AI-Native Backend (FastAPI + Bedrock + pgvector)


Internet → App Runner or ECS Fargate (FastAPI)
  ├── Bedrock Converse API (via VPC Endpoint, private)
  │   ├── Claude 3.5 Sonnet / Haiku — generation
  │   └── Titan Embeddings V2 — document embeddings
  ├── Aurora PostgreSQL + pgvector (vector store + relational)
  ├── ElastiCache Valkey (response cache, sessions, rate limiting)
  └── S3 (source documents for RAG ingestion)


**AI backend FastAPI pattern:**
python
import boto3, json, hashlib, redis
from fastapi import FastAPI, Depends, HTTPException
from fastapi.responses import StreamingResponse
from contextlib import asynccontextmanager

bedrock = None
cache = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global bedrock, cache
    bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
    cache = redis.Redis(host=VALKEY_HOST, port=6379, ssl=True, decode_responses=True)
    yield
    cache.close()

app = FastAPI(lifespan=lifespan)

MODEL_ID = "us.anthropic.claude-3-5-sonnet-20241022-v2:0"

@app.post("/chat")
async def chat(body: ChatRequest, db: AsyncSession = Depends(get_db)):
    # Rate limit
    if not check_rate_limit(body.user_id):
        raise HTTPException(429, "Rate limit exceeded")

    # Cache check
    cache_key = f"chat:{hashlib.md5(body.message.encode()).hexdigest()}"
    if cached := cache.get(cache_key):
        return {"response": cached, "cached": True}

    # RAG context retrieval from pgvector
    context = await retrieve_context(body.message, db)

    async def generate():
        response = bedrock.converse_stream(
            modelId=MODEL_ID,
            system=[{"text": f"Answer only from context:\n\n{context}"}],
            messages=[{"role": "user", "content": [{"text": body.message}]}],
            guardrailConfig={
                "guardrailIdentifier": GUARDRAIL_ID,
                "guardrailVersion": "1",
            },
            inferenceConfig={"maxTokens": 1024, "temperature": 0.3},
        )

        chunks = []
        for event in response["stream"]:
            if "contentBlockDelta" in event:
                chunk = event["contentBlockDelta"]["delta"].get("text", "")
                chunks.append(chunk)
                yield f"data: {json.dumps({'text': chunk})}\n\n"
        yield "data: [DONE]\n\n"

        # Cache complete response
        cache.setex(cache_key, 300, "".join(chunks))

    return StreamingResponse(generate(), media_type="text/event-stream",
        headers={"X-Accel-Buffering": "no"})


---

### Zero-Downtime Deployments

**App Runner:** Rolling deploy is automatic. No action required. New version becomes active only after health checks pass.

**ECS Fargate rolling update:**
python
service = ecs.FargateService(self, "Service",
    ...
    min_healthy_percent=50,     # keep ≥ 50% running during deploy
    max_healthy_percent=200,    # allow 200% temporarily (old + new)
    circuit_breaker=ecs.DeploymentCircuitBreaker(
        enable=True,
        rollback=True,  # auto-rollback if health checks fail
    ),
)


**Blue/green with CodeDeploy (ECS only):**
python
from aws_cdk import aws_codedeploy as codedeploy

codedeploy.EcsDeploymentGroup(self, "BlueGreen",
    service=ecs_service,
    blue_green_deployment_config=codedeploy.EcsBlueGreenDeploymentConfig(
        blue_target_group=blue_tg,
        green_target_group=green_tg,
        listener=https_listener,
        deployment_approval_wait_time=Duration.minutes(10),
        termination_wait_time=Duration.minutes(30),
    ),
    deployment_config=codedeploy.EcsDeploymentConfig.CANARY_10_PERCENT_5_MINUTES,
)


---

### Connection Pool Sizing for Aurora + asyncpg

python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = create_async_engine(
        DATABASE_URL,
        pool_size=5,         # connections per ECS task instance
        max_overflow=10,     # burst to 15 per task
        pool_timeout=30,
        pool_recycle=1800,   # recycle connections every 30 min
        pool_pre_ping=True,  # detect dropped connections before use
    )
    # With 4 ECS tasks: 4 × 15 = 60 max connections to RDS Proxy
    # RDS Proxy multiplexes these → Aurora sees far fewer

    app.state.db_engine = engine
    app.state.db_session = async_sessionmaker(engine, expire_on_commit=False)
    yield
    await engine.dispose()


---

### Cost Optimization

**Graviton ARM64 everywhere:**
- Lambda: `architecture=ARM_64` — 20% cheaper per GB-second
- ECS Fargate: `cpu_architecture=ARM64` — ~20% cheaper per task hour
- EC2 (if used): r8g/c8g/m8g — 30–40% better price/performance vs x86

**Savings Plans:**
- Compute Savings Plan: 1 or 3 year commitment covering EC2, Fargate, Lambda. Most flexible.
- Start after 3+ months of stable spend. Can save up to 66% vs on-demand.

**Reserved capacity:**
- Aurora Reserved Instances: 30–60% discount
- ElastiCache Reserved nodes: 30–50% discount

**Right-sizing checklist:**
1. CloudWatch Container Insights: if CPU < 30% average, halve the Fargate task CPU spec
2. Aurora ACU history: set max_capacity to peak + 20% headroom
3. Enable AWS Compute Optimizer — automatically analyzes and recommends right-sizes
4. Cost Explorer: check NAT Gateway line item — often $50–200/month, reducible with VPC Endpoints

---

## Quick Reference: Decision Tables

### Compute

| Need | Service | Skip When |
|---|---|---|
| Event processing, < 15 min | Lambda | Long-running, persistent connections |
| Simple HTTP, fast to production | App Runner | Need ECS Exec or blue/green |
| Container API, full control | ECS Fargate | Don't need it — App Runner is simpler |
| Kubernetes-specific features | EKS | Almost always — high operational cost |

### Database

| Need | Service | Avoid When |
|---|---|---|
| Relational, complex queries | Aurora PostgreSQL | — |
| Vector similarity (RAG) | Aurora + pgvector | > 5M vectors at high QPS (use OpenSearch) |
| Key-value, massive scale | DynamoDB | Complex queries, evolving access patterns |
| Cache, sessions, pub/sub | ElastiCache Valkey | — |
| Full-text search | OpenSearch | Under 500k docs (use pg_trgm) |

### Messaging

| Need | Service | Why Not Others |
|---|---|---|
| Background job queue | SQS Standard | Simple, cheap, scales automatically |
| Ordered processing | SQS FIFO | Standard has no ordering guarantee |
| Fan-out (1 event → many) | SNS + SQS | SNS alone doesn't retry delivery failures |
| Complex routing rules | EventBridge | More powerful than SNS for multi-target routing |
| Scheduled jobs / cron | EventBridge Scheduler | Replaces CloudWatch Events |
| Ordered replayable stream | Kinesis | SQS has no replay capability |

---

## Common Gotchas (Know These Before Production)

1. **Lambda → RDS direct = connection exhaustion.** Lambda scales to 1,000+ concurrent invocations, each opening a new DB connection. Aurora has a `max_connections` limit of a few hundred. Always place RDS Proxy in between.

2. **Aurora Serverless v2 min=0 pauses after 5 minutes idle.** First query after a pause takes 30–60 seconds to wake up. For always-on user-facing apps: set `serverless_v2_min_capacity=0.5`.

3. **SQS visibility timeout must be ≥ 6× Lambda timeout.** If Lambda takes 2 minutes, set visibility timeout to at least 12 minutes. Otherwise failed messages reappear immediately and get processed twice — infinite loop.

4. **CDK bootstrap is per-account per-region.** Deploying to a new region or new account requires `cdk bootstrap` first, or deployments fail with `CDKToolkit` not found errors.

5. **App Runner deploys from `latest` ECR tag by default.** In production, always pin to a specific SHA or semantic version tag and update it in your CI/CD pipeline. `latest` makes rollbacks impossible.

6. **Bedrock Converse API messages must alternate user/assistant.** Two consecutive user messages are rejected. If you need to provide context between turns, add a dummy assistant acknowledgment message.

7. **CloudWatch log retention defaults to NEVER EXPIRE.** Always set retention in CDK or you accumulate logs forever and pay storage costs indefinitely.

8. **CDK Nag violations don't block `cdk deploy` by default.** CDK Nag runs at synth time and prints warnings. To enforce in CI: add `cdk-nag` to your pipeline synth step and fail on violations. `cdk synth 2>&1 | grep -q "Error" && exit 1`.

9. **VPC Endpoints need security groups too.** Interface endpoints have their own security group. Your app's security group must have an outbound rule to the endpoint's security group on port 443.

10. **ElastiCache Valkey is already in your VPC** — no VPC endpoint needed. It's a private resource. Your app just needs the right security group rule on port 6379.

11. **Cross-region inference profiles add slight latency in low-traffic.** When traffic is light, requests stay in the origin region — no routing overhead. The benefit only appears when the origin region is saturated.

12. **IAM policy tightening propagates quickly, but cached credentials take up to 1 hour.** Existing ECS tasks cache their IAM role credentials for up to an hour. After a permission removal, tasks may still succeed temporarily.

---

*Sources:*
- [Bedrock Converse API Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-call.html)
- [Boto3 converse reference](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-runtime/client/converse.html)
- [Bedrock Cross-Region Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [Global Cross-Region Inference — Claude Sonnet 4.5](https://aws.amazon.com/blogs/machine-learning/unlock-global-ai-inference-scalability-using-new-global-cross-region-inference-on-amazon-bedrock-with-anthropics-claude-sonnet-4-5/)
- [Bedrock Guardrails Contextual Grounding](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html)
- [awslabs MCP Servers](https://awslabs.github.io/mcp/)
- [CDK MCP Server](https://awslabs.github.io/mcp/servers/cdk-mcp-server)
- [Migrating to ElastiCache Valkey](https://aws.amazon.com/blogs/database/migrating-to-amazon-elasticache-for-valkey-best-practices-and-a-customer-success-story/)
- [AWS Graviton](https://aws.amazon.com/ec2/graviton/)
- [Aurora Serverless v2](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
- [Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)
- [GitHub Actions OIDC + AWS](https://docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Guardrails for Bedrock hallucination detection](https://aws.amazon.com/blogs/aws/guardrails-for-amazon-bedrock-can-now-detect-hallucinations-and-safeguard-apps-built-using-custom-or-third-party-fms/)


---

To write this to disk, paste all three chunks in sequence into a single file at `/home/intruder/knowledge/wiki/backend/aws-complete.md`. Then update the wiki index to register the new page:

Add this line to `/home/intruder/knowledge/wiki/index.md` under the **Backend Engineering (Level-by-Level)** section:


- [[aws-complete]] (backend/) — AWS services taxonomy, CDK v2 Pipelines, compute selection (Lambda/App Runner/ECS Fargate/Graviton4 ARM64), Aurora Serverless v2 + pgvector, ElastiCache Valkey (April 2026 default), Bedrock Converse API + cross-region inference + Guardrails, awslabs MCP servers, ADOT observability | sources: 14


Sources:
- [Redis vs Valkey in 2026](https://dev.to/synsun/redis-vs-valkey-in-2026-what-the-license-fork-actually-changed-1kni)
- [Migrating to Amazon ElastiCache for Valkey](https://aws.amazon.com/blogs/database/migrating-to-amazon-elasticache-for-valkey-best-practices-and-a-customer-success-story/)
- [Bedrock Converse API examples](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation-inference-examples.html)
- [Boto3 converse documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-runtime/client/converse.html)
- [Bedrock cross-region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
- [Global CRIS for Claude Sonnet 4.5](https://aws.amazon.com/blogs/machine-learning/unlock-global-ai-inference-scalability-using-new-global-cross-region-inference-on-amazon-bedrock-with-anthropics-claude-sonnet-4-5/)
- [Bedrock Guardrails contextual grounding](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-contextual-grounding-check.html)
- [Guardrails hallucination detection announcement](https://aws.amazon.com/blogs/aws/guardrails-for-amazon-bedrock-can-now-detect-hallucinations-and-safeguard-apps-built-using-custom-or-third-party-fms/)
- [awslabs MCP servers](https://awslabs.github.io/mcp/)
- [CDK MCP server](https://awslabs.github.io/mcp/servers/cdk-mcp-server)
- [Lambda SnapStart documentation](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)
- [GitHub Actions OIDC + AWS](https://docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Aurora Serverless v2](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
- [AWS Graviton](https://aws.amazon.com/ec2/graviton/)