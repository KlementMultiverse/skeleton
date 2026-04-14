---
description: AWS cloud architecture rules for AI-native FastAPI backends (April 2026). 227 rules.
paths: ["*.py", "*.tf", "*.yaml", "cdk/**/*.py", "infra/**/*.py"]
---

# AWS Architecture Patterns — Rules

> This file owns: IAM, compute selection, S3, databases, messaging, networking, Bedrock, IaC, observability, anti-patterns.
> Security rules → security.md / security-complete.md. LLM observability → observability-patterns.md

---

## IAM and Security

1. NEVER use the AWS root account for any operation — create an IAM Identity Center (SSO) user or role with least privilege; root credentials must be MFA-enabled and locked in a vault; use AWS Organizations SCPs to prevent root API usage across member accounts.

2. NEVER hardcode `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in code, Dockerfiles, `.env` files, or CI secrets stores — use OIDC role assumption in CI/CD (GitHub Actions OIDC → `aws-actions/configure-aws-credentials`) and instance profiles / ECS task roles at runtime.

3. ALWAYS attach IAM roles to compute (ECS task roles, Lambda execution roles, EKS IRSA, EC2 instance profiles) instead of long-lived access keys — task roles provide auto-rotating temporary credentials via IMDS; static keys require manual rotation and are the #1 AWS credential exposure vector.

4. ALWAYS apply least privilege: list specific actions and specific resource ARNs — never use `"Action": "*"` or `"Resource": "*"` for IAM, KMS, Secrets Manager, or S3 buckets containing user data; scope every statement to the minimum ARN pattern needed.

5. ALWAYS use CDK grant methods (`bucket.grant_read(role)`, `secret.grant_read(role)`, `queue.grant_consume_messages(role)`) instead of manual `PolicyStatement` blocks — grant methods generate minimum-required policies and update automatically when resource ARNs change.

6. ALWAYS use IAM permissions boundaries on roles created by CI/CD pipelines — a pipeline that can create arbitrary roles is a privilege escalation path; boundaries cap the maximum permissions any role the pipeline creates can hold.

7. ALWAYS use OIDC role assumption for GitHub Actions and GitLab CI — configure the `aws:sub` condition to restrict to specific repo and branch; OIDC tokens are short-lived and revoke automatically.

8. ALWAYS run CDK Nag (`cdk_nag.AwsSolutionsChecks`) in CI before every deploy — it catches IAM wildcards, missing encryption, missing access logs, and insecure defaults; failures must block the pipeline.

9. NEVER grant `iam:PassRole` broadly — scope to the specific role ARN and specific service principal that can receive it; broad `PassRole` enables privilege escalation to any service.

10. ALWAYS use separate IAM roles for ECS execution role (ECR pull + CloudWatch Logs write) and ECS task role (application permissions: S3/Bedrock/DynamoDB) — conflating them creates over-permissioned application code.

11. ALWAYS enforce MFA via `aws:MultiFactorAuthPresent: "true"` condition on destructive actions (`DeleteBucket`, `DeleteCluster`, `cloudformation:DeleteStack`).

12. ALWAYS use `aws:RequestedRegion` condition to lock roles to operated regions — prevent credential misuse in regions where your logging and detection are not configured.

13. ALWAYS rotate access keys older than 90 days — use AWS Config rule `access-keys-rotated`; keys that are never rotated are silent long-term exposure risks.

14. ALWAYS store secrets in AWS Secrets Manager (not SSM Parameter Store) for database credentials, API keys, and OAuth client secrets that require automatic rotation — Secrets Manager's rotation Lambda integration rotates RDS/Aurora passwords automatically; SSM does not.

15. ALWAYS use AWS IAM Identity Center (formerly SSO) as the single authentication source for all human access across AWS accounts — avoid creating individual IAM users; IAM Identity Center integrates with your IdP (Okta, Azure AD, Google Workspace) and provides centralized access audit.

---

## Compute Selection

16. NEVER deploy a FastAPI application on Lambda if any endpoint has a P99 duration > 60 seconds — Lambda's 15-minute hard limit looks generous, but cold start + Bedrock latency + streaming regularly pushes past 60s; use ECS Fargate for consistent latency SLAs.

17. ALWAYS use Lambda for event-driven workloads (SQS consumers, S3 event processors, EventBridge targets, scheduled cron) — Lambda's per-invocation scaling and zero-idle cost are exactly right for this shape.

18. ALWAYS use ECS Fargate for production APIs requiring WebSocket support, sub-500ms P99 SLAs, streaming responses, or persistent in-memory state — Lambda cold starts and container recycling make these impossible to guarantee.

19. ALWAYS use App Runner for the first deployed iteration when the team has no existing ECS/EKS infrastructure — App Runner provides HTTPS, auto-scaling, VPC connectivity, and health checks with zero ALB/cluster configuration; migrate to ECS Fargate when you need fine-grained control.

20. ALWAYS use EKS when you have 10+ microservices, need Kubernetes-native tooling (Helm, Argo CD, Karpenter, Cilium), or when multiple teams require namespace isolation.

21. ALWAYS build for ARM64 (Graviton4 as of 2026) — Graviton4 delivers 30-40% better price-performance for Python workloads; set `cpu_architecture=ecs.CpuArchitecture.ARM64` or `architecture=lambda_.Architecture.ARM_64`; most public container images now publish multi-arch manifests.

22. NEVER connect Lambda directly to RDS/Aurora without RDS Proxy — Lambda can spawn thousands of concurrent invocations each opening a new DB connection, exhausting `max_connections` instantly; RDS Proxy pools connections and reduces DB connection count by up to 66%.

23. ALWAYS right-size Lambda memory with AWS Lambda Power Tuning before production — Lambda memory controls proportional CPU; the optimal setting for a Python FastAPI handler is rarely the default 128 MB; test at 512, 1024, 2048 MB.

24. ALWAYS use FARGATE_SPOT capacity provider for ECS worker tasks (SQS consumers, batch) with FARGATE fallback — Spot is up to 70% cheaper; the 2-minute interruption notice is manageable with proper SIGTERM handling.

25. NEVER deploy a production service to a single AZ — ECS services must have `desired_count >= 2` distributed across ≥ 2 AZs; a single-AZ service fails completely during AZ impairments.

26. ALWAYS set ECS task `health_check.start_period` to 1.5× your observed container start time — without it, ECS kills tasks before FastAPI finishes initializing.

27. ALWAYS enable ECS service deployment circuit breaker with rollback — without it, a bad deploy replaces all healthy tasks and requires manual intervention.

---

## S3 Patterns

28. ALWAYS set `BlockPublicAccess.BLOCK_ALL` on every S3 bucket at creation — never disable any of the four block-public-access settings without a documented CDN-origin use case reviewed by security.

29. NEVER construct S3 URLs by string interpolation (`https://bucket.s3.amazonaws.com/key`) — always use `generate_presigned_url()` for downloads (15-60 min expiry) and `generate_presigned_post()` for uploads (5-15 min expiry).

30. ALWAYS enable S3 versioning on buckets storing user data, ML models, or application artifacts — add `noncurrent_version_expiration=Duration.days(30)` to control storage cost.

31. ALWAYS use multipart upload for objects > 100 MB — `TransferConfig(multipart_threshold=100*1024*1024, max_concurrency=10)`; single-part uploads above 100 MB fail on intermittent networks and cannot resume.

32. ALWAYS enable server-side encryption — `BucketEncryption.S3_MANAGED` for standard; `BucketEncryption.KMS` with a CMK for HIPAA/PCI-DSS/SOC2 compliance.

33. ALWAYS add S3 lifecycle rules — Intelligent Tiering for unpredictable access patterns; explicit transitions (Standard → IA at 30 days → Glacier at 90 days) for known patterns; S3 costs accumulate silently without lifecycle policies.

34. ALWAYS add S3 event notifications to trigger downstream processing on upload — event-driven reduces latency from minutes to seconds vs polling.

35. ALWAYS use structured S3 key prefixes with versions for ML artifacts (`models/{name}/{version}/weights.bin`) — versioned keys make rollback a key change, not a data operation.

---

## Database and Cache

36. ALWAYS use Aurora PostgreSQL Serverless v2 for new production FastAPI backends on AWS — scales from 0.5 to 128 ACU within seconds, no capacity planning, fully compatible with SQLAlchemy async + asyncpg; avoid Aurora Serverless v1 (fully deprecated in 2025).

37. ALWAYS set `serverless_v2_min_capacity` to 0.5 ACU for dev/staging and ≥ 2 ACU for production — minimum capacity determines cold response time; 0.5 ACU in production causes noticeable first-request latency after idle.

38. ALWAYS place RDS Proxy between Lambda and Aurora — see rule 22; RDS Proxy also provides IAM authentication (`iam_auth=True`), avoiding DB password storage in Lambda environment variables.

39. ALWAYS use DynamoDB on-demand (`BillingMode.PAY_PER_REQUEST`) until you have ≥ 2 weeks of stable predictable traffic — incorrect provisioning causes throttling or wasted spend.

40. ALWAYS set TTL on DynamoDB tables storing ephemeral data (sessions, rate limit counters, idempotency keys) — without TTL, these tables grow without bound.

41. NEVER use DynamoDB Scan in production code — Scan is O(N) regardless of filters; replace with Query on a GSI or redesign the data model.

42. ALWAYS design DynamoDB access patterns before creating the table — primary key and GSI design cannot change after creation; adding a GSI later works, but changing the primary key requires a new table and full data migration.

43. ALWAYS use ElastiCache Valkey (the AWS-native Redis fork, replacing Redis OSS in 2025-2026) for session storage and rate limiting in ECS deployments — Valkey is now the default in new ElastiCache clusters; enable in-transit encryption and auth tokens.

44. ALWAYS use MemoryDB for Redis when Redis data must survive node failure (financial ledgers, Redis Streams as an event log) — MemoryDB writes to a Multi-AZ journal on every write; ElastiCache Redis does not.

45. ALWAYS use `asyncpg` with SQLAlchemy async engine for FastAPI + PostgreSQL — synchronous `psycopg2` blocks the event loop.

---

## Messaging

46. ALWAYS configure a DLQ on every SQS queue with `max_receive_count=3` — messages failing 3 times move to the DLQ; without a DLQ, poison messages loop indefinitely consuming processing capacity.

