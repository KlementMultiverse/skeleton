---
description: GCP cloud architecture rules for AI-native FastAPI backends (April 2026). 205 rules.
paths: ["*.py", "*.tf", "*.yaml", "*.json"]
---

# GCP Patterns — AI-Native Backend (April 2026)

> Companion wiki: `~/knowledge/wiki/backend/gcp-complete.md`
> These rules apply to any project using GCP services.

---

## Project and Resource Hierarchy

1. ALWAYS create separate GCP projects per environment (prod, staging, dev) — treat GCP projects as the unit of isolation, the same way AWS accounts are. Sharing a project across environments mixes billing, IAM, quotas, and audit logs.

2. ALWAYS use folders to group related projects (production/, non-production/, shared/) — folder-level IAM policies cascade to all child projects, enabling org-wide controls without per-project repetition.

3. ALWAYS link projects to billing accounts by business unit or cost center — a single billing account for all projects makes cost attribution impossible. Separate billing accounts make chargeback tractable.

4. NEVER grant `roles/editor` or `roles/owner` to service accounts — they are overly broad. Grant the minimum predefined role that covers the service's actual needs.

5. ALWAYS prefer predefined roles over custom roles — only create custom roles when a predefined role is too broad and you can enumerate the exact permissions needed.

6. NEVER grant project-level IAM to individual user emails except for break-glass scenarios — bind users to groups and grant IAM to groups. Individual bindings create unmanageable sprawl.

---

## Service Accounts

7. NEVER download service account key JSON files — use Workload Identity Federation for CI/CD systems, and ADC (Application Default Credentials) for local development. Key files are long-lived credentials that can be exfiltrated silently.

8. ALWAYS enforce `constraints/iam.disableServiceAccountKeyCreation` as an Organization Policy — this blocks key downloads org-wide even if a developer tries to circumvent rule 7.

9. ALWAYS create a dedicated service account per Cloud Run service — never share service accounts across services. One SA per service enforces least privilege and makes audit trails legible.

10. NEVER reuse the Compute Engine default service account — it has broad project-level editor permissions. Always create purpose-built SAs with minimal roles.

11. ALWAYS use service account impersonation for human developers who need SA-level access during testing — `gcloud auth application-default login --impersonate-service-account SA@PROJECT.iam.gserviceaccount.com`. No key file, short-lived token.

---

## gcloud CLI and ADC

12. ALWAYS use `gcloud auth application-default login` for local development — this sets ADC to your user credentials so all `google-cloud-*` libraries authenticate without GOOGLE_APPLICATION_CREDENTIALS.

13. NEVER set GOOGLE_APPLICATION_CREDENTIALS to a key file path in developer machines — this defeats the purpose of ADC and creates hidden credential dependencies.

14. ALWAYS use named gcloud configurations for multi-project work — `gcloud config configurations create prod && gcloud config configurations activate prod`. Prevents accidentally running commands against the wrong project.

15. ALWAYS use `--format json` when parsing gcloud output in scripts — table format changes between versions; JSON is stable.

16. ALWAYS use `--filter` instead of piping to grep on gcloud commands — `gcloud run services list --filter "region:us-central1"` is faster (server-side) and more reliable than post-processing.

---

## Cloud Run — Runtime

17. ALWAYS use Cloud Run Gen 2 runtime for new services — specify `--execution-environment gen2` or set it in Terraform. Gen 2 has full Linux kernel support, larger instance sizes, and better networking.

18. NEVER use Cloud Run Gen 1 for new deployments — Gen 1 lacks background thread support, has simulated networking, and will be deprecated.

19. ALWAYS set `--min-instances 1` for production services — cold starts for Python FastAPI containers are 2-4 seconds. At least one warm instance eliminates cold start latency for real users.

20. ALWAYS set `--cpu-boost` (startup CPU boost) on Cloud Run services — this allocates extra CPU during instance startup, reducing cold start time by 30-50% with no change to steady-state cost.

21. ALWAYS configure an explicit `--max-instances` limit — without it, traffic spikes can exhaust downstream databases and APIs. Set max-instances based on your AlloyDB max_connections divided by your pool size.

22. NEVER set concurrency above your async event loop capacity for Python asyncio services — for FastAPI with asyncio, 80-150 is a safe concurrency range. Exceeding it causes memory pressure without latency benefit.

23. ALWAYS set always-on CPU (`--cpu-throttling=false`) for Cloud Run services that use background threads, WebSockets, or Pub/Sub pull consumers — CPU-throttled instances freeze between requests, breaking these patterns.

24. ALWAYS configure startup probes in Terraform/Cloud Run config — a startup probe hitting `/health` prevents Cloud Run from sending traffic to instances that haven't finished loading ML models or establishing DB connections.

---

## Cloud Run — Networking

25. ALWAYS attach a VPC connector to any Cloud Run service that accesses private resources — AlloyDB, Memorystore Valkey, and private GCE instances require the Serverless VPC Access connector to be reachable.

26. ALWAYS use `--vpc-egress private-ranges-only` unless you need to route all outbound traffic through a Cloud NAT gateway — routing only RFC1918 traffic through the VPC preserves internet access without the NAT cost for most workloads.

27. NEVER put Cloud Run service URLs directly in other services as static config — use Cloud Run's service name discovery or load balancer URLs. Direct revision URLs break on every deployment.

28. ALWAYS place Cloud Run services behind the Global External HTTPS Load Balancer for multi-region deployments — the load balancer handles anycast routing, SSL termination, and Cloud Armor WAF attachment.

---

## Cloud Run — Deployment Patterns

29. ALWAYS use traffic splitting for zero-downtime deployments — deploy new revision with `--no-traffic`, validate it at the canary tag URL, then shift traffic incrementally before cutting over fully.

30. NEVER use `--allow-unauthenticated` for internal services — only public-facing APIs need unauthenticated invocations. Internal service-to-service calls should use OIDC tokens via `--service-account` + invoker binding.

31. ALWAYS set environment variables via `--set-secrets` referencing Secret Manager rather than `--set-env-vars` for sensitive values — env vars with literal values appear in Cloud Run service config in plain text.

32. ALWAYS tag container images with git SHA — `gcr.io/PROJECT/IMAGE:$(git rev-parse --short HEAD)` instead of `:latest`. Tags are immutable; `:latest` is not, making rollbacks ambiguous.

---

## Cloud Run Jobs

33. ALWAYS use Cloud Run Jobs for database migrations, data exports, and batch processing — jobs run to completion and exit; they do not maintain an HTTP endpoint.

34. NEVER run database migrations inside a Cloud Run Service's startup code — migrations can take minutes and block health checks, causing the service to be killed and restarted in a loop. Use a Cloud Run Job executed before the service deploy.

---

## Database Selection

35. ALWAYS use AlloyDB for PostgreSQL for new AI/RAG workloads — AlloyDB AI provides built-in pgvector support with SCANN indexing, which outperforms HNSW at datasets > 1M vectors. Cloud SQL lacks this built-in capability.

36. NEVER use Cloud SQL for new workloads that require vector similarity search at production scale — Cloud SQL supports pgvector but lacks the SCANN index type and columnar engine that AlloyDB provides.

37. ALWAYS use Cloud SQL for PostgreSQL when AlloyDB's overhead is unjustified — smaller datasets (< 100GB), no vector/AI needs, or cost-sensitive greenfield projects where AlloyDB's 2x price premium isn't warranted.

38. NEVER use Firestore in Datastore mode for new projects — Datastore mode is deprecated as of 2024. Use Firestore Native mode exclusively.

39. ALWAYS use Firestore Native mode for document-model workloads, mobile offline sync, and real-time listener use cases — these are Firestore's strengths over AlloyDB.

40. NEVER use Firestore for relational data with complex joins — Firestore is a document database. Relational queries require denormalization or multiple round trips. Use AlloyDB instead.

41. ALWAYS use BigQuery for analytics queries, ML workflows, and long-term trace/log retention — BigQuery's serverless model eliminates capacity planning for infrequent but large analytical queries.

42. ALWAYS use Memorystore for Valkey as the caching and session store — Valkey replaced Redis OSS in Google Cloud Memorystore. The API is identical; the same client libraries work unchanged.

43. NEVER deploy a self-managed Redis or Valkey on GCE — use Memorystore. Self-managed Redis on VMs requires manual patching, failover configuration, and backup setup.

---

## AlloyDB

44. ALWAYS connect to AlloyDB via the AlloyDB Auth Proxy connector (`google-cloud-alloydb-connector`) — direct IP connections to AlloyDB require IP allowlisting and client certificates. The connector handles mTLS automatically using ADC.

45. ALWAYS use the async connector (`AsyncConnector`) with asyncpg for AlloyDB in FastAPI services — the sync connector blocks the event loop.

46. ALWAYS enable the columnar engine on AlloyDB instances used for analytics or hybrid OLTP/OLAP — `database_flags = { "google_columnar_engine.enabled" = "on" }` in Terraform. It accelerates aggregation queries 10-100x with no application changes.

47. ALWAYS use SCANN index instead of HNSW for vector columns in AlloyDB at datasets > 500K vectors — SCANN builds 10x faster and queries 3x faster than HNSW at large scale.

48. ALWAYS set `num_leaves` on the SCANN index based on dataset size — a rough guide: `num_leaves = sqrt(row_count)`, minimum 50, maximum 5000. Too few leaves reduces accuracy; too many slows index builds.

49. NEVER issue AlloyDB connection setup inside request handlers — create the connection pool once in the FastAPI lifespan startup and inject it via dependency.

50. ALWAYS size the AlloyDB connection pool using `(alloydb_max_connections * 0.8) / num_cloud_run_instances` — over-provisioning connections exhausts AlloyDB's connection limit and triggers errors for all services sharing the instance.

---

## Vertex AI and Gemini

51. ALWAYS use Gemini 2.0 Flash (`gemini-2.0-flash-001`) as the default LLM for new workloads — it is the cost-effective April 2026 standard. Upgrade to Gemini 2.5 Pro only for tasks where Flash measurably fails your quality threshold.

52. NEVER hardcode model names as bare strings like `"gemini-pro"` — always use the full versioned model ID (`"gemini-2.0-flash-001"`) to prevent unexpected behavior when Google aliases change.

53. ALWAYS call `vertexai.init(project=..., location=...)` once at application startup — not inside request handlers. The init call establishes credentials and configuration; calling it per-request wastes CPU.

54. ALWAYS use `generate_content_async()` for Vertex AI calls in FastAPI — the synchronous `generate_content()` blocks the event loop. Use async or `run_in_executor` if the SDK version predates native async.

55. ALWAYS set `max_output_tokens` in every Gemini generation config — leaving it unset allows the model to generate up to its full context window, producing unbounded latency and cost.

56. ALWAYS stream Gemini responses for outputs expected to exceed 500 tokens — streaming with `stream=True` reduces time-to-first-token from seconds to ~200ms, dramatically improving perceived performance.

57. NEVER pass raw user-supplied content directly to Gemini without explicit delimiters — wrap it in `<user_message>...</user_message>` tags and instruct the model that content inside those tags is data, not instructions. Prompt injection via Gemini is a documented attack vector.

58. ALWAYS initialize Vertex AI with the `staging_bucket` parameter when deploying Reasoning Engine agents — the staging bucket stores serialized agent code during deployment.

59. ALWAYS use Vertex AI Model Garden for Claude when your architecture requires Claude on GCP — `anthropic.AnthropicVertex` routes billing through GCP and keeps model calls within your project's VPC perimeter.

60. ALWAYS use `text-embedding-004` for text embeddings on Vertex AI (April 2026 current) — not `textembedding-gecko` which is the deprecated predecessor.

---

## Vertex AI Vector Search

61. ALWAYS use AlloyDB pgvector + SCANN for datasets up to ~10M vectors — Vertex AI Vector Search adds operational complexity (separate service, separate billing, round-trip overhead) that's unwarranted at this scale.