47. ALWAYS set `visibility_timeout` to ≥ 6× the maximum consumer processing time — if processing takes 30s, set timeout to 180s; too-short timeouts cause duplicate processing.

48. ALWAYS implement idempotent SQS handlers — standard queues provide at-least-once delivery; use Redis or DynamoDB conditional write to deduplicate processed message IDs.

49. ALWAYS use SQS long polling (`WaitTimeSeconds=20`) — short polling wastes API calls and cost; long polling reduces SQS costs by up to 95% at low throughput.

50. ALWAYS use EventBridge instead of SNS for new event-driven architectures — EventBridge adds content-based filtering, schema registry, event replay, and cross-account buses; prefer it for service-to-service events.

51. ALWAYS configure SQS as a Lambda event source with `report_batch_item_failures=True` and return `{"batchItemFailures": [...]}` — without this, a single failed message causes the entire batch to retry, reprocessing already-succeeded messages.

52. ALWAYS include a correlation ID in every SQS message body — it is the only way to trace a DLQ message back to the originating HTTP request in CloudWatch Logs.

53. ALWAYS use Kinesis Data Streams (or MSK) instead of SQS when you need message replay, time-windowed aggregations, or multiple independent consumers reading the same stream.

54. NEVER use SQS FIFO queues unless your workload genuinely requires strict per-entity ordering — FIFO has a 3,000 msg/sec ceiling vs effectively unlimited for standard; misuse creates a bottleneck.

---

## Networking

55. ALWAYS deploy all application resources (ECS tasks, Lambda ENIs, RDS, ElastiCache) in private subnets — public subnets are for internet-facing ALBs and NAT gateways only.

56. ALWAYS add VPC Gateway Endpoints for S3 and DynamoDB in every VPC — these are free and route traffic through the AWS backbone instead of NAT gateways; without them, Lambda reading from S3 100K times/day costs ~$4.50/day in NAT charges.

57. ALWAYS add VPC Interface Endpoints for Secrets Manager, SSM, ECR, CloudWatch Logs, and Bedrock when running AI workloads in private subnets — eliminates NAT data transfer charges on large Bedrock prompt/response payloads and satisfies compliance requirements that prohibit external egress.

58. NEVER use a single NAT gateway for a multi-AZ ECS service — if the NAT gateway's AZ fails, all private-subnet resources in other AZs lose internet access; use one NAT gateway per AZ, or eliminate NAT dependency via VPC endpoints for AWS services.

59. ALWAYS use Security Group references (not CIDR ranges) for inter-service access inside a VPC — `peer=other_sg` is correct; `peer=ec2.Peer.any_ipv4()` on a database security group is a critical misconfiguration.

60. ALWAYS set ALB `idle_timeout` and uvicorn `--timeout-keep-alive` such that uvicorn's keepalive exceeds the ALB idle timeout by ≥ 15 seconds — if uvicorn closes a keepalive connection before the ALB does, the ALB sends a request on a closed connection producing 502 errors; set uvicorn keepalive to 75s when ALB idle timeout is 60s.

61. ALWAYS enable ALB access logs to S3 — ALB access logs are the authoritative record for debugging 502/504 errors and security incident response; cost is negligible.

62. ALWAYS use Route53 health checks and DNS failover for multi-region deployments — health checks detect regional failures within 10-30 seconds.

---

## Bedrock and AI Services (April 2026)

63. ALWAYS use the Bedrock Converse API instead of `invoke_model` with raw JSON — `bedrock_runtime.converse()` provides a consistent interface across all models, handles message formatting, and returns structured output; `invoke_model` requires model-specific JSON that changes between model families.

64. ALWAYS check `response["stopReason"]` after every Bedrock Converse call — `"max_tokens"` means output was truncated; truncated AI responses that your code treats as complete are silent quality failures.

65. ALWAYS set explicit `maxTokens` on every Bedrock API call — omitting it defaults to the model's maximum output (200K+ tokens on claude-opus-4-6), which can produce a $10+ charge from a single malformed request.

66. PREFER Bedrock over the direct Anthropic API in regulated environments (HIPAA, PCI-DSS, FedRAMP, SOC2) or when VPC security policy prohibits outbound HTTPS to non-AWS endpoints — Bedrock signs with AWS IAM, logs to CloudTrail, and routes through VPC endpoints.

67. NEVER pass raw user input as the `text` field in a Bedrock message — wrap in explicit delimiters (`<user_input>{text}</user_input>`) and instruct the model in the system prompt that content inside those tags is untrusted; Bedrock provides no built-in prompt injection protection.

68. ALWAYS implement exponential backoff with jitter for Bedrock `ThrottlingException` and `ModelStreamErrorException` — Bedrock has per-model tokens-per-minute quotas; retry with `base=1.0, cap=60.0, jitter=uniform(0, delay*0.5)`, max 5 attempts; request quota increases before production launch.

69. ALWAYS use Bedrock Knowledge Bases for production RAG when document corpus is < 10 GB and you do not need custom chunking — Knowledge Bases manages embedding, OpenSearch Serverless vector storage, retrieval, and citation tracking; a custom pipeline with the same reliability takes weeks to build.

70. ALWAYS request Bedrock model access in the console before deploying code — models are not enabled by default; `AccessDeniedException` in production is avoidable.

71. ALWAYS store the Bedrock model ID as a configuration value — model IDs change with new versions; a config value updates without a code deploy.

72. ALWAYS use Bedrock Guardrails for production AI features handling regulated data — Guardrails provide topic denial, PII detection/masking, grounding checks, and content filtering at the Bedrock API layer; configuring them in CDK takes 20 lines and provides compliance-grade protection.

73. ALWAYS use Bedrock cross-region inference profiles for latency-critical workloads — cross-region profiles automatically route to the lowest-latency endpoint across configured regions; this is the April 2026 replacement for single-region endpoint routing.

---

## IaC and CDK (April 2026)

74. NEVER hardcode AWS account IDs or region strings in CDK stacks — use `self.account` and `self.region`, or pass as stack parameters; hardcoded values break when deploying to dev/staging/prod accounts.

75. ALWAYS use CDK L2/L3 constructs before falling back to L1 (`Cfn*`) — L2 constructs enforce security defaults, generate least-privilege policies via grant methods, and reduce code by 60-80%.

76. ALWAYS run `cdk diff` before `cdk deploy` in CI and fail the pipeline if the diff contains resource replacements on stateful resources (RDS, ElastiCache, DynamoDB) — replacement = destroy-and-recreate; CloudFormation will delete your database.

77. ALWAYS set `removal_policy=RemovalPolicy.RETAIN` on RDS clusters, DynamoDB tables, S3 buckets with user data, and ElastiCache clusters — CDK default is `DESTROY`; `cdk destroy` with default settings permanently deletes production data.

78. ALWAYS store Terraform state in S3 + DynamoDB locking — never commit local state files; never run `terraform apply` without a configured remote backend in team environments.

79. ALWAYS use separate AWS accounts (not workspaces) for production vs non-production — workspaces share the same account; `terraform destroy` in the wrong workspace destroys production.

80. ALWAYS tag every AWS resource with `Environment`, `Service`, `Owner`, and `CostCenter` — tags are the only way to allocate costs and identify owners during incident response; enforce with AWS Config rule `required-tags`.

81. ALWAYS use CDK Aspects for tag enforcement — `cdk.Tags.of(app).add("Environment", env_name)` at the app level rather than per-construct.

82. ALWAYS use AWS CDK v2 with the `App` construct and use CDK Pipelines (`pipelines.CodePipeline`) for multi-stage deployments — CDK v1 is EOL; CDK Pipelines provides self-mutating pipelines that update themselves before deploying application changes.

---

## MCP Servers for AWS (April 2026)

83. ALWAYS configure the `awslabs.core-mcp-server` and `awslabs.aws-documentation-mcp-server` in `.claude/settings.json` when building on AWS — these provide real-time AWS service documentation, CDK construct references, and CloudFormation schema lookups directly in the editor without leaving the terminal.

84. ALWAYS use `awslabs.cdk-mcp-server` for CDK construct exploration and `awslabs.cost-analysis-mcp-server` for pre-deploy cost estimation — the CDK MCP server resolves L2 construct APIs from the latest CDK version, preventing the most common mistake of using deprecated CDK v1 APIs.

85. ALWAYS use `awslabs.cloudwatch-logs-mcp-server` for production debugging sessions — it enables natural-language CloudWatch Logs Insights queries without writing JMESPath manually, and surfaces errors across log groups in a single query.

86. NEVER store AWS credentials in MCP server configuration — MCP servers inherit AWS credentials from the environment (instance profile, OIDC, or `~/.aws/config` SSO session); never add `AWS_ACCESS_KEY_ID` to the `env` field in `.claude/settings.json`.

---

## Observability

87. ALWAYS emit structured JSON logs to stdout in ECS and Lambda — CloudWatch Logs captures stdout automatically; JSON enables Logs Insights field queries without regex.

88. ALWAYS include `request_id`, `user_id`, `service`, `environment`, and `duration_ms` in every log line.

89. ALWAYS instrument FastAPI with AWS X-Ray using `aws-xray-sdk` and `patch_all()` — X-Ray shows exactly which downstream (RDS, Bedrock, S3, Redis) is the latency bottleneck.

90. ALWAYS set CloudWatch alarms on: ECS CPU > 80% for 3 consecutive minutes, ALB 5xx rate > 1% over 5 minutes, ALB P99 latency > 5000ms, SQS DLQ depth > 0.

91. ALWAYS enable CloudWatch Container Insights on every ECS cluster — adds per-task CPU/memory/network metrics; without it, you cannot identify which task is causing high CPU.

92. ALWAYS create a CloudWatch cost anomaly detector per production account — alert on > 20% week-over-week spend increase; Bedrock token spikes and NAT data charges are detectable before the monthly bill.

93. NEVER use 5-minute CloudWatch metric resolution for latency SLA alarms — use `period=Duration.minutes(1)`; a 5-minute period means a P99 spike goes undetected for up to 5 minutes.

94. ALWAYS set `retention=logs.RetentionDays.ONE_MONTH` on all CloudWatch log groups and archive to S3 Glacier for long-term compliance — CloudWatch at $0.03/GB/month vs S3 Glacier at $0.004/GB/month.

---

## Anti-Patterns