62. ALWAYS migrate to Vertex AI Vector Search when your AlloyDB vector table exceeds 10M rows or your vector search latency SLO cannot be met — at this scale, dedicated ANN infrastructure outperforms a general-purpose DB.

63. NEVER mix vector storage (Vertex AI Vector Search) and metadata storage (AlloyDB) queries in the hot path without caching — the extra round trip adds 20-50ms per query. Cache vector search results by query embedding hash.

---

## Pub/Sub

64. ALWAYS use Push subscriptions over Pull subscriptions for Cloud Run consumers — push subscriptions have Pub/Sub initiate delivery to your Cloud Run endpoint, eliminating the polling loop and scaling with Cloud Run's concurrency naturally.

65. ALWAYS set a Dead Letter Topic on every Pub/Sub subscription — after N failed delivery attempts (set `--max-delivery-attempts 5`), messages land in the DLT for debugging rather than being silently dropped.

66. ALWAYS make Pub/Sub message handlers idempotent — Pub/Sub guarantees at-least-once delivery. A message can be delivered multiple times. Use a deduplication key (message attribute or database record) to detect and skip reprocessing.

67. NEVER return 5xx from a Pub/Sub push endpoint unless you want the message redelivered — return 200/204 to acknowledge (even if you chose not to process it). Return 4xx/5xx only when you want Pub/Sub to retry delivery.

68. ALWAYS use Pub/Sub message attributes for routing and filtering — `event_type`, `tenant_id`, `version` as attributes enable subscription filters without parsing the message body.

69. ALWAYS use BigQuery subscriptions when the goal is streaming Pub/Sub data into BigQuery — direct BigQuery subscriptions eliminate the consumer Cloud Run service for this pattern.

70. ALWAYS use Cloud Tasks instead of Pub/Sub when you need exactly-one HTTP target delivery, deduplication by task name, or per-task scheduling — Pub/Sub is fan-out; Tasks is targeted.

---

## Secret Manager

71. ALWAYS store all secrets in Secret Manager — database passwords, API keys, OAuth client secrets, JWT signing keys. Never in environment variables set at deploy time with literal values, never in `terraform.tfvars` committed to git.

72. ALWAYS inject secrets into Cloud Run as environment variables via `--set-secrets` — this reads the secret at instance startup and injects it as an env var, avoiding per-request Secret Manager API calls.

73. NEVER call Secret Manager's `access_secret_version` inside request handlers — call it once at application startup (or lifespan). Each SDK call is a network round trip and an IAM check.

74. ALWAYS version secrets — add a new version rather than overriding the current value. Keep at least the previous version active during rotation to avoid a brief window where both old and new credential holders fail.

75. ALWAYS use Secret Manager's rotation notifications (Pub/Sub topic + rotation schedule) for secrets that require periodic rotation — manual rotation is unreliable; automate it.

76. ALWAYS grant `roles/secretmanager.secretAccessor` to the Cloud Run service account, not to the project — grant at the secret resource level, not the project level, to enforce least privilege per secret.

---

## Workload Identity Federation

77. ALWAYS use Workload Identity Federation for GitHub Actions, GitLab CI, and other CI/CD systems — WIF issues short-lived tokens tied to the CI job's OIDC identity. No JSON key files, no credential rotation, no expiry management.

78. NEVER create a service account key for CI/CD pipelines — if your CI system can't use WIF (very rare), escalate to your infrastructure team. Do not accept "we need a key file" as the answer.

79. ALWAYS use `attribute.repository` in WIF attribute mappings to bind the pool provider to a specific GitHub repository — without this, any GitHub OIDC token (from any repo in the world) could potentially authenticate to your pool.

80. ALWAYS use `google-github-actions/auth@v2` in GitHub Actions workflows — it handles the WIF token exchange natively without scripting `gcloud auth workload-identity-pools ...`.

81. ALWAYS set `id-token: write` permissions in GitHub Actions workflows that use WIF — the workflow needs permission to request the OIDC token from GitHub's token endpoint.

---

## Cloud Storage

82. ALWAYS enable uniform bucket-level access on all GCS buckets — per-object ACLs are a legacy pattern that creates unpredictable access behavior. IAM controls all access in uniform mode.

83. ALWAYS use V4 signed URLs — not V2. V4 is the current standard; V2 is deprecated and less secure. Maximum expiry is 7 days.

84. NEVER generate signed URLs using a service account key file — use ADC with `google.auth.default()`. The signing service (`iam.serviceAccounts.signBlob`) handles signing without key files.

85. ALWAYS set GCS lifecycle rules for cost optimization — Standard → Nearline at 30 days, Nearline → Coldline at 90 days, delete at 365 days (adjust for your retention requirements).

86. ALWAYS serve uploaded user files via signed URLs, not public URLs — signed URLs enforce expiry and can be revoked by rotating the SA. Public URLs cannot be revoked once distributed.

87. NEVER store uploaded files on Cloud Run instance filesystem — Cloud Run instances are ephemeral and share no filesystem. Use GCS for all persistent file storage.

---

## IAM and Terraform

88. ALWAYS use `google_project_iam_member` in Terraform, not `google_project_iam_binding` — `iam_binding` is authoritative for that role and will REMOVE any other members with that role when applied. `iam_member` is additive. In shared projects, `iam_binding` silently destroys other teams' access.

89. ONLY use `google_project_iam_binding` in Terraform when you are the sole owner of that role on the project and you want authoritative enforcement — this is rare in practice.

90. ALWAYS use `google_secret_manager_secret_iam_member` to grant secret access at the secret resource level — granting `roles/secretmanager.secretAccessor` at the project level gives the SA access to ALL secrets in the project.

91. ALWAYS use the GCS backend for Terraform remote state — `backend "gcs" { bucket = "..." prefix = "..." }`. Enable bucket versioning for state recovery after accidental `terraform destroy`.

---

## Infrastructure as Code

92. NEVER use Google Cloud Deployment Manager for new infrastructure — it is a legacy tool that is rarely updated and has limited community support. Use Terraform or Pulumi.

93. ALWAYS pin Terraform provider versions with `~>` constraints — `version = "~> 6.0"` allows patch updates but prevents major version upgrades that may contain breaking changes.

94. ALWAYS use the `google-beta` provider for Terraform resources that use features in beta — some AlloyDB, Cloud Run, and Vertex AI features are only in the beta provider. Reference them as `google-beta` explicitly.

95. ALWAYS store Terraform state in GCS with versioning enabled — versioning enables `gsutil cp gs://bucket/prefix/terraform.tfstate.BACKUP .` recovery when state is corrupted.

96. ALWAYS use Terragrunt for managing multiple environments with the same Terraform modules — copy-paste of entire Terraform configs per environment leads to configuration drift.

97. NEVER commit `terraform.tfstate` or `terraform.tfstate.backup` to git — these files contain resource IDs and potentially sensitive values. They belong in the GCS remote backend.

---

## Observability

98. ALWAYS emit structured JSON logs from Cloud Run services — Cloud Logging automatically parses JSON stdout, making fields like `user_id`, `latency_ms`, and `error` queryable in the Logs Explorer.

99. ALWAYS include `severity`, `user_id`, `request_id`, and `latency_ms` in every structured log entry from Cloud Run — these are the minimum fields for useful operational debugging.

100. NEVER use `print()` as the logging mechanism in Cloud Run — use Python's `logging` module or structured JSON to stdout. `print()` produces unstructured text that cannot be queried by field.

101. ALWAYS configure a Log Router sink to BigQuery for Cloud Run logs — Cloud Logging's default retention is 30 days. For compliance or long-term analysis, route to BigQuery with `--use-partitioned-tables`.

102. ALWAYS set up at least one uptime check and one alerting policy on every production Cloud Run service — GCP does not alert you when your service is down by default. Configure uptime checks + notification channels (Slack, PagerDuty) explicitly.

103. ALWAYS use OpenTelemetry with `CloudTraceSpanExporter` for distributed tracing — do not use `google-cloud-trace` directly; OTel is the vendor-neutral standard and works with Cloud Trace's exporter.

104. ALWAYS use `BatchSpanProcessor` for OTel trace export — `SimpleSpanProcessor` is synchronous and blocks request threads on every span export. Batch is async and non-blocking.

105. ALWAYS sample traces with `TraceIdRatioBased(0.1)` in production — 10% sampling captures representative data without the storage and export cost of 100% sampling. Always-sample errors regardless of rate.

---

## Security — GCP Specific

106. ALWAYS enable VPC Service Controls for projects that handle sensitive data or regulated workloads — VPC SC creates an API-level perimeter that prevents data exfiltration even with valid credentials outside the perimeter.

107. ALWAYS use Binary Authorization to enforce that only images built by your CI system (Cloud Build) can be deployed to Cloud Run — it prevents arbitrary container images from being deployed with a compromised gcloud credential.

108. NEVER expose AlloyDB or Memorystore Valkey to public internet IPs — these services must be on private VPC IPs only, accessible via VPC connector from Cloud Run. The AlloyDB Auth Proxy handles the mTLS layer; no firewall rules needed for port 5432.

---

## GCP MCP Servers

109. ALWAYS configure `google-cloud-mcp` with ADC (`gcloud auth application-default login`) and never with a service account key file — the MCP server inherits ADC automatically.

110. ALWAYS scope the Cloud Run service account used for `google-cloud-mcp` to read-only roles during development — `roles/bigquery.dataViewer`, `roles/storage.objectViewer`, `roles/secretmanager.viewer` (not secretAccessor). The MCP server should not be able to modify production data.

---

*April 2026. GCP SDK versions: google-cloud-aiplatform >= 1.70, google-cloud-alloydb-connector >= 1.5, google-cloud-pubsub >= 2.25, Terraform google provider ~> 6.0.*

---

## Artifact Registry, Build, CDN, Security (Gap-Check Pass 1)

111. NEVER push container images to `gcr.io` for new projects — Container Registry is deprecated April 2026. Use Artifact Registry (`REGION-docker.pkg.dev/PROJECT/REPO/IMAGE`) exclusively.

112. ALWAYS create an Artifact Registry Docker repository per environment per region — grant `roles/artifactregistry.reader` to the Cloud Run SA and `roles/artifactregistry.writer` to the Cloud Build SA.

113. ALWAYS configure liveness and readiness probes separately on Cloud Run Gen 2 — startup probe fires at start; liveness detects deadlocks (restart instance); readiness controls traffic routing (remove from LB without restart). All three serve distinct purposes.

114. ALWAYS set `failureThreshold >= 3` and `periodSeconds >= 10` on Cloud Run liveness probes — low thresholds cause cascading restarts under load.

115. ALWAYS use Private Service Connect (PSC) when connecting Cloud Run to AlloyDB at scale — PSC has no bandwidth cap; Serverless VPC Access connector is capped at 1 Gbps shared across all services.

116. NEVER use Public IP AlloyDB instances in production — enable Private IP only (`ipv4_enabled = false` in Terraform); connect via AlloyDB Auth Proxy or PSC endpoint.

117. ALWAYS use Cloud Build for container builds in GCP-native stacks — it runs inside your project's VPC, accesses Artifact Registry without WIF, and produces provenance attestations for Binary Authorization.

118. NEVER set `_BUILD_TIMEOUT` > 20 minutes in Cloud Build — parallelize slow steps with `waitFor` instead.

119. ALWAYS attach Cloud Armor to the Global External HTTPS LB — apply at minimum the OWASP ModSecurity CRS preconfigured rule set plus a rate-based ban rule.

120. NEVER rely on VPC firewall rules alone for publicly accessible Cloud Run endpoints — Cloud Armor blocks DDoS at the Google edge before traffic enters your VPC.

121. ALWAYS use Identity-Aware Proxy (IAP) to protect internal Cloud Run services (admin UIs, internal APIs) — IAP verifies Google identity at the LB before any byte reaches your application.