95. NEVER deploy an AI endpoint to Lambda that calls Bedrock synchronously with P99 > 10s — Lambda cold start (1-5s) + Bedrock (2-15s) + streaming = users receiving 504 from API Gateway before response arrives; use ECS Fargate for AI-serving endpoints.

96. NEVER store model weights, large embeddings (> 50 MB), or training datasets in a Git repository or Docker image — use S3 versioned keys and pull at container startup.

97. NEVER make direct DB connections from Lambda without RDS Proxy — the single most common Lambda DB scaling failure.

98. NEVER use a public ECR repository for production container images — push all images to private ECR with image scanning on push enabled (`scan_on_push=True`).

99. NEVER skip CloudTrail for production accounts — CloudTrail records every API call including `DeleteBucket`, `PutBucketPolicy`, and `GetSecretValue`; without it, security incident investigation is impossible.

100. NEVER create SQS queues, DynamoDB tables, or S3 buckets with environment suffixes as the only isolation mechanism (`my-queue-prod` vs `my-queue-staging`) in the same AWS account — use separate AWS accounts for production vs non-production.

101. NEVER use Lambda local ZIP handlers with Python dependencies in CDK — use `aws_lambda_python_alpha.PythonFunction` which builds a Docker-based asset bundle; hand-built ZIPs fail with `Module not found` for compiled extensions (numpy, cryptography, psycopg2-binary).

102. NEVER expose RDS, ElastiCache, or DynamoDB endpoints in public subnets — all database resources must be in private or isolated subnets with security groups restricting ingress to the application SG only.

103. NEVER inject `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` via the ECS task definition `environment` field — these appear in plaintext in the task definition and in CloudTrail `DescribeTaskDefinition` calls; use `secrets` field with `ecs.Secret.from_secrets_manager()`.

104. NEVER call `bedrock:InvokeModel` with `"Resource": "*"` in task role policies without a model ARN condition — scope to specific model ARNs (`arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-*`) to prevent accidental invocation of the most expensive models.

105. NEVER use AWS Lambda SnapStart for Python functions — SnapStart is Java-only as of April 2026; enabling it on a Python function silently falls back to standard cold start without error.

106. NEVER deploy to production without enabling AWS GuardDuty, Security Hub, and Config in every account and region — GuardDuty detects credential misuse and anomalous API activity in real time; Security Hub aggregates findings across services; Config tracks resource configuration drift; the combined cost is < $50/month for most workloads.

---

## AWS CLI v2 and SSO Workflow

107. ALWAYS authenticate with AWS CLI v2 via IAM Identity Center SSO — run `aws configure sso` once per profile, then `aws sso login --profile <name>` at session start; never run `aws configure` with static access keys on developer machines.

108. ALWAYS set `AWS_PROFILE` in your shell session after `aws sso login` — every subsequent CLI command and SDK call picks it up automatically; without it, commands silently fall back to the `[default]` profile.

109. ALWAYS use `--query` with JMESPath to extract fields from CLI output rather than piping to `jq` — `aws ecs list-tasks --query 'taskArns[0]' --output text` returns the raw string directly; JMESPath is built into the CLI and requires no extra tooling.

110. ALWAYS use `--output json` in CI scripts that parse CLI output programmatically — `text` and `table` output formats are for humans; they change formatting across CLI versions and break downstream `grep`/`awk`.

111. NEVER share a named profile across multiple roles or environments in `~/.aws/config` — create one profile per account-role pair (e.g. `[profile prod-admin]`, `[profile staging-deploy]`); a typo that invokes the wrong profile on a destructive command is unrecoverable.

---

## Bedrock April 2026 — Model IDs and API Patterns

112. ALWAYS use the `us.` cross-region inference profile prefix for latency-critical Bedrock calls in us-east-1 — model IDs as of April 2026: `us.anthropic.claude-sonnet-4-5-20251001-v1:0` (Sonnet 4.5), `us.anthropic.claude-haiku-4-5-20251001-v1:0` (Haiku 4.5); cross-region profiles absorb regional capacity shortfalls automatically.

113. ALWAYS use the Bedrock `ConverseStream` API for FastAPI streaming responses — pair with `StreamingResponse(media_type="text/event-stream")` and yield each `contentBlockDelta` chunk as an SSE event; `invoke_model_with_response_stream` requires model-specific JSON and is deprecated in favor of ConverseStream.

114. ALWAYS use Bedrock Prompt Caching by adding `{"cachePoint": {"type": "default"}}` as a content block after static system prompt content exceeding 1024 tokens — Bedrock prompt cache TTL is 5 minutes; hits reduce input token cost by 90% and latency by 60%; verify via `response["usage"]["cacheReadInputTokenCount"] > 0`.

115. ALWAYS use Bedrock Inline Agents API (`bedrock-agent-runtime:InvokeInlineAgent`) for one-shot agentic tasks rather than creating a named Bedrock Agent resource — Inline Agents accept tool definitions and session state per invocation with no pre-provisioned resource; named Agents add cold start and version management overhead for dynamic tool sets.

---

## ECS Fargate Graviton ARM64 CDK

116. ALWAYS specify `runtime_platform=ecs.RuntimePlatform(cpu_architecture=ecs.CpuArchitecture.ARM64, operating_system_family=ecs.OperatingSystemFamily.LINUX)` on every `FargateTaskDefinition` targeting Graviton — omitting it lets Fargate silently schedule on x86 instances; your ARM64 Docker image will fail to launch with `exec format error` if the wrong architecture is selected.

117. ALWAYS build Docker images for ARM64 in CI using `docker buildx build --platform linux/arm64` — building on an x86 CI runner without `buildx` produces an amd64 image that crashes on Graviton Fargate; use QEMU emulation (`--platform linux/arm64`) or an ARM64 CI runner.

118. ALWAYS set `cpu=512` and `memory_limit_mib=1024` as the minimum for FastAPI + asyncpg on ARM64 Fargate — the default 256 CPU / 512 MB causes OOM kills under Bedrock streaming response buffering; profile with Container Insights before tuning down.

---

## App Runner Specifics

119. ALWAYS configure App Runner health check with `path="/health"`, `interval=10`, `timeout=5`, `healthy_threshold=1`, `unhealthy_threshold=3` — the default health check uses TCP port check only; HTTP path checks catch application-level startup failures that a TCP check misses.

120. ALWAYS set App Runner `auto_scaling_configuration` with `min_size=1`, `max_size=10`, `max_concurrency=50` and tune `max_concurrency` based on your P99 latency target — App Runner scales by concurrent requests per instance; setting `max_concurrency` too high over-saturates the instance before scaling out.

121. ALWAYS use App Runner VPC Connector when the application needs to access RDS, ElastiCache, or internal services — without a VPC Connector, App Runner runs in AWS-managed infrastructure with no private network access; add the connector ARN via `CfnService.NetworkConfigurationProperty`.

---

## Lambda Powertools and EMF

122. ALWAYS use AWS Lambda Powertools for Python (`aws-lambda-powertools`) for Lambda structured logging, tracing, and custom metrics — use `@logger.inject_lambda_context` to auto-inject `request_id`, `function_name`, `cold_start`; use `@tracer.capture_lambda_handler` for X-Ray subsegments; use `@metrics.log_metrics` for EMF-format CloudWatch custom metrics.

123. ALWAYS emit custom metrics via Embedded Metric Format (EMF) using Powertools `Metrics` — EMF writes metric data as structured JSON to CloudWatch Logs, creating CloudWatch custom metrics with zero PutMetricData API calls; this is zero-cost at low volume vs PutMetricData at $0.01 per 1000 calls.

124. ALWAYS add `@tracer.capture_method` to every function that calls Bedrock, RDS Proxy, S3, or SQS inside a Lambda handler — Powertools creates an X-Ray subsegment for each call with latency and error metadata; without it, the X-Ray service map shows only the Lambda node, not the downstream services.

---

## SQS DLQ and Visibility Timeout

125. ALWAYS set SQS queue `visibility_timeout` to exactly 6× the Lambda function `timeout` when the queue is a Lambda event source — AWS documentation recommends 6× to account for batch processing time plus retries; a mismatch causes reprocessing: if Lambda timeout is 30s, set SQS visibility timeout to 180s.

126. ALWAYS create the DLQ in the same CDK stack as the source queue and output the DLQ URL as a CloudFormation export — DLQs created in separate stacks cause circular dependencies and complicate cross-account DLQ consumers.

---

## Secrets Manager Aurora Rotation

127. ALWAYS enable Secrets Manager automatic rotation for Aurora credentials using the `aws-secretsmanager-rotation-lambdas` rotation Lambda — set `rotation_days=30` maximum; enable `rotate_immediately_on_update=True`; the rotation Lambda updates both the Secrets Manager secret and the Aurora user password atomically, preventing credential drift.

128. ALWAYS use RDS Proxy IAM authentication (`iam_auth=True`) as the primary Aurora access pattern in Lambda and ECS — this eliminates DB password storage entirely; the application calls `generate_db_auth_token()` to get a 15-minute ephemeral password using the task's IAM role; Secrets Manager rotation is only needed for the master password used by RDS Proxy itself.

---

## CDK v2 Specific Patterns

129. ALWAYS use CDK context (`cdk.json` `context` key or `-c key=value` CLI flag) to parameterize environment-specific values (VPC CIDR, instance sizes, domain names) instead of environment variables read inside the CDK app — context values are recorded in `cdk.context.json` and reproduced deterministically; environment variables at synth time produce non-reproducible CloudFormation templates.

130. ALWAYS call `cdk bootstrap` with `--trust` and `--cloudformation-execution-policies` when setting up cross-account deployments — `cdk bootstrap` creates the CDKToolkit stack per account; without explicit trust, the pipeline role in the tools account cannot deploy to target accounts.

---

## Cost Optimization

131. ALWAYS purchase Compute Savings Plans covering your baseline ECS Fargate and Lambda usage after 2 weeks of stable production traffic — Compute Savings Plans apply to Fargate, Lambda, and EC2 with no instance-type commitment; 1-year no-upfront provides 17% discount on Fargate and 17% on Lambda; 3-year all-upfront reaches 37%.

132. ALWAYS use Graviton ARM64 for all Lambda functions and ECS tasks — ARM64 Lambda is 20% cheaper per GB-second and typically 10-19% faster for Python workloads; ARM64 Fargate is 20% cheaper per vCPU-hour and 10% cheaper per GB-hour than x86 Fargate.