122. ALWAYS verify the `X-Goog-IAP-JWT-Assertion` header in your FastAPI app when IAP is enabled — prevents bypass attacks where traffic reaches Cloud Run directly (not via the LB).

123. ALWAYS use Cloud Run over GKE Autopilot for stateless FastAPI services that don't require Kubernetes primitives — choose GKE Autopilot only for DaemonSets, StatefulSets, custom admission webhooks, or GPU workloads.

124. ALWAYS run the AlloyDB Auth Proxy as a Cloud Run sidecar container (April 2026 sidecar support) — connects on `127.0.0.1:5432` with zero network hop; configure `dependsOn` so proxy starts first.

125. ALWAYS verify Firebase ID tokens server-side using `firebase-admin` SDK in a FastAPI dependency — `auth.verify_id_token(id_token)`. Never trust the decoded payload from the client.

126. ALWAYS enable Vertex AI Model Monitoring for production Gemini endpoints — configure `skew_thresholds` and `drift_thresholds`; alert when prediction drift exceeds 0.3 Jensen-Shannon divergence.

127. ALWAYS use Vertex AI Experiments to A/B test LLM prompt versions — log prompt version and quality metrics per run; use `compare_runs()` before promoting a new prompt to production.

128. ALWAYS use Cloud Spanner instead of AlloyDB when your workload requires multi-region strong consistency or 99.999% SLA — Spanner provides external consistency that AlloyDB does not.

129. ALWAYS use Bigtable for time-series workloads >1TB, IoT at >100K writes/second, or ML feature stores — use reversed timestamp prefix in row keys to avoid hot spots. NEVER use Bigtable for relational queries or datasets under 1TB.

130. ALWAYS configure Cloud CDN on the Global HTTPS LB for cacheable Cloud Run responses — set `Cache-Control: public, max-age=300, s-maxage=600`; use `Vary: Authorization` to prevent cross-user cache hits.

---

## Gap-Check Pass 2 — Pub/Sub ordering, Cloud Tasks, Eventarc, AlloyDB SCANN, Gemini function calling, Vertex grounding, Cloud Run traffic splitting, gcloud AI/ML patterns

131. ALWAYS enable message ordering on a Pub/Sub subscription with `--enable-message-ordering` and set `ordering_key` on published messages when strict per-key sequencing is required — without an ordering key, Pub/Sub delivers messages in parallel with no sequence guarantee; ordering keys cause all messages with the same key to be delivered to the same subscriber in publish order.

132. NEVER mix ordered and unordered messages on the same subscription — if a message with an ordering key is published to a topic, all subsequent messages with that key must also carry it; dropping the key mid-stream causes the subscription to stall until the out-of-order message is acknowledged or the ordering state is reset.

133. ALWAYS use Pub/Sub exactly-once delivery (April 2026 GA) by setting `--enable-exactly-once-delivery` on the subscription and acknowledging with the ack ID within the ack deadline — exactly-once delivery requires the publisher to set `message_id` deduplication; duplicate publishes within the dedup window (10 minutes) are discarded.

134. ALWAYS configure Cloud Tasks queues with explicit `--max-dispatches-per-second` and `--max-concurrent-dispatches` — without rate limits, a large backlog drains instantly and overwhelms the target service; set rate limits to match the target service's sustainable throughput, not its burst capacity.

135. ALWAYS configure the Cloud Tasks retry schedule with `--max-attempts`, `--min-backoff`, `--max-backoff`, and `--max-doublings` — the default unlimited retries with exponential backoff can cause tasks to retry for days; set `--max-attempts 10` and `--max-backoff 300s` as a baseline.

136. ALWAYS use Cloud Tasks task name deduplication (`task.name = "projects/P/locations/L/queues/Q/tasks/UNIQUE_ID"`) for idempotent task submission — if a task with that name already exists and hasn't completed, Cloud Tasks returns `ALREADY_EXISTS`; this prevents double-submission without a separate dedup store.

137. ALWAYS use Eventarc to trigger Cloud Run services from GCP events (Audit Log, Pub/Sub, GCS, Workflows) — Eventarc handles retry logic, dead-lettering, and CloudEvents formatting automatically; do not build custom Pub/Sub push subscriptions for GCP-native event sources.

138. ALWAYS filter Eventarc triggers to the minimum required event types using `--event-filters` — unfiltered triggers on busy resources (GCS buckets, Audit Logs) generate enormous trigger volumes; filter by `methodName`, `serviceName`, and `resourceName` to reduce invocations and cost.

139. ALWAYS validate the CloudEvents `ce-type` and `ce-source` headers in your Cloud Run handler when receiving Eventarc triggers — these headers identify the event type and source; validate before processing to prevent spoofed event delivery from non-Eventarc sources.

140. ALWAYS create AlloyDB SCANN indexes with `CREATE INDEX ON embeddings USING scann (embedding vector_cosine_ops)` — use cosine ops for normalized embeddings (most embedding models), L2 ops only when your distance metric requires Euclidean distance. HNSW is valid but SCANN outperforms it on AlloyDB at > 500K vectors.

141. ALWAYS use Gemini function calling with explicit `tool_config` set to `mode=ANY` when you need guaranteed tool use, and `mode=AUTO` when the model should decide — `mode=NONE` disables tools; omitting `tool_config` defaults to AUTO but this default may change between model versions, so always specify it explicitly.

142. ALWAYS define `function_declarations` with complete JSON Schema `parameters` including `type`, `description`, and `required` fields — incomplete schemas cause Gemini to generate malformed tool call arguments; validate every function declaration against the JSON Schema spec before deploying.

143. ALWAYS use Vertex AI grounding with Google Search (`tools=[Tool(google_search_retrieval=GoogleSearchRetrieval())]`) for factual queries requiring up-to-date information — grounding reduces hallucination on current events, product information, and citations; use it when Gemini's training cutoff is insufficient for the task.

144. ALWAYS use Cloud Run traffic splitting for blue/green deployments by tagging revisions: `gcloud run services update-traffic SERVICE --to-revisions REVISION=10` increments to 10%, validate, then `--to-revisions REVISION=100` for full cutover — never delete the previous revision until the new one is validated under production traffic.