---

## GuardDuty and Security Hub Integration

133. ALWAYS enable GuardDuty ECS Runtime Monitoring and Lambda Protection in addition to the base GuardDuty service — ECS Runtime Monitoring detects container escapes and crypto-mining in running tasks; Lambda Protection detects malicious code execution and data exfiltration from Lambda; both are disabled by default and require explicit activation.

134. ALWAYS configure a Security Hub → EventBridge → SNS → Slack integration for `CRITICAL` and `HIGH` severity findings with `WorkflowStatus=NEW` — Security Hub findings older than 24 hours without triage indicate an unmonitored security posture; automated alerting closes the detection-to-response gap.

---

## EventBridge Patterns

135. ALWAYS use EventBridge Scheduler instead of CloudWatch Events cron rules for scheduled tasks — Scheduler supports one-time schedules, flexible rate expressions, timezone-aware cron, and dead-letter queues for missed invocations; CloudWatch Events cron has no DLQ and fires in UTC only.

136. ALWAYS register event schemas in the EventBridge Schema Registry when your service publishes custom events — the Schema Registry generates typed Python bindings (`aws-eventbridge-schema-registry`); consuming services validate event structure at compile time instead of discovering shape mismatches at runtime.

---

## Valkey / ElastiCache Serverless

137. ALWAYS use ElastiCache Serverless (Valkey engine) for new ECS deployments when cache traffic has variable or unpredictable patterns — Serverless scales from zero to millions of operations/second with no cluster sizing; use Provisioned clusters only when you need predictable sub-millisecond P99 at sustained high throughput (> 100K ops/sec constant).

138. ALWAYS enable ElastiCache Global Datastore for Valkey when the application is multi-region — Global Datastore provides < 1 second replication lag across regions; read replicas in the secondary region serve local reads; use it for session replication and rate limiting counters that must be consistent across regions.

---

## IAM Identity Center CI/CD Patterns

139. ALWAYS use IAM Identity Center trusted token issuer OIDC configuration for GitHub Actions instead of static IAM users in member accounts — configure a trust policy with `token.actions.githubusercontent.com` as the OIDC provider and `aws:sub` condition scoped to `repo:<org>/<repo>:ref:refs/heads/main`; this makes the CI role assumption auditable in CloudTrail and revocable from the Identity Center console without rotating static keys.

---

## S3 Presigned URLs

140. ALWAYS use `generate_presigned_url(ClientMethod="get_object", ExpiresIn=900)` for download links and `generate_presigned_post(ExpiresIn=300)` for browser-direct uploads — PUT presigned URLs (`ClientMethod="put_object"`) are correct for server-side SDK uploads; `generate_presigned_post` is required for HTML form-based browser uploads where the browser cannot set `Content-Type` headers dynamically; mixing them produces `SignatureDoesNotMatch` errors that are difficult to diagnose.

141. NEVER set presigned URL expiry beyond 7 days (604800 seconds) — this is the AWS hard maximum for SigV4-signed presigned URLs; requests beyond this limit always return `RequestExpired`; for long-lived access use CloudFront signed URLs with a key group instead, which support arbitrary TTLs and can be revoked via the CloudFront key invalidation API.

142. ALWAYS generate presigned URLs on the server side using the task role or Lambda execution role — never generate them client-side or pass AWS credentials to a browser; the presigned URL itself is the capability token; generating it server-side ensures the operation is gated by your authorization layer before the URL is issued.

---

## Aurora Serverless v2

143. ALWAYS set `serverless_v2_scaling_configuration` with `min_capacity=0.5` and `max_capacity` equal to your expected peak load in ACU (1 ACU ≈ 2 GB RAM; 1 ACU handles ~50 simple queries/second) — Aurora Serverless v2 scales in 0.5 ACU increments within seconds; set `max_capacity` conservatively at first and raise it once you profile actual ACU usage in CloudWatch metric `ServerlessDatabaseCapacity`.

144. NEVER enable Aurora Serverless v2 pause (zero ACU scaling) for production databases — paused clusters take 15-30 seconds to resume on the first connection, causing request timeouts; pause is acceptable for dev/staging only; for production set `min_capacity >= 0.5` to keep at least one warm ACU.

145. ALWAYS size the connection pool with `pool_size = min(max_connections, acus * 90)` where `max_connections` for Aurora Serverless v2 is approximately `ACU * 90` — at 2 ACU that is 180 connections; set `asyncpg` or `sqlalchemy` pool `max_overflow=0` to prevent connection storms when ACUs scale down; the connection count drops with ACUs during scale-in and pooled connections beyond capacity are dropped.

---

## CloudWatch Application Signals and ADOT

146. ALWAYS instrument FastAPI services with ADOT (AWS Distro for OpenTelemetry) instead of the X-Ray SDK for new deployments in April 2026 — ADOT is the AWS-supported OTel distribution; it emits OTel-native traces that feed both CloudWatch Application Signals and X-Ray simultaneously; the legacy X-Ray SDK does not emit OTel spans and will not receive new features.

147. ALWAYS enable CloudWatch Application Signals on ECS services by adding the ADOT sidecar container (`public.ecr.aws/aws-observability/aws-otel-collector:latest`) and setting the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable to `http://localhost:4317` — Application Signals auto-discovers service dependencies, builds a service map, and computes SLO metrics (availability, latency) from OTel spans with no custom metric instrumentation; this replaces manual CloudWatch metric math for SLA dashboards.

148. ALWAYS define CloudWatch Application Signals SLOs in CDK using `CfnServiceLevelObjective` with a rolling 28-day evaluation window and `attainment_goal=99.9` for production API services — Application Signals SLOs generate automated burn-rate alarms that alert before an SLO breach, not after; the alarms fire at 14× and 6× burn rates to give 1-hour and 6-hour warning respectively.

---

## Lambda Function URLs vs API Gateway

149. ALWAYS use Lambda Function URLs (not API Gateway) for single-function backends with no routing, no authentication middleware, and no WAF requirement — Function URLs provide HTTPS with AWS_IAM or no-auth at zero additional cost; API Gateway adds $3.50/million requests and 10ms median latency overhead that is unjustified for simple function exposure.

150. ALWAYS use API Gateway HTTP API (not REST API) when you need request routing, JWT authorizers, CORS management, or rate limiting across multiple Lambda functions — HTTP API is 71% cheaper than REST API ($1.00 vs $3.50 per million) and has lower latency; only choose REST API when you specifically need API keys, usage plans, or request/response transformation.

151. NEVER use Lambda Function URLs for public-facing AI endpoints that accept user input — Function URLs have no WAF integration and no request throttling per IP; a single client can exhaust your Lambda concurrency; use API Gateway with a WAF WebACL and a rate-based rule limiting to 1000 requests/5 minutes per IP.

---

## ECS Service Connect vs App Mesh

152. ALWAYS use ECS Service Connect for service-to-service communication within an ECS cluster — Service Connect is built into ECS (GA April 2024), requires zero sidecar management, integrates with CloudWatch Container Insights for per-route latency metrics, and supports HTTP/1.1 and HTTP/2; App Mesh requires Envoy sidecar management, manual mTLS certificate rotation, and adds 2-4 seconds of pod startup time; App Mesh received no new features after 2023 and is in maintenance mode.

153. ALWAYS configure Service Connect with a `client_alias` port mapping and a `portName` that matches the container port — the `client_alias` provides a stable DNS name (`http://my-service:8080`) that survives task replacement; hardcoding the task ENI IP in inter-service calls breaks on every deployment.

---

## ECS Exec for Debugging

154. ALWAYS enable ECS Exec on ECS services in non-production environments by setting `enable_execute_command=True` on the `FargateService` CDK construct and attaching the `ssmmessages:CreateControlChannel`, `ssmmessages:CreateDataChannel`, `ssmmessages:OpenControlChannel`, `ssmmessages:OpenDataChannel` permissions to the task role — ECS Exec provides interactive shell access to running containers via AWS Systems Manager without a bastion host, SSH keys, or open inbound security group ports; it is the replacement for SSH-based debugging.

155. NEVER enable ECS Exec in production without restricting the `ecs:ExecuteCommand` IAM action to specific task ARNs and requiring MFA via `aws:MultiFactorAuthPresent: "true"` condition — interactive shell access to a production container is equivalent to SSH root access; gate it behind MFA and break-glass procedures logged in CloudTrail.

---

## KMS Key Types and Rotation

156. ALWAYS use customer-managed KMS keys (CMKs) instead of AWS-managed keys for S3 buckets, RDS clusters, and Secrets Manager secrets containing regulated data (PII, PHI, payment data) — CMKs allow key rotation, key policy customization, cross-account grant, CloudTrail logging of every `kms:Decrypt` call, and emergency key disabling; AWS-managed keys cannot be rotated on demand, cannot be disabled, and cannot be used cross-account.

157. ALWAYS enable automatic annual KMS CMK rotation (`enable_key_rotation=True` in CDK) and additionally perform manual rotation every 90 days for keys protecting PCI-DSS or HIPAA data — automatic rotation creates a new key material version annually while preserving the key ARN; data encrypted with old material is transparently decrypted; the CloudTrail event `RotateKey` confirms rotation occurred.

---

## NAT Gateway Cost Optimization

158. ALWAYS audit NAT Gateway data processing charges monthly using Cost Explorer grouped by `NAT Gateway` usage type — NAT Gateway costs $0.045/GB processed and $0.045/AZ-hour; an ECS service making frequent Bedrock API calls without a VPC Interface Endpoint can generate $500+/month in NAT charges alone; add VPC Interface Endpoints for Bedrock, S3, ECR, and Secrets Manager first, then measure the reduction before adding more endpoints.

---

## SQS vs Kinesis Decision

159. ALWAYS use SQS when: each message is consumed by exactly one worker, message order within a shard does not matter, processing is idempotent and retries are acceptable, and throughput is < 10,000 msg/sec — use Kinesis Data Streams when: multiple independent consumers must read the same records (fan-out), you need message replay within a 7-day window, you require strict ordering within a partition key, or your producer writes > 1 MB/sec per shard; choosing Kinesis for simple task queues adds shard management complexity with no benefit.

---

## DynamoDB Single-Table Design

160. ALWAYS design DynamoDB tables with a single-table pattern using a generic `PK`/`SK` composite key and entity-type prefixes (`USER#<id>`, `ORDER#<id>`) — single-table design allows heterogeneous item types to be queried in one round trip via `PK = "USER#123" AND SK begins_with "ORDER#"`; multi-table designs require N separate `GetItem` calls for N entity types, multiplying read latency and cost.

161. NEVER create more than 5 GSIs on a DynamoDB table — each GSI consumes additional write capacity units on every write (one WCU per GSI per item); a table with 10 GSIs multiplies write cost by 11×; redesign overloaded access patterns using sparse indexes (only items with the GSI attribute are indexed) or use a composite sort key that encodes multiple dimensions.

---

## IAM Permission Boundaries for Developer Self-Service

162. ALWAYS attach a permission boundary policy to every IAM role created by developers in a shared AWS account — the boundary policy caps the maximum permissions regardless of what the role's inline or managed policies grant; a boundary of `PowerUserAccess` minus `iam:*` prevents developers from creating roles that escalate their own privileges; enforce attachment via an SCP: `Deny iam:CreateRole` unless `iam:PermissionsBoundary` condition matches the org-standard boundary ARN.

---

## API Gateway vs ALB vs App Runner Built-in

163. ALWAYS use App Runner's built-in HTTPS termination (no extra cost) for single-service backends with < 10,000 req/min — App Runner includes HTTPS, custom domain, and auto-scaling with no ALB charge ($0.008/LCU-hour saved); add an ALB only when you need path-based routing across multiple target groups or WAF integration; add API Gateway only when you need Lambda integration, JWT authorizers, or usage plans.

---

## AWS Config and Compliance

164. ALWAYS enable AWS Config with the managed rules `restricted-ssh`, `restricted-common-ports`, `s3-bucket-public-read-prohibited`, `s3-bucket-server-side-encryption-enabled`, `rds-instance-public-access-check`, `access-keys-rotated`, and `mfa-enabled-for-iam-console-access` in every account — these seven rules cover the majority of SOC2 and HIPAA infrastructure controls; Config continuously evaluates resource configuration and marks non-compliant resources immediately, providing the audit evidence required for certification.

165. ALWAYS enable AWS Config conformance packs (`AWS-BestPracticesConformancePack` or the HIPAA/PCI-DSS managed packs) in production accounts rather than managing individual Config rules — conformance packs bundle 50-150 rules into a single deployable unit, output compliance summaries to S3, and satisfy auditor requests for a "controls inventory" without manual documentation.

---

## CloudTrail Configuration

166. ALWAYS enable CloudTrail with `include_global_service_events=True`, `is_multi_region_trail=True`, `enable_log_file_validation=True`, and S3 log delivery — a single-region trail misses IAM, STS, and Route53 API calls (global services); log file validation detects tampering via SHA-256 digest files; without multi-region coverage, an attacker operating in `ap-southeast-1` while your trail covers only `us-east-1` is invisible.

167. ALWAYS enable CloudTrail S3 data events (`DataResourceType=AWS::S3::Object`) for buckets containing user data, model weights, and secrets exports — management events cover S3 bucket-level API calls (`CreateBucket`, `DeleteBucket`) but not object-level reads and writes (`GetObject`, `PutObject`, `DeleteObject`); data events are essential for detecting unauthorized data exfiltration and satisfying HIPAA audit log requirements; enable selectively on high-value buckets to control cost ($0.10/100K events).

---

## ECR Image Scanning and Lifecycle

168. ALWAYS enable ECR enhanced scanning (AWS Inspector integration) on production repositories instead of basic scanning — enhanced scanning uses Inspector's continuously updated CVE database and scans images on push AND re-scans when new CVEs are published for previously-clean images; basic scanning runs only on push and uses a point-in-time vulnerability database; set `scan_on_push=True` and configure Inspector in CDK via `CfnRepository(image_scanning_configuration={"scanOnPush": True})`.

169. ALWAYS add ECR lifecycle policies to every repository to prevent unbounded storage cost — at minimum: keep the last 10 tagged images (`tagStatus=tagged, countType=imageCountMoreThan, countNumber=10`) and expire all untagged images older than 1 day (`tagStatus=untagged, countType=sinceImagePushed, countUnit=days, countNumber=1`); ECR charges $0.10/GB/month; a CI pipeline that pushes on every commit without a lifecycle policy accumulates hundreds of gigabytes within weeks.

---

## Bedrock Model Invocation Logging

170. ALWAYS enable Bedrock model invocation logging at the account level by configuring `bedrock:PutModelInvocationLoggingConfiguration` with `s3Config.bucketName` pointing to a versioned, KMS-encrypted S3 bucket — invocation logs capture the model ID, input token count, output token count, latency, and stop reason for every Bedrock API call; this is mandatory for SOC2 and HIPAA audit trails proving that only approved models were invoked and that no PII leaked through prompt inputs; logs are not stored by default.

171. NEVER log Bedrock invocation full request/response bodies (`s3Config.largeDataDeliveryS3Config`) in production when prompts may contain PII — enable metadata-only logging (`s3Config` with `invocationLogsConfig.usePromptData=false`) for HIPAA workloads; full body logging with PII in prompts requires the S3 bucket to be treated as a PHI data store with corresponding access controls, audit logging, and retention policies.

---

## Step Functions vs Lambda Orchestration

172. ALWAYS use AWS Step Functions Standard Workflows instead of a Lambda-calling-Lambda orchestration chain when the workflow has more than 3 sequential steps, requires human approval tasks, has branches based on business logic outcomes, or must resume after a failure — Lambda-calling-Lambda chains lose state on any failure with no built-in retry or resume; Step Functions persists state between steps, provides visual debugging in the console, and executes retry logic with exponential backoff natively; the $0.025/1000 state transitions cost is negligible compared to debugging a failed multi-step Lambda chain in production.

173. ALWAYS use Step Functions Express Workflows (not Standard) for high-volume event processing workflows (> 1000 executions/minute) that complete within 5 minutes — Express Workflows cost $1.00/million state transitions vs Standard at $25.00/million; Express Workflows are at-least-once execution with CloudWatch Logs as the audit trail; Standard Workflows are exactly-once with DynamoDB state persistence; use Standard for financial transactions and Express for log processing and AI pipeline fan-out.

---

## SNS Message Filtering

174. ALWAYS configure SNS subscription filter policies on Lambda subscriptions to route only matching messages — without filter policies, every SNS message invokes every subscribed Lambda regardless of relevance; a filter policy like `{"event_type": ["order.created", "order.updated"]}` prevents a high-volume `page.viewed` event stream from invoking an order-processing Lambda 10,000 times/minute at unnecessary cost; filter policies are evaluated at the SNS layer with no Lambda invocation for non-matching messages.

---

## ElastiCache Valkey Backup and High Availability

175. ALWAYS enable automatic daily snapshots on ElastiCache Valkey Provisioned clusters with `snapshot_retention_limit >= 7` days and configure a `snapshot_window` during off-peak hours — ElastiCache does not retain snapshots by default (`snapshot_retention_limit=0`); a cluster failure without snapshots means complete cache data loss; for Valkey Serverless, enable the automated backup feature in the cluster configuration.

176. ALWAYS enable `automatic_failover_enabled=True` and `multi_az_enabled=True` on ElastiCache Valkey Provisioned replication groups in production — without Multi-AZ, a primary node failure triggers a manual failover requiring operator action; automatic failover promotes a replica to primary within 1-2 minutes; multi-AZ places the replica in a different AZ so an AZ failure does not affect both primary and replica simultaneously.

---

## Aurora Blue/Green Deployments

177. ALWAYS use Aurora Blue/Green Deployments for major version upgrades and schema migrations in production as of April 2026 — Blue/Green creates a synchronized green replica, applies the upgrade or migration to the green cluster, runs validation, then performs a switchover with < 1 second of write downtime via DNS cutover; in-place major version upgrades require a maintenance window outage of 15-30 minutes; initiate Blue/Green via `rds:CreateBlueGreenDeployment` or CDK `CfnBlueGreenDeployment`; delete the blue cluster after validating the green cluster for 24 hours.

---

## SSM Session Manager

178. ALWAYS use AWS Systems Manager Session Manager instead of SSH or bastion hosts for interactive access to EC2 instances and ECS tasks — Session Manager requires no inbound security group ports (zero attack surface), logs every session to CloudWatch Logs and S3 for compliance, integrates with IAM for access control including MFA requirements, and works from the AWS Console or `aws ssm start-session` CLI; every bastion host is an additional attack surface, patching burden, and cost center that Session Manager eliminates entirely.

---

## API Gateway Usage Plans and Throttling

179. ALWAYS configure API Gateway usage plans with API keys for B2B and partner API access — usage plans define request quotas (per day/week/month) and throttle rates (requests/second and burst) per API key; without usage plans, a single misbehaving B2B client can exhaust your Lambda concurrency or Bedrock token quota affecting all clients; set account-level throttle as the hard ceiling and per-key throttle as the per-client limit.

180. ALWAYS set API Gateway stage-level default throttling (`default_route_throttle`) with `throttling_burst_limit` and `throttling_rate_limit` before public launch — without stage-level throttling, the account-level default (10,000 req/sec burst, 5,000 req/sec steady) applies; for AI-serving endpoints where each request costs $0.01-$0.10 in Bedrock tokens, an unthrottled stage allows a single DDoS to generate thousands of dollars in Bedrock charges within minutes.

---

## Lambda Layers for Shared Dependencies

181. ALWAYS publish shared Python dependencies (heavy packages: `boto3`, `pydantic`, `numpy`, `sqlalchemy`, `aws-lambda-powertools`) as Lambda Layers when the same packages are used across 3+ Lambda functions — layers reduce per-function deployment package size, enabling faster cold starts (Lambda downloads the function zip separately from layers; smaller zips initialize faster); update the layer version in CDK with `lambda.LayerVersion.from_layer_version_arn()` and pin the layer version ARN to avoid silent version drift across functions.

---

## Multi-Account Strategy

182. ALWAYS use AWS Organizations with a multi-account structure separating production, staging, development, security, and log archive accounts — a minimum viable structure has five accounts: management (Organizations root only), security (GuardDuty delegated admin, Security Hub aggregator, CloudTrail log archive), production, staging, and development; never deploy production workloads in the management account; never store CloudTrail logs in the same account they monitor (log tampering prevention).