145. ALWAYS use `gcloud ai models list --region=us-central1` and `gcloud ai endpoints list` to audit deployed Vertex AI endpoints — stale endpoints incur per-node-hour charges even with zero traffic; set a calendar reminder to audit endpoints monthly.

---

## Gap-Check Pass 3 — Reasoning Engine, Gemini multimodal, BigQuery Vector Search, Cloud Run GPU, AlloyDB AI embeddings, Secret rotation, GCS signed URLs v4, Pub/Sub BigQuery subscription

146. ALWAYS deploy LangChain and LangGraph agents to Vertex AI Reasoning Engine (`reasoning_engines.ReasoningEngine.create()`) when you need a fully managed agent runtime — Reasoning Engine handles autoscaling, request routing, and session state persistence without managing Cloud Run services for agent orchestration.

147. ALWAYS call `reasoning_engines.ReasoningEngine.create(agent, requirements=[...], extra_packages=[...])` with a pinned `requirements` list — Reasoning Engine builds a container from your spec; unpinned requirements produce non-reproducible builds that break silently when upstream packages update.

148. ALWAYS pass multimodal inputs to Gemini 2.0 using `Part.from_uri(gcs_uri, mime_type=...)` for large files stored in GCS — inline base64 (`Part.from_data(bytes, ...)`) is limited to ~20MB and adds serialization overhead; GCS URIs allow Gemini to read files directly without the API payload size limit.

149. ALWAYS set `mime_type` explicitly when passing images, video, audio, or documents to Gemini — do not rely on content-type inference; `image/jpeg`, `video/mp4`, `audio/mp3`, and `application/pdf` are valid MIME types for Gemini 2.0 multimodal inputs.

150. ALWAYS use BigQuery Vector Search (`VECTOR_SEARCH()` function, GA April 2026) for analytics-scale vector similarity over data already in BigQuery — it eliminates the ETL step of exporting embeddings to a separate vector store; use AlloyDB pgvector for transactional queries and BigQuery Vector Search for analytical/batch similarity workloads.

151. NEVER use BigQuery Vector Search as a replacement for AlloyDB pgvector in online serving paths — BigQuery query startup latency (100ms-2s) is unacceptable for synchronous API responses; BigQuery Vector Search is optimized for batch and analytical use cases, not sub-100ms online retrieval.

152. ALWAYS use Cloud Run GPU instances (`--accelerator type=nvidia-l4,count=1`) for inference workloads that require GPU but don't justify GKE — L4 GPUs on Cloud Run (April 2026) support PyTorch, TensorFlow, and vLLM containers; Cloud Run GPU autoscales to zero between requests, eliminating idle GPU costs that GKE node pools incur.

153. ALWAYS set `--cpu 8 --memory 32Gi` as a minimum alongside `--accelerator type=nvidia-l4` on Cloud Run GPU services — the L4 GPU requires sufficient CPU and memory headroom for host-side pre/post-processing; underspecification causes GPU wait stalls.

154. ALWAYS use AlloyDB AI's built-in `google_ml.embed(provider, model, content)` function to generate embeddings directly in SQL — this eliminates the application-layer embedding call and associated latency for bulk embedding generation and index updates; it calls the Vertex AI embedding API from within the database transaction.

155. ALWAYS grant `roles/aiplatform.user` to the AlloyDB service account when using `google_ml.embed()` — AlloyDB calls Vertex AI on behalf of the database instance; without this role, embedding generation fails with a permission denied error.

156. ALWAYS use Secret Manager automatic rotation with a Cloud Functions trigger on the `SECRET_VERSION_ADDED` Pub/Sub event — the rotation function generates a new credential, stores it as a new secret version, updates the dependent service, and disables the old version; manual rotation is error-prone and misses the rotation window.

157. ALWAYS include the exact `host` and `content-type` headers in the signed URL request at signing time and in the actual HTTP request — GCS V4 signed URLs validate that headers present at signing time exactly match headers on the actual request; a signed URL generated without `Content-Type` fails if the client sends it.

158. ALWAYS set the maximum expiry on GCS V4 signed URLs to 7 days (`expiration=datetime.timedelta(days=7)`) — V4 signed URLs expire after a maximum of 604800 seconds (7 days); requesting longer expiry raises an exception. For long-lived access, use IAM and public bucket policies instead.

159. ALWAYS use Pub/Sub BigQuery subscriptions (`--bigquery-table PROJECT:DATASET.TABLE`) when streaming Pub/Sub messages directly into BigQuery — this eliminates a Cloud Run consumer service, Dataflow, or Pub/Sub-to-BigQuery Dataflow template; the subscription handles schema mapping via `--use-topic-schema` if a schema is registered.

160. NEVER use the Pub/Sub BigQuery subscription without a registered topic schema — without `--use-topic-schema`, all message data lands in a single `data` BYTES column with no column mapping; define a Pub/Sub schema (Avro or Protobuf) on the topic first.

---

## Gap-Check Pass 4 — Cloud Run execution environment, Memorystore TLS, min-instances cost, CUDs, Org Policy, Error Reporting, Cloud Run CORS, GCP APM

161. ALWAYS set `--execution-environment gen2` and `--sandbox=none` for Cloud Run services that require maximum throughput — `sandbox=minivm` (the gen2 default) adds a lightweight VM isolation layer; `sandbox=none` removes that layer for trusted first-party code, reducing cold start time by ~200ms and steady-state overhead by ~5%.

162. NEVER use `sandbox=none` for Cloud Run services that execute untrusted user code (sandboxed eval, user-supplied scripts) — the VM sandbox provides the only isolation boundary between user code and the host; removing it in untrusted execution contexts is a container escape risk.

163. ALWAYS connect to Memorystore Valkey with TLS in production — set `ssl=True` (redis-py) or `ssl_context` (valkey-py); Memorystore Valkey supports TLS on port 6378. Plaintext connections on port 6379 expose session tokens and cache payloads to network sniffing within the VPC.