183. ALWAYS enforce organization-wide guardrails using SCPs attached to the root OU or environment-specific OUs — critical SCPs: deny `cloudtrail:StopLogging`, deny `config:StopConfigurationRecorder`, deny `ec2:DisableEbsEncryptionByDefault`, deny creation of untagged resources, and deny resource creation outside approved regions; SCPs override even account-level administrator permissions and cannot be bypassed by account root users.

---

## ECS Task Metadata Endpoint v4

184. ALWAYS query the ECS task metadata endpoint v4 (`http://169.254.170.2/v4/task`) from within running containers to retrieve the task ARN, cluster name, container CPU/memory limits, and network configuration — the endpoint requires no credentials and is available in every Fargate task; use it at startup to inject the task ARN into structured log lines (`TASK_ARN = requests.get(os.environ["ECS_CONTAINER_METADATA_URI_V4"] + "/task").json()["TaskARN"]`), enabling CloudWatch Logs Insights queries that correlate log entries to specific task deployments; without the task ARN in logs, identifying which task revision produced an error requires guesswork.

---

## S3 Account-Level Block Public Access

185. ALWAYS enable S3 Block Public Access at the AWS account level in addition to per-bucket settings — account-level settings (`aws s3control put-public-access-block --account-id <id> --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true`) prevent any future bucket from being made public, even if a developer accidentally sets `BlockPublicAccess.BLOCK_ALL` to `false` at the bucket level; CDK does not apply account-level settings automatically; enforce via AWS Config rule `s3-account-level-public-access-blocks` and remediate with a Config auto-remediation SSM document.

---

## Security Group Egress Restrictions

186. NEVER leave Security Group egress rules as the CDK default `allow_all_outbound=True` on application security groups — `allow_all_outbound=True` creates an `0.0.0.0/0` egress rule permitting connections to any IP on any port; for ECS tasks and Lambda ENIs, restrict egress to: the database SG on port 5432, the ElastiCache SG on port 6379, VPC CIDR on port 443 for VPC endpoints, and the NAT gateway SG for external HTTPS only; a compromised container with unrestricted egress can exfiltrate data to any internet endpoint; set `allow_all_outbound=False` and add explicit egress rules using SG peer references, not CIDR ranges.

---

## AWS WAF v2

187. ALWAYS attach an AWS WAF v2 WebACL to every API Gateway stage and App Runner service that accepts public internet traffic — attach the AWS Managed Rules Common Rule Set (`AWSManagedRulesCommonRuleSet`) and the Amazon IP Reputation List rule group as the baseline; add a rate-based rule limiting to 1000 requests per 5-minute window per IP before any managed rules; WAF is not enabled by default on API Gateway or App Runner and must be associated explicitly via `wafv2:AssociateWebACL` or CDK `wafv2.CfnWebACLAssociation`; for App Runner the association scope is `REGIONAL`.

188. NEVER set WAF rule priority without explicitly ordering rules: rate-based rules FIRST (priority 0-9), IP reputation lists SECOND (priority 10-19), common rule sets THIRD (priority 20-99), custom rules LAST (priority 100+) — WAF evaluates rules in ascending priority order and stops at the first `BLOCK` action; placing a broad common rule set before your rate limiter means slow-rate attacks from known-clean IPs bypass the rate limit.

---

## VPC Flow Logs

189. ALWAYS enable VPC Flow Logs on every VPC with `traffic_type=ALL` and deliver to both CloudWatch Logs (for real-time alerting) and S3 (for long-term retention and Athena queries) — VPC Flow Logs record accepted and rejected traffic at the ENI level; rejected flows are the primary signal for misrouted traffic, security group misconfiguration, and network-level attack detection; flow logs are not enabled by default; in CDK use `vpc.add_flow_log("FlowLog", destination=ec2.FlowLogDestination.to_cloud_watch_logs(log_group))`.

190. ALWAYS use the custom flow log format including `${vpc-id} ${subnet-id} ${instance-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status} ${flow-direction} ${traffic-path}` — the default format omits `flow-direction` and `traffic-path` which are required to distinguish ingress vs egress traffic and whether traffic transited a NAT gateway, VPC endpoint, or internet gateway; without these fields, egress exfiltration analysis requires cross-referencing ENI IDs with routing tables.

---

## AWS Health Dashboard and EventBridge Health Events

191. ALWAYS subscribe to AWS Health events via EventBridge using a rule matching `{"source": ["aws.health"], "detail-type": ["AWS Health Event"]}` and route to an SNS topic — Health events provide advance notice of scheduled maintenance (RDS minor version upgrades, ECS infrastructure replacements), and real-time notification of service degradations affecting your specific resources; relying on the Health Dashboard console means discovering an impacting event hours after it started; configure the EventBridge rule in the `us-east-1` region because Health events are always delivered there regardless of affected region.

---

## ECS Capacity Providers

192. ALWAYS configure ECS clusters with a `CapacityProviderStrategy` using both `FARGATE` and `FARGATE_SPOT` capacity providers for worker tasks — set `FARGATE_SPOT` with `weight=4` and `base=0` and `FARGATE` with `weight=1` and `base=1`; this runs the first task on guaranteed Fargate and distributes additional tasks 80% on Spot and 20% on Fargate; FARGATE_SPOT tasks receive a 2-minute SIGTERM before interruption — implement a SIGTERM handler that finishes the current SQS message and checkpoints state; do not use FARGATE_SPOT for API-serving tasks where interruption causes dropped requests.

---

## CloudWatch Synthetics Canaries

193. ALWAYS create CloudWatch Synthetics canaries for every public API endpoint and critical user flow in production — a canary runs a headless script on a schedule (every 1-5 minutes) from multiple AWS regions, measures availability and latency, and triggers a CloudWatch Alarm on failure; canaries detect endpoint failures that do not generate backend errors (DNS failures, CDN misconfigurations, certificate expiry) which CloudWatch metrics from inside the VPC cannot observe; create canaries in CDK using `synthetics.Canary` with the `syn-nodejs-puppeteer-6.3` runtime and alert on `SuccessPercent < 100` over 2 consecutive periods.

---

## AWS Backup

194. ALWAYS use AWS Backup to manage RDS Aurora, DynamoDB, and EFS backup policies centrally — create a Backup Plan with a daily backup rule (`schedule_expression="cron(0 2 * * ? *)"`) with a `delete_after=Duration.days(35)` retention and a weekly copy rule that replicates to a separate region with `delete_after=Duration.days(365)`; AWS Backup provides a single audit dashboard for backup compliance across all resources, generates `BackupJobCompleted` EventBridge events for alerting on backup failures, and satisfies the SOC2 CC9.1 backup control; never rely solely on RDS automated backups which do not copy cross-region and cannot be centrally managed.

---

## CodeArtifact for Private Python Packages

195. ALWAYS use AWS CodeArtifact as the PyPI upstream proxy and private package repository for production Python projects — configure CodeArtifact with an upstream connection to `pypi.org` so all package installs route through CodeArtifact (`pip install --index-url https://<domain>-<account>.d.codeartifact.<region>.amazonaws.com/pypi/<repo>/simple/`); CodeArtifact caches all public packages, enabling air-gapped builds in private subnets without NAT egress to pypi.org, and blocks packages that match a deny list; publish internal shared libraries to the same CodeArtifact domain so Lambda Layers and ECS images reference a single authenticated registry; authenticate with `aws codeartifact get-authorization-token` (12-hour token) in CI.

---

## CDK Docker Asset Bundling

196. ALWAYS use `aws_lambda_python_alpha.PythonFunction` or a custom `aws_lambda.Code.from_asset()` with a `bundling` option using a Docker image matching the Lambda runtime for Python Lambda deployments — bundling installs dependencies inside a container matching the Lambda execution environment, ensuring compiled extensions (cryptography, numpy, psycopg2-binary) are built for the correct glibc version; hand-assembled ZIPs built on a macOS or Ubuntu developer machine produce `ImportError: /lib/x86_64-linux-gnu/libz.so.1: version GLIBC_2.29 not found` errors at Lambda cold start; use `BundlingOptions(image=lambda_.Runtime.PYTHON_3_12.bundling_image, command=["pip", "install", "-r", "requirements.txt", "-t", "/asset-output"])`.

---

## Bedrock Model Evaluation

197. ALWAYS use Bedrock Model Evaluation before switching to a new foundation model in production — Bedrock's built-in evaluation API (`bedrock:CreateEvaluationJob`) runs automated metrics (BERTScore, ROUGE, accuracy) against your golden dataset stored in S3 without requiring external evaluation infrastructure; submit an evaluation job with `evaluationConfig.automated.datasetMetricConfigs` specifying your task type and dataset S3 URI; compare `evaluationSummary.statistics` across model versions before updating the model ID config; model swaps without evaluation introduce silent quality regressions.

---

## Bedrock Prompt Management

198. ALWAYS use Bedrock Prompt Management (`bedrock:CreatePrompt`, `bedrock:CreatePromptVersion`) to version and deploy prompts in production instead of storing prompt text in application code or environment variables — create a prompt with variants for A/B testing, promote specific versions to a `production` alias via `bedrock:UpdatePrompt`, and fetch at runtime via `bedrock_agent:GetPrompt(promptIdentifier=arn, promptVersion="production")`; prompt changes deployed without version tracking make rollback impossible and audit trails incomplete; Bedrock Prompt Management integrates with model invocation logging so each logged invocation records the prompt ARN and version.

---

## SageMaker vs Bedrock Decision

199. NEVER deploy a custom fine-tuned or self-hosted model on SageMaker Real-Time Inference when Bedrock Custom Model Import or Bedrock Fine-Tuning satisfies the requirement — SageMaker endpoints require instance selection, auto-scaling configuration, model artifact management in S3, container image management in ECR, and VPC endpoint configuration; Bedrock Custom Model Import (GA April 2025) accepts Llama and Mistral fine-tuned weights directly, serves them via the standard Converse API, and scales to zero between requests; choose SageMaker only when the model architecture is not supported by Bedrock import, when you require GPU instance type control for latency SLAs, or when inference code customization beyond the standard inference spec is required.

---

## Titan Embeddings vs Cohere Embed on Bedrock