164. ALWAYS calculate the min-instances breakeven before setting `--min-instances > 0` — compare `(cold_start_latency_ms * cold_start_rate_per_second * p99_impact_cost)` against `(min_instance_cost_per_month)`; for services receiving > 1 request per minute, min-instances=1 pays for itself in latency SLO savings.

165. ALWAYS purchase AlloyDB Committed Use Discounts (1-year or 3-year) for production database instances — AlloyDB CUDs provide 25% (1-year) to 52% (3-year) discounts on vCPU and memory costs; production databases are always-on workloads that qualify immediately for CUD savings.

166. NEVER apply a single Cloud SQL or AlloyDB CUD to multiple instance types — CUDs are region-scoped and resource-type-scoped; a vCPU CUD for AlloyDB does not cover Cloud SQL vCPUs; plan CUDs per service per region.

167. ALWAYS enforce the `constraints/compute.vmExternalIpAccess` Organization Policy to restrict which projects can assign external IPs to VMs — this prevents accidental public exposure of internal services and satisfies CIS GCP Benchmark control 4.9.

168. ALWAYS enforce the `constraints/iam.requireOsLogin` Organization Policy for all projects that use Compute Engine — OS Login ties SSH access to Google identity and removes per-VM `~/.ssh/authorized_keys` management, making SSH access revocable via IAM.

169. ALWAYS enable Google Cloud Error Reporting for Cloud Run services by emitting structured log entries with a `@type: "type.googleapis.com/google.devtools.clouderrorreporting.v1beta1.ReportedErrorEvent"` field — Error Reporting automatically groups, deduplicates, and alerts on exceptions extracted from structured logs; no SDK required for Cloud Run.

170. ALWAYS configure Cloud Run CORS at the application layer (FastAPI `CORSMiddleware`) — Cloud Run infrastructure does not provide CORS headers; CORS is purely application responsibility. IAP and Cloud Armor operate below the HTTP CORS layer and cannot substitute for it.

171. ALWAYS use Cloud Trace, Cloud Profiler, and Cloud Debugger together as the GCP APM stack — Cloud Trace for distributed request tracing, Cloud Profiler for continuous CPU/memory flame graphs in production, Cloud Debugger (Snapshot) for non-breaking production inspection; the three tools correlate via `trace_id`.

172. ALWAYS install the `google-cloud-profiler` package and call `googlecloudprofiler.start(service="NAME", service_version="VERSION")` at Cloud Run startup for production performance analysis — Cloud Profiler has < 1% CPU overhead and provides continuous heap and CPU profiles without separate profiling runs.

173. ALWAYS use `google-cloud-error-reporting` SDK or structured log emission for error grouping — do not rely on `logging.exception()` alone; without the Error Reporting structured type field, exceptions appear as unstructured log lines that cannot be grouped or alerted on in Error Reporting.

174. ALWAYS set `X-Cloud-Trace-Context` propagation in your FastAPI middleware by reading the incoming header and passing it to outbound `httpx` requests — without trace context propagation, Cloud Trace shows disconnected single-service spans instead of an end-to-end distributed trace.

---

## Gap-Check Pass 5 — AlloyDB pool sizing, Vertex batch vs online, Terraform state, Cloud SQL vs AlloyDB migration, Cloud Run domain mapping vs Global LB, Pub/Sub schema registry, GKE Autopilot Spot, Cloud Profiler

175. ALWAYS size the AlloyDB asyncpg connection pool as `max_size = min(alloydb_max_connections * 0.8 / num_cloud_run_instances, 20)` and `min_size = 2` — the upper bound of 20 per instance prevents a single Cloud Run revision from monopolizing the DB during scale-up; `min_size = 2` keeps two warm connections ready without health-check round trips.

176. ALWAYS set `max_inactive_connection_lifetime = 300` on asyncpg pools connecting to AlloyDB — AlloyDB Auth Proxy drops idle connections after 10 minutes; setting pool recycle below that threshold prevents `connection reset by peer` errors on revived connections.

177. ALWAYS use Vertex AI batch prediction for embedding generation, evaluation scoring, and classification jobs over > 10K records — batch prediction costs 50% less than online prediction and is not subject to per-minute quota limits; use online prediction only for synchronous user-facing inference.

178. NEVER use Vertex AI online prediction endpoints for background jobs or scheduled tasks — online endpoints are priced per node-hour with a 1-node minimum; an always-on endpoint for batch workloads pays for idle capacity. Use batch prediction or Cloud Run Jobs with direct Gemini API calls instead.

179. ALWAYS store Terraform state in a GCS bucket with `versioning = true` and `uniform_bucket_level_access = true`, and enable state locking via the GCS backend's built-in lock — the GCS backend acquires a lock object before writes; without versioning, a corrupted state file is unrecoverable.

180. ALWAYS use separate GCS state buckets per environment (prod/staging/dev) — a single bucket with different prefixes does not protect against an operator running `terraform destroy -target=module.prod` against the wrong prefix; bucket-level separation adds an IAM boundary.

181. NEVER configure Terraform GCS backend with `credentials` pointing to a key file — use ADC or WIF; the `credentials` field in the GCS backend configuration stores a key file path in plaintext in `backend.tf`, which is committed to git.

182. ALWAYS migrate from Cloud SQL to AlloyDB using the Database Migration Service (DMS) with continuous replication mode — DMS handles schema conversion, initial load, and ongoing CDC replication; cutover involves a brief maintenance window with `STOP REPLICATION` rather than a full outage migration.

183. ALWAYS validate AlloyDB-specific features (SCANN indexes, columnar engine, `google_ml.embed()`) in a staging AlloyDB instance before DMS cutover — these features require AlloyDB-specific extensions that do not exist on Cloud SQL; running them in staging catches extension compatibility issues before production cutover.

184. ALWAYS use Global External HTTPS Load Balancer instead of Cloud Run domain mapping when you need WAF (Cloud Armor), multi-region failover, CDN, or custom SSL policies — Cloud Run domain mapping provides a simple CNAME-based custom domain with no WAF, no CDN, and single-region scope.