200. ALWAYS use `amazon.titan-embed-text-v2:0` (1536 dimensions, Matryoshka representation learning) as the default embedding model for new RAG applications on Bedrock in April 2026 — Titan Embeddings V2 supports dimension reduction to 256 or 512 without retraining (Matryoshka), reducing OpenSearch Serverless vector storage cost by 83% at 256 dimensions with < 5% MTEB benchmark degradation; use `cohere.embed-multilingual-v3` instead when the document corpus contains non-English content (100+ languages supported) or when the MTEB multilingual benchmark score is a contractual requirement; never mix embeddings generated by different models or dimension settings in the same vector index — similarity scores are not comparable across models.

---

## Transit Gateway vs VPC Peering

201. ALWAYS use Transit Gateway instead of VPC Peering when connecting more than 3 VPCs — VPC Peering is non-transitive: N VPCs require N*(N-1)/2 peering connections (10 VPCs = 45 peerings), each requiring manual route table entries in every subnet; Transit Gateway is a hub-and-spoke router where each VPC attaches once and routing is centrally managed via a Transit Gateway route table; at 3+ VPCs the operational cost of Peering exceeds the Transit Gateway attachment cost ($0.05/attachment/hour plus $0.02/GB processed); use VPC Peering only for simple two-VPC scenarios (application VPC to shared-services VPC) where the non-transitivity is acceptable and the $0.01/GB cross-AZ data charge matters at scale.

202. NEVER route traffic between a production VPC and a development VPC through Transit Gateway on the same attachment with no route table separation — Transit Gateway supports multiple route tables; create a `prod-rt` and a `dev-rt` with blackhole routes preventing prod-to-dev lateral movement; failing to isolate route tables means a compromised development account can reach production databases through the same Transit Gateway; enforce isolation via SCP denying `ec2:CreateTransitGatewayRoute` in member accounts (all route changes go through the network account).

---

## IPv6 Dual-Stack

203. ALWAYS enable IPv6 dual-stack on new VPCs and assign `/56` IPv6 CIDR blocks to the VPC and `/64` blocks to each subnet — AWS provides IPv6 prefixes from its own pool at no charge; ALBs, ECS tasks with `assignPublicIpv6=true`, and Lambda ENIs support IPv6 natively in April 2026; enabling IPv6 at VPC creation costs nothing and avoids IPv4 address exhaustion as AWS increases the charge for public IPv4 addresses ($0.005/hour per address as of February 2024); set `ipv6_cidr_block=True` on the CDK `Vpc` construct and `map_public_ip_on_launch=False` while setting `assign_ipv6_address_on_creation=True` on public subnets.

204. NEVER disable `enableDnsSupport` or `enableDnsHostnames` on any VPC that uses VPC Interface Endpoints — VPC Interface Endpoints create private DNS entries that override the public service endpoint DNS names (e.g. `bedrock-runtime.us-east-1.amazonaws.com` resolves to the endpoint ENI IP instead of the public endpoint); if `enableDnsSupport=false`, Route53 Resolver cannot respond inside the VPC and endpoint DNS override fails silently, causing traffic to route through NAT to the public endpoint; if `enableDnsHostnames=false`, EC2 instances do not receive hostnames required for RDS endpoint resolution; both attributes must be `true` on every VPC that has Interface Endpoints — verify with `aws ec2 describe-vpc-attribute --attribute enableDnsSupport`.

---

## PrivateLink for Cross-Account Service Sharing

205. ALWAYS use AWS PrivateLink (VPC Endpoint Services) to expose internal microservices across VPC or account boundaries instead of VPC Peering or Transit Gateway when the consumer needs access to a single service endpoint — PrivateLink creates a one-directional, service-specific connection: the consumer's VPC sends traffic to a private IP in their own VPC backed by an NLB in the provider's VPC; unlike Peering or Transit Gateway, PrivateLink never gives the consumer network-level access to the provider VPC CIDR, preventing lateral movement; create the endpoint service with `ec2.VpcEndpointService(nlb=nlb, acceptance_required=True)` and require explicit approval of each consumer account's connection request; use PrivateLink for shared internal APIs, data plane services, and SaaS-style platform services consumed by multiple product teams.

---

## RDS Parameter Groups and Aurora Storage

206. ALWAYS create a custom RDS parameter group for Aurora PostgreSQL clusters instead of using the default parameter group — the default parameter group cannot be modified; custom parameter groups allow tuning `shared_buffers` (default 25% of RAM; increase to 40% for read-heavy workloads), `work_mem` (default 4 MB; increase to 64 MB for complex queries with sorts and hash joins), `max_connections` (default formula `LEAST({DBInstanceClassMemory/9531392}, 5000)`), `log_min_duration_statement` (set to 1000ms in production to capture slow queries), and `pg_stat_statements` (add to `shared_preload_libraries` for query-level analytics); create via CDK `rds.ParameterGroup` and reference in `AuroraCluster`; parameter group changes on running clusters require either a reboot (static parameters) or apply immediately (dynamic parameters) — check `ApplyType` before applying in production.

207. NEVER assume Aurora Serverless v2 storage scales without limit — Aurora Serverless v2 storage scales automatically up to 128 TiB per cluster with no action required; however, storage never automatically scales down after deletion; run `VACUUM` and monitor `FreeStorageSpace` CloudWatch metric; set a CloudWatch alarm at 80% of your projected maximum storage to catch runaway table bloat before hitting the ceiling; storage is billed at $0.10/GB-month above the minimum 10 GB; for clusters with large delete-heavy workloads, schedule a weekly `VACUUM FULL` during maintenance windows and monitor `n_dead_tup` via `pg_stat_user_tables`.

---

## DynamoDB TTL Implementation

208. ALWAYS implement DynamoDB TTL for session data, idempotency keys, rate limit windows, and temporary records by adding a `ttl` attribute (Unix epoch integer) to items and enabling TTL on that attribute via `table.enable_time_to_live_attribute("ttl")` — DynamoDB deletes expired items within 48 hours of the expiry epoch at no additional cost (no WCU consumed); TTL deletion is eventually consistent: items past their epoch may still be returned in queries for up to 48 hours; add a filter expression `FilterExpression=Attr("ttl").gt(int(time.time()))` to any `Query` or `Scan` that must exclude logically-expired items immediately; never use DynamoDB TTL for hard-deadline security token revocation — use a Redis sorted set with exact expiry instead.

---

## ElastiCache Connection Pool Sizing

209. ALWAYS size the `redis-py` connection pool with `ConnectionPool(max_connections=N)` where `N = (task_cpu_units / 1024) * 10` for ECS Fargate tasks — a 1-vCPU Fargate task should hold no more than 10 Redis connections; each asyncio coroutine that awaits a Redis command blocks one connection; with `max_connections` unset (default unlimited), a traffic spike spawns thousands of connections that exhaust ElastiCache's per-node connection limit (65,000 for Valkey, but effective throughput degrades above 10,000 connections); for async code use `aioredis.ConnectionPool` with the same formula; set `socket_connect_timeout=0.5` and `socket_timeout=1.0` to fail fast on ElastiCache node failover rather than blocking event loop coroutines indefinitely.

---

## pgvector HNSW on Aurora

210. ALWAYS create pgvector HNSW indexes with `CREATE INDEX CONCURRENTLY` and tune `m` and `ef_construction` before production load — the default IVFFlat index in pgvector is deprecated in favor of HNSW for Aurora PostgreSQL 15.5+ with pgvector 0.7+; create with `CREATE INDEX CONCURRENTLY ON items USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=64)`; `m` controls graph connectivity (higher = better recall, more memory; 16 is the production default), `ef_construction` controls build-time search depth (higher = better recall, slower build; 64 is the minimum for 95%+ recall); set `SET hnsw.ef_search = 40` per session or per query for query-time recall tuning; HNSW indexes are 2-3× larger than IVFFlat but deliver sub-millisecond query times at 95%+ recall vs IVFFlat's variable recall; never build HNSW indexes without `CONCURRENTLY` — non-concurrent HNSW index creation on a large embedding table holds an exclusive table lock for minutes.

---

## AWS Budgets Programmatic Alerts

211. ALWAYS create AWS Budgets with SNS notification actions (not just email) for every production account — email-only budget alerts are unreliable for on-call response; configure `budgets:CreateBudget` with `NotificationsWithSubscribers` using `SubscriptionType=SNS` pointing to an SNS topic that fans out to PagerDuty or Slack; create three budgets per account: (1) monthly cost budget at 80% and 100% of expected spend with `ACTUAL` threshold type for spend alerts, (2) monthly cost budget at 120% with `FORECASTED` threshold type for early warning before month-end, (3) Bedrock service-specific budget at 150% of baseline to catch token spike incidents independently; in CDK use `budgets.CfnBudget` — there is no L2 construct as of April 2026; budgets are a separate service from Cost Anomaly Detector (rule 92) which detects percentage-based anomalies; budgets enforce absolute dollar thresholds.

---

## Cost Allocation Tags

212. ALWAYS activate Cost Allocation Tags in the AWS Billing console for every custom tag key used in your tagging strategy (`Environment`, `Service`, `Owner`, `CostCenter`) — resource tags are applied to AWS resources but are NOT automatically available in Cost Explorer until you activate them as cost allocation tags in `Billing → Cost allocation tags → Activate`; activation applies only to costs from the activation date forward (not retroactively); active cost allocation tags appear as filterable dimensions in Cost Explorer, Budgets, and Cost and Usage Reports; without activated tags, Cost Explorer shows only service-level spend with no way to split costs between teams, features, or environments; enforce the tag set on all resources via AWS Config rule `required-tags` (rule 80) but treat tag activation as a separate billing-plane operation required within the first week of account setup.

---

## Lambda Power Tuning

213. ALWAYS run the AWS Lambda Power Tuning tool (open-source Step Functions state machine, deploy via AWS SAR: `arn:aws:serverlessrepo:us-east-1:451282441545:applications/aws-lambda-power-tuning`) against every Lambda function before production launch and after any significant dependency change — invoke the state machine with `{"lambdaARN": "<arn>", "powerValues": [128, 256, 512, 1024, 2048, 3008], "num": 10, "payload": "<representative_event>", "parallelInvocation": true, "strategy": "cost"}` to find the memory setting that minimizes cost; for latency-critical functions use `"strategy": "speed"`; Python FastAPI handlers typically find the cost-optimal setting between 512-1024 MB (rule 23 identifies this range empirically); the tool generates a visualization URL showing cost vs latency tradeoff at each memory setting; re-run after adding new AWS SDK calls (Bedrock, RDS) as these change the cost-optimal memory point by changing the I/O-bound proportion of execution time.

---

## Bedrock Agents Code Interpreter (April 2026)

214. ALWAYS use Bedrock Agents Code Interpreter for agentic workflows that require data analysis, chart generation, or dynamic computation — Code Interpreter provides a sandboxed Python execution environment accessible to the agent via a built-in tool; enable it in CDK via `bedrock.CfnAgent` with `actionGroups: [{actionGroupName: "UserInputAction", parentActionGroupSignature: "AMAZON.CodeInterpreter"}]`; Code Interpreter runs in an isolated AWS-managed sandbox (no internet, no VPC access, no persistent state between turns) with numpy, pandas, matplotlib, scipy, and scikit-learn pre-installed; pass data files as S3 URIs in the session state and retrieve generated charts as base64-encoded artifacts in `returnControl` response events; never use Code Interpreter for workflows requiring database access or secrets — the sandbox has no network egress; cost is $0.0008 per code execution block in addition to standard model token costs.

---

## S3 Intelligent-Tiering

215. ALWAYS enable S3 Intelligent-Tiering for S3 buckets with unpredictable or mixed access patterns exceeding 128 KB average object size — Intelligent-Tiering monitors access patterns per object and automatically moves objects between Frequent Access, Infrequent Access (40% cheaper after 30 days), Archive Instant Access (68% cheaper after 90 days), and Archive Access (95% cheaper after 180 days) tiers; there is a per-object monitoring fee of $0.0025 per 1000 objects per month — this fee makes Intelligent-Tiering cost-effective only for objects > 128 KB (smaller objects cost more in monitoring fees than they save in storage); enable via lifecycle rule `Transition` to `INTELLIGENT_TIERING` storage class at day 0 in CDK `s3.Bucket.add_lifecycle_rule()`; rule 33 lists it as an option for unpredictable patterns but does not cover the 128 KB threshold or the optional deep archive tiers that must be explicitly activated via `IntelligentTieringConfiguration` with `optionalTiers`.

---

## Mangum — FastAPI on Lambda

216. ALWAYS wrap a FastAPI application with `Mangum` when deploying it behind API Gateway or a Lambda Function URL — `handler = Mangum(app, lifespan="off")` is the Lambda entry point; set `lifespan="off"` because Lambda containers are not long-lived processes and FastAPI lifespan startup/shutdown events fire on every cold start, not on process start; use `lifespan="auto"` only if you explicitly need startup initialization (e.g. opening a DB connection pool) and have profiled that the overhead is acceptable in your cold start budget; install with `pip install mangum` and set it as the Lambda `handler` in CDK `lambda_.Function(handler="app.handler")` where `app.py` exports the `handler` variable.

217. NEVER deploy a FastAPI application via Mangum to Lambda with synchronous SQLAlchemy or `psycopg2` blocking the event loop — Lambda's single-threaded execution means a blocking DB call stalls all other concurrent requests that Lambda routes to that container; use `asyncpg` with `sqlalchemy[asyncio]` and the async engine; if the codebase is synchronous and refactoring is not feasible, deploy to ECS Fargate instead of Lambda.

---

## FastAPI Lifespan for boto3 Client Initialization

218. ALWAYS initialize boto3 clients and connection pools inside the FastAPI `lifespan` context manager (not at module import time) for ECS Fargate deployments — clients created at import time capture IAM credentials at image build time, not at container start; task role credentials are only available after the ECS agent injects them into IMDS; create clients inside `@asynccontextmanager async def lifespan(app): yield` and store on `app.state`; example: `app.state.bedrock = boto3.client("bedrock-runtime", region_name=settings.aws_region)` inside the lifespan yields; this pattern also allows graceful shutdown cleanup (flushing metrics, closing DB pools) in the post-yield block.

---

## pydantic-settings with SSM Parameter Store

219. ALWAYS use SSM Parameter Store `String` type (not `SecureString`) for non-secret application configuration that varies by environment (feature flags, model IDs, endpoint URLs, timeout values) — `SecureString` uses KMS encryption which adds 1-2ms latency per `GetParameter` call and requires `kms:Decrypt` IAM permission; use Secrets Manager (rule 14) for credentials; use SSM `String` for everything else; fetch SSM parameters at startup via `ssm.get_parameters_by_path(Path="/myapp/prod/", WithDecryption=False)` and load into pydantic-settings using a custom `settings_customise_sources` that calls SSM before environment variables; cache SSM values in-process with a 60-second TTL to avoid per-request SSM API calls ($0.05 per 10,000 standard parameter requests).

220. NEVER store database passwords, API keys, or OAuth client secrets in SSM Parameter Store `String` type — `String` parameters are stored unencrypted and readable by anyone with `ssm:GetParameter` on the resource; use Secrets Manager for all credential storage (rule 14); the distinction is: SSM `String` = config, SSM `SecureString` = acceptable for secrets only in cost-sensitive non-production environments, Secrets Manager = mandatory for production credentials requiring rotation.

---

## Lambda Concurrency — Reserved vs Provisioned

221. ALWAYS set Lambda reserved concurrency on AI-serving functions to cap the maximum simultaneous Bedrock calls — without reserved concurrency, a traffic spike spawns unlimited Lambda instances each calling Bedrock, exhausting your per-account Bedrock tokens-per-minute quota and generating runaway charges; set `reserved_concurrent_executions` in CDK `lambda_.Function` to `(bedrock_tpm_quota / avg_input_tokens_per_request / avg_duration_seconds)`; this deliberately throttles Lambda (returning 429 to the caller) rather than throttling Bedrock (causing latency spikes inside running invocations); set reserved concurrency to `0` on functions that should never execute in production until explicitly enabled (canary, admin, migration functions).

222. ALWAYS use provisioned concurrency for Lambda functions that are the critical path of a synchronous user-facing API and have cold start P99 > 500ms — provisioned concurrency pre-warms the specified number of execution environments, eliminating cold starts for those invocations; cost is $0.000004646/GB-second provisioned + $0.000009646/GB-second invocation vs $0.0000166667/GB-second standard (you pay for both provisioned idle time and per-request time); calculate the break-even: if your function receives > 1 request every 40 seconds on average, provisioned concurrency is cost-neutral or cheaper than standard; use Application Auto Scaling (`lambda_.ScalableTarget`) to scale provisioned concurrency on a schedule to match traffic patterns rather than over-provisioning 24/7.

---

## S3 Bucket Naming and SSL Constraints

223. NEVER use dots (`.`) in S3 bucket names intended for HTTPS virtual-hosted-style access — bucket names with dots produce SSL certificate warnings because AWS's wildcard certificate `*.s3.amazonaws.com` does not match multi-level subdomains like `my.bucket.name.s3.amazonaws.com`; dots are valid for path-style access (deprecated as of 2023) but path-style is being phased out; use hyphens instead of dots in all bucket names; additionally, bucket names must be globally unique across all AWS accounts, 3-63 characters, lowercase alphanumeric and hyphens only, must not start with `xn--`, `sthree-`, or end with `-s3alias`; in CDK, omit the `bucket_name` parameter to let CloudFormation generate a unique name, or use a name prefix including account ID and region to guarantee uniqueness: `f"{app_name}-{account}-{region}"`.

---

## Aurora DNS Failover and Connection Handling

224. ALWAYS use the Aurora cluster writer endpoint (not individual instance endpoints) in your SQLAlchemy connection string and set the asyncpg `ssl="require"` parameter — Aurora's cluster endpoint DNS record automatically updates to the new writer instance within 30 seconds of a failover; however, connection pools cache DNS at the OS level; configure the Aurora cluster parameter `rds.force_ssl=1` and set `pool_pre_ping=True` in SQLAlchemy to detect stale connections post-failover; additionally set `connect_args={"server_settings": {"application_name": service_name}, "statement_cache_size": 0}` — `statement_cache_size=0` prevents asyncpg's prepared statement cache from causing `prepared statement does not exist` errors after a failover reconnects to a new Aurora instance that has no knowledge of the prior connection's prepared statements.

225. NEVER set `pool_recycle` below 1800 seconds (30 minutes) for Aurora Serverless v2 connections — Aurora Serverless v2 can scale down ACUs during low-traffic periods and may close idle connections; a `pool_recycle` of 1800s ensures connections are recycled before Aurora closes them server-side; combine with `pool_pre_ping=True` as a defense-in-depth; never set `pool_recycle` to match the Aurora `idle_in_transaction_session_timeout` exactly — set it 120 seconds shorter to avoid a race condition where the pool recycles a connection the instant before Aurora would time it out, causing a missed pre-ping.

---

## LocalStack for Local AWS Development

226. ALWAYS use LocalStack for local development and unit testing of AWS service integrations (S3, SQS, DynamoDB, Secrets Manager, SSM) — configure boto3 clients with `endpoint_url=os.environ.get("AWS_ENDPOINT_URL", None)` so the same code points to LocalStack in development and real AWS in production; set `AWS_ENDPOINT_URL=http://localhost:4566` in the local `.env` file and leave it unset in production; add LocalStack to `docker-compose.yml` with `services.localstack.image=localstack/localstack:latest` and `SERVICES=s3,sqs,dynamodb,secretsmanager,ssm`; never mock boto3 with `unittest.mock.patch` for integration tests — mocked clients cannot catch schema errors, missing resources, or IAM permission misconfigurations that LocalStack exposes.

---

## CDK Stack Deletion Protection and Testing

227. ALWAYS set `termination_protection=True` on every CDK production stack — CDK maps this to CloudFormation `TerminationProtection`; a `cdk destroy` or console delete on a protected stack fails with an error rather than deleting all resources including RDS clusters and S3 buckets; set it in the `Stack` constructor: `super().__init__(scope, id, termination_protection=(env_name == "prod"), **kwargs)`; additionally write CDK assertion tests using `aws_cdk.assertions.Template.from_stack(stack)` to verify that `termination_protection` is enabled, `removal_policy=RETAIN` is set on stateful resources, and no security groups have `0.0.0.0/0` ingress on non-443 ports; CDK assertions run in `pytest` with no AWS credentials required and catch infrastructure regressions before `cdk synth`.