185. ALWAYS use Cloud Run domain mapping only for simple single-region services that need a custom domain with no additional LB features — it provisions a managed certificate and routes traffic with zero Terraform config; use it for non-critical or low-traffic services where LB cost ($18+/month) is unjustified.

186. ALWAYS register Pub/Sub schemas in the Schema Registry (`gcloud pubsub schemas create NAME --type=AVRO --definition-file=schema.avsc`) and attach them to topics with `--message-encoding=JSON` or `--message-encoding=BINARY` — schemas are validated at publish time, rejecting malformed messages before they enter the subscription queue.

187. ALWAYS use Avro schemas over Protobuf schemas in the Pub/Sub Schema Registry when the consumer is Python — the `fastavro` library handles Avro deserialization with better Python ecosystem support than `protobuf`; use Protobuf only when cross-language binary compatibility is required.

188. ALWAYS use GKE Autopilot Spot node pools (`nodeConfig.spot = true`) for batch ML training jobs and Spark workloads — Spot nodes provide 60-91% cost reduction over standard nodes; batch workloads tolerate preemption via checkpointing. NEVER run stateful production services on Spot nodes.

189. ALWAYS configure `PodDisruptionBudget` with `minAvailable = 1` for all GKE Autopilot deployments that serve traffic — Autopilot may evict pods for bin-packing during scale-down; without a PDB, all replicas of a small deployment can be evicted simultaneously.

---

## Gap-Check Pass 6 — Final anti-patterns, NEVER rules, FastAPI health endpoint patterns, Terraform testing, future-forward patterns

190. NEVER deploy a Cloud Run service without a `/health` endpoint that returns HTTP 200 within 1 second — Cloud Run's startup probe and load balancer health check both hit this endpoint; a missing or slow health endpoint causes rolling deployments to stall and triggers unnecessary instance restarts.

191. ALWAYS implement the FastAPI health endpoint as a two-tier response: `/health` returns 200 immediately for liveness (Cloud Run), and `/health/ready` checks DB connectivity and dependency availability for readiness (LB health check) — conflating the two causes healthy instances to be removed from the LB when a downstream dependency has a transient error.

192. NEVER call AlloyDB, Memorystore, or external APIs inside the `/health` liveness endpoint — liveness probes fire frequently (every 10 seconds); blocking I/O in liveness checks causes probe timeouts under load, triggering cascading restarts.

193. ALWAYS implement structured health responses with `{"status": "ok", "version": GIT_SHA, "checks": {...}}` from `/health/ready` — the `version` field enables verification that the correct revision is deployed; `checks` exposes individual dependency status for debugging.

194. ALWAYS use the Terraform `terraform test` framework (Terraform 1.6+) for infrastructure testing — write `.tftest.hcl` files that apply a plan, run assertions on resource attributes, and destroy; this validates infrastructure before `terraform apply` to production environments.

195. NEVER test Terraform modules only with `terraform validate` and `terraform plan` — `validate` checks syntax only; `plan` does not catch IAM propagation delays, quota exhaustion, or regional resource constraints; `terraform test` with an actual apply against a dedicated test project catches these.

196. ALWAYS use Cloud Build local emulator (`cloud-build-local`) for iterating on `cloudbuild.yaml` changes — the emulator runs build steps locally using Docker, catches step failures before consuming Cloud Build minutes, and avoids triggering downstream deployments during pipeline development.

197. NEVER put Cloud Build substitution variables that contain secrets in `cloudbuild.yaml` directly — use `secretEnv` with Secret Manager references (`availableSecrets.secretManager`) and `$$VAR` syntax in build steps; plain substitution variables (`$_VAR`) appear in Cloud Build logs.

198. NEVER use `latest` as a Terraform module source version — always pin to a specific git tag or commit SHA: `source = "git::https://github.com/org/modules.git//cloudrun?ref=v2.3.1"`; unpinned modules silently pull breaking changes on the next `terraform init -upgrade`.

199. ALWAYS enable Vertex AI Feature Store for ML features that are shared across multiple models or pipelines — Feature Store provides online serving (< 10ms) and offline batch retrieval from the same feature definition; duplicating feature logic across pipelines creates training-serving skew.

200. ALWAYS use Gemini 2.5 Pro (`gemini-2.5-pro-preview-MMDD`) for complex multi-step reasoning tasks when it becomes stable (expected mid-2026) — Gemini 2.5 Pro with extended thinking provides significantly better performance on code generation, mathematical reasoning, and multi-hop retrieval than 2.0 Flash; use it selectively for tasks where quality justifies 5-10x cost premium.

201. NEVER rely on Vertex AI model aliases (`gemini-pro`, `gemini-flash`) in production code — aliases resolve to different model versions over time without notice; pin to the full versioned model ID and update intentionally after evaluating quality changes.

202. ALWAYS instrument Cloud Run services with `google-cloud-profiler` and enable Trillium TPU instances on Vertex AI for large-scale training workloads (when available in your region, 2026) — Trillium (TPU v5e successor) provides 4x training throughput over v4 TPUs at equivalent cost for Gemini-class model fine-tuning.

203. NEVER deploy Gemini fine-tuned models to Vertex AI dedicated endpoints for low-traffic use cases — use the shared endpoint (`tuned_model.generate_content()`) which has no per-hour cost and scales to zero; dedicated endpoints require a minimum of 1 deployed replica at $2-8/hour regardless of traffic.

204. ALWAYS use VPC Service Controls in combination with Cloud KMS Customer-Managed Encryption Keys (CMEK) for AlloyDB and GCS in regulated workloads — VPC SC prevents data exfiltration via API; CMEK ensures Google cannot decrypt your data without your key; together they satisfy FedRAMP High and HIPAA technical safeguard requirements.

205. ALWAYS run `gcloud organizations get-iam-policy ORG_ID --format=json | jq '.bindings[] | select(.role | contains("Owner") or contains("Editor"))'` as a weekly audit job in Cloud Scheduler — primitive roles (`roles/owner`, `roles/editor`) granted at the organization level give unrestricted access to every project; this audit detects policy drift before it becomes a breach.
