---
description: Azure cloud architecture rules for AI-native FastAPI backends (April 2026). 231 rules.
paths: ["*.py", "*.bicep", "*.tf", "*.yaml"]
---

# Azure Patterns — Rules (April 2026)

> Companion wiki: `~/knowledge/wiki/backend/azure-complete.md`
> These rules apply to any Azure-hosted FastAPI backend as of April 2026.

---

## Identity and Authentication

1. NEVER use service principal client secrets for any compute resource (Container Apps, AKS, Functions, VMs) — use Managed Identity instead. Client secrets expire, leak into logs, and require manual rotation.

2. ALWAYS use `DefaultAzureCredential` from `azure-identity` as the single credential for all Azure service clients — it automatically resolves to Managed Identity in production and `az login` in local development with no code change.

3. NEVER hardcode Azure OpenAI API keys in code or environment variables in production — use Managed Identity with `get_bearer_token_provider(credential, "https://cognitiveservices.azure.com/.default")` and the `azure_ad_token_provider` parameter on `AzureOpenAI`.

4. ALWAYS use system-assigned Managed Identity unless a resource needs to authenticate as the same identity from multiple compute resources — use user-assigned only when you have a documented multi-resource identity sharing requirement.

5. NEVER create Azure AD (Entra ID) app registration client secrets with an expiry longer than 1 year — shorter is better; 90 days is the recommended maximum for service-to-service keys.

6. ALWAYS use Workload Identity Federation for GitHub Actions CI/CD pipelines — configure a federated credential linking the GitHub repo + branch to the Entra app registration. Never store `AZURE_CLIENT_SECRET` as a GitHub secret.

7. NEVER grant a Managed Identity `Owner` or `Contributor` at subscription scope — follow least privilege. Grant only the specific built-in roles needed at the resource group or individual resource scope.

8. ALWAYS use `ManagedIdentityCredential(client_id="...")` explicitly when a resource has multiple user-assigned managed identities — `DefaultAzureCredential` will pick an arbitrary one if multiple are present on the same resource.

9. NEVER use the `az login --service-principal --password` flow with a client secret in production scripts or CI pipelines — use federated credentials (`--federated-token`) or Managed Identity (`--identity`) instead.

---

## Key Vault

10. ALWAYS use Key Vault references in Container Apps for secrets rather than fetching secrets in application code — the pattern `keyvaultref:https://{vault}.vault.azure.net/secrets/{name},identityref:system` injects the secret as an environment variable with no SDK required in the app.

11. NEVER use legacy Key Vault access policies — always use Azure RBAC (`--enable-rbac-authorization true`) for Key Vault. Access policies are a deprecated authorization model as of April 2026.

12. ALWAYS enable soft delete and purge protection on every Key Vault — `enable-soft-delete: true`, `retention-days: 90`, `enable-purge-protection: true`. This is required for most compliance frameworks and prevents accidental or malicious permanent deletion.

13. NEVER grant `Key Vault Secrets User` role at the vault scope to application identities — scope it to the specific secret path (`{vault-id}/secrets/{secret-name}`) to enforce least privilege. Only deploy pipelines need vault-wide access.

14. ALWAYS store the Application Insights connection string in Key Vault and inject it via Key Vault reference — it is a credential-equivalent value that grants write access to your telemetry data.

15. NEVER rotate a Key Vault secret by deleting and recreating with the same name — use the versioning model: create a new version, update the consumer (new deployment), then let the old version expire.

---

## Compute — Container Apps

16. ALWAYS use Azure Container Apps (ACA) for new containerized workloads — do not use Azure App Service for new builds. App Service is legacy; ACA provides scale-to-zero, KEDA, Dapr, and built-in revision management.

17. NEVER choose AKS over Container Apps unless you have a specific requirement: direct Kubernetes API access, custom node pools (GPU, ARM), specific Kubernetes operators, or fine-grained CNI networking. For most API and worker workloads, ACA is sufficient and dramatically simpler.

18. ALWAYS use `--min-replicas 1` for external-facing APIs to avoid cold starts on the request path. Use `--min-replicas 0` only for background workers and batch jobs that can tolerate startup latency.

19. NEVER deploy a new Container App image by updating the existing revision in-place — always use `az containerapp update --image new-image:tag` which creates a new revision. Old revisions remain and can receive traffic instantly during rollback.

20. ALWAYS use Container App revision labels and traffic splitting for zero-downtime deployments: deploy new revision with 0% traffic, validate via its direct revision URL, then gradually shift traffic.

21. ALWAYS set resource limits on Container App containers (`--cpu` and `--memory`) — Container Apps without limits share the environment's capacity and can starve other apps.

22. NEVER use Container Apps `--scale-rule-type http` with `concurrentRequests` set higher than 100 for APIs with P95 latency over 500ms — calculate the correct concurrency target as `(target_replicas * requests_per_second * avg_latency_seconds)`.

23. ALWAYS configure Service Bus-based KEDA scaling (`--scale-rule-type azure-servicebus`) for worker Container Apps — this scales to zero when idle and responds to queue depth, eliminating idle compute cost.

24. ALWAYS assign the system-assigned Managed Identity to every Container App at creation — `--system-assigned`. Do this before the first deployment so the identity is stable.

25. NEVER expose Container App ingress as `external` for internal service-to-service communication — use `internal` ingress and communicate within the Container Apps Environment via the app's internal FQDN.

26. ALWAYS use Container Apps Jobs for database migrations, seed scripts, and one-time operational tasks — jobs have the same image, environment, and identity as the app, with job-specific execution semantics and logging.

---

## Database — PostgreSQL Flexible Server

27. NEVER use Azure Database for PostgreSQL Single Server — it reached end-of-life in March 2025. Always use Flexible Server.

28. ALWAYS use the built-in PgBouncer connection pooler (port 6432) for application connections — direct port 5432 connections bypass pooling and cause connection exhaustion at scale. Only use port 5432 for DDL migrations, `LISTEN`/`NOTIFY`, and advisory locks.

29. ALWAYS enable Entra ID authentication on PostgreSQL Flexible Server for production — use `az postgres flexible-server ad-admin set` to configure the Entra admin, then create PostgreSQL roles for managed identities via `pgaadauth_create_principal`.

30. ALWAYS enable the `vector` extension on PostgreSQL Flexible Server when using pgvector — configure via `az postgres flexible-server parameter set --name azure.extensions --value "VECTOR"`.

31. ALWAYS use HNSW indexes (`USING hnsw`) for pgvector in production rather than IVFFlat — HNSW requires no training data, handles incremental inserts, and has lower query latency at high recall thresholds.

32. NEVER use `Zone Disabled` high availability for production PostgreSQL Flexible Server — use `ZoneRedundant` for RPO near zero. The standby in a different availability zone provides automatic failover without manual intervention.

33. ALWAYS configure `max_inactive_connection_lifetime` in asyncpg connection pools when using Managed Identity tokens for PostgreSQL auth — tokens expire every 60 minutes. Recycling connections before expiry prevents authentication failures mid-pool.

34. NEVER use the `autocommit` connection mode for transactional application logic — all application connections should use explicit transactions via asyncpg's `acquire()` context manager.

---

## Database — Cosmos DB

35. ALWAYS design the partition key before creating a Cosmos DB container — it is immutable after creation. Choose a key with high cardinality, even distribution, and that appears in every query's WHERE clause.

36. NEVER use monotonically increasing values (`createdAt`, sequential integers) as a Cosmos DB partition key — they create hot partitions where all new writes go to the same physical partition, throttling the entire account.

37. ALWAYS use synthetic partition keys (`{tenantId}:{entityType}`) for multi-tenant SaaS workloads — this distributes data evenly while keeping each tenant's data collocated within their partition.

38. NEVER use Cosmos DB serverless tier for workloads requiring geo-replication or consistent high throughput — use provisioned throughput with autoscale instead.

39. ALWAYS specify the partition key in Cosmos DB queries when known at query time — cross-partition queries consume significantly more RUs and have higher latency.

40. ALWAYS use the Cosmos DB Change Feed for event-driven processing of data mutations — it is more reliable than polling and avoids additional Service Bus hops for Cosmos-originated events.

41. NEVER use Cosmos DB for relational data with complex JOINs or multi-table ACID transactions — use PostgreSQL Flexible Server. Cosmos DB multi-document transactions are limited to a single partition.

---

## Azure OpenAI Service

42. ALWAYS use `AzureOpenAI` (or `AsyncAzureOpenAI`) from the `openai` library — never use the base `OpenAI` client for Azure OpenAI Service. The endpoint, authentication, and API versioning differ.

43. NEVER use Azure OpenAI API keys in production code or environment variables — authenticate exclusively via Managed Identity using `azure_ad_token_provider=get_bearer_token_provider(credential, "https://cognitiveservices.azure.com/.default")`.

44. ALWAYS set `max_tokens` (GPT-4o) or `max_completion_tokens` (o-series) on every Azure OpenAI API call — never leave it unset. A missing limit allows the model to consume the full context window and produce unexpected $10+ charges from a single request.

45. ALWAYS use `max_completion_tokens` (not `max_tokens`) for o3 and o4-mini reasoning models — the `max_tokens` parameter is deprecated for o-series and controls a different limit.

46. NEVER set `temperature` for o3 or o4-mini models — they use `reasoning_effort` ("low", "medium", "high") instead. Setting temperature on reasoning models causes a validation error.

47. ALWAYS use `reasoning_effort="low"` for o4-mini on latency-sensitive paths and `reasoning_effort="high"` for complex single-user analytical tasks — the three levels have roughly 3x latency/cost differences.

48. NEVER use GPT-4o or o3 for tasks reliably handled by GPT-4o-mini — classification, entity extraction, routing, simple summarization, and structured output generation do not require the larger model. GPT-4o-mini is ~30x cheaper per token.

49. ALWAYS use `text-embedding-3-small` (1536 dims) as the default embedding model — only upgrade to `text-embedding-3-large` (3072 dims) when benchmarking shows measurable retrieval quality improvement for your specific dataset.

50. ALWAYS leverage Matryoshka Representation Learning (MRL) truncation for `text-embedding-3-large` when storage cost is a concern — passing `dimensions=256` or `dimensions=512` reduces storage by 6-12x with minimal quality loss.

51. ALWAYS configure content safety filters on Azure OpenAI deployments via Azure AI Foundry — never deploy a production Azure OpenAI resource with default permissive filter settings.

52. ALWAYS catch `BadRequestError` with `e.code == "content_filter"` and handle it explicitly — content filter rejections are expected events in production and should return a specific user-facing message, not an unhandled 500.

53. ALWAYS use PTU (Provisioned Throughput Units) deployments for predictable high-volume workloads with SLA requirements — implement a PTU-to-standard overflow pattern using `RateLimitError` catch to avoid user-facing 429s when PTU capacity is exhausted.

54. ALWAYS cache embedding API responses keyed by `sha256(input_text)` in Redis with a TTL of at least 7 days — embeddings are deterministic for the same model version. Caching eliminates redundant API calls for duplicate or repeated content.

55. ALWAYS check `response.usage.prompt_tokens_details.cached_tokens` for GPT-4o calls with large system prompts — a cached_tokens count of 0 on repeated calls indicates prompt structure is preventing cache hits (system prompt is being modified).

56. NEVER use the Azure OpenAI Assistants API for stateless request-response patterns — Assistants are for stateful, multi-turn, file-using conversations. For one-shot completions, use chat completions directly.

57. ALWAYS use `async with client.chat.completions.stream(...)` for streaming responses in async FastAPI endpoints — this is the context manager streaming API that properly handles cleanup on client disconnect.

---

## Messaging — Service Bus

58. ALWAYS use Azure Service Bus Premium tier for production workloads — it provides VNet integration, private endpoints, large message support (100MB), and guaranteed throughput. Standard tier is for dev/test only.

59. NEVER send a Service Bus message without setting `time_to_live` — messages without TTL accumulate in the queue indefinitely if not consumed, causing storage bloat and stale message processing.

60. ALWAYS set `max_delivery_count` to 5 or less on Service Bus queues and subscriptions — messages exceeding this count are automatically moved to the Dead Letter Queue. Higher values delay problem detection.

61. ALWAYS implement explicit DLQ monitoring — set an alert when `ActiveMessageCount` on the DLQ (`{queue}/$deadletterqueue`) exceeds 0. A non-empty DLQ is a processing failure, not a background condition.

62. NEVER use `complete_message()` before you have successfully processed the message — complete after processing. On any exception path, either `abandon_message()` (transient) or `dead_letter_message()` (permanent) the message.

63. ALWAYS use Service Bus sessions (`session_id`) when processing messages that must maintain per-entity ordering — e.g., per-user state machine events, per-order event sequences. Sessions guarantee FIFO within the session.

64. NEVER poll Service Bus queues with a sleep loop — use the `async for message in receiver:` iterator pattern with `max_wait_time` configured. The SDK handles long polling efficiently.

65. ALWAYS use `create_message_batch()` and `send_messages(batch)` when publishing more than one message — batch sends reduce network round trips and are atomic within the batch.

---

## Storage — Blob Storage

66. ALWAYS use User Delegation SAS tokens for client-facing (presigned) blob access — never use Account Key SAS in production. User Delegation SAS is signed by Entra ID credentials, can be revoked via user delegation key revocation, and does not expose the account key.

67. NEVER expose the storage account access key to application code — grant the Container App's Managed Identity `Storage Blob Data Contributor` or `Storage Blob Data Reader` role and use `DefaultAzureCredential`.

68. ALWAYS set `allow-blob-public-access false` on all storage accounts — never enable anonymous blob access. All access must be authenticated.

69. ALWAYS set `min-tls-version TLS1_2` on storage accounts — TLS 1.0 and 1.1 are deprecated and disabled by Azure Policy in most enterprise tenants anyway; enforce it explicitly.

70. ALWAYS configure lifecycle management policies on blob containers — move blobs to Cool after 30 days, Archive after 90 days, delete after your retention requirement. Unconfigured storage grows unbounded and incurs unnecessary Hot tier costs.

71. NEVER use `SAS` token expiry longer than 1 hour for upload/download operations from untrusted clients — if you need longer-lived access, generate a new SAS token from your API rather than extending the initial token.

---

## Infrastructure as Code — Bicep

72. NEVER write or modify Azure ARM JSON templates directly — always use Bicep, which compiles to ARM. ARM JSON is the build artifact, not the source. Editing ARM JSON is error-prone and loses Bicep's type safety and linting.

73. ALWAYS run `az deployment group what-if` before every production Bicep deployment — review the change diff for unexpected modifications (especially deletions) before applying.

74. ALWAYS use `az bicep lint` in CI as a blocking check before deployment — Bicep linting catches type errors, missing properties, and best practice violations before they reach production.

75. NEVER use `listKeys()` or `listConnectionStrings()` in Bicep to pass secrets between resources — use Key Vault references or Managed Identity instead. `listKeys()` embeds the secret in the deployment state file.

76. ALWAYS use the `@secure()` decorator on Bicep parameters that contain secrets — this prevents the value from appearing in deployment logs and the ARM deployment history.

77. ALWAYS pin Bicep resource API versions (e.g., `@2024-03-01`) — using `latest` makes deployments non-deterministic across Bicep compiler versions. Use the `az bicep generate-params` to get current stable API versions.

78. ALWAYS output the Container App's FQDN, managed identity principal ID, and resource IDs from Bicep templates — these outputs are consumed by subsequent deployment steps without requiring additional `az` queries.

79. NEVER deploy infrastructure from local developer machines in production — always run Bicep deployments from CI/CD pipelines authenticated via Workload Identity Federation.

---

## Infrastructure as Code — Terraform

80. ALWAYS use `azurerm` provider version `~> 4.0` for new Terraform modules — azurerm v4 changed the default behavior for several resources (storage accounts, Key Vault) and removed deprecated aliases.

81. ALWAYS use the `azapi` provider for Azure resources not yet available in `azurerm` — `azapi` calls the ARM API directly and supports all resource types immediately, without waiting for azurerm provider releases.

82. ALWAYS store Terraform state in Azure Blob Storage with state locking — configure the `azurerm` backend with a dedicated storage account using LRS replication and private access only.

83. NEVER commit `terraform.tfstate` or `terraform.tfstate.backup` to version control — these files contain sensitive resource metadata and credentials in plaintext.

84. ALWAYS use `azurerm_role_assignment` with `principal_type = "ServicePrincipal"` when assigning roles to Managed Identities — omitting `principal_type` causes unnecessary Entra ID lookups and can produce intermittent errors.

---

## Observability

85. ALWAYS instrument FastAPI with `FastAPIInstrumentor` and configure `azure-monitor-opentelemetry` — auto-instrumentation captures all HTTP request traces, durations, and error rates with no manual code required.

86. ALWAYS set `cloud_role_name` in Application Insights configuration to the Container App name — this is how the Application Map and cross-service traces identify your service. Without it, all services appear as "Unknown Service".

87. ALWAYS use KQL (Kusto Query Language) for Log Analytics queries, not legacy OMS query syntax — write all queries in KQL format. Use `summarize`, `bin()`, `project`, `extend`, and `render` operators.

88. NEVER set Log Analytics workspace retention below 30 days for production workspaces — 30 days is the minimum for meaningful incident investigation. 90 days is recommended for compliance-sensitive workloads.

89. ALWAYS create metric alerts for P95 latency > 5s and error rate > 1% over a 5-minute window — these are the two most important early warning signals for production API degradation.

90. ALWAYS create a Log Analytics alert for Dead Letter Queue depth > 0 on all Service Bus queues — DLQ messages indicate processing failures that require manual investigation.

91. NEVER use `SimpleSpanProcessor` in production OpenTelemetry configuration — it blocks the request thread on every span export. Use `BatchSpanProcessor` with the Azure Monitor exporter.

---

## Networking and Security

92. NEVER allow public network access to PostgreSQL Flexible Server in production — use VNet integration and private endpoints. Allow public access only temporarily during initial setup if private networking is not yet configured.

93. ALWAYS configure a Container Apps Environment with VNet injection when PostgreSQL Flexible Server, Azure Cache for Redis, or Service Bus require private endpoint access — without VNet injection, Container Apps cannot access private-endpoint-only resources.

94. NEVER expose Azure management APIs (Key Vault, Service Bus management plane, Storage management plane) from within application code — these are control plane operations. Use the data plane SDKs (`azure-keyvault-secrets`, `azure-servicebus`, `azure-storage-blob`).

95. ALWAYS set `allow_azure_services_access = false` on PostgreSQL Flexible Server firewall rules when using private endpoints — the "Allow Azure Services" rule creates a broad IP exception that bypasses your VNet security.

96. NEVER put Azure subscription IDs, tenant IDs, or resource IDs in client-facing API responses — these are internal identifiers that help attackers map your Azure resource topology.

---

## Cost and Scaling

97. ALWAYS set `min-replicas 0` for Container App workers and batch jobs — idle workers at minimum 1 replica incur constant compute cost. Workers should scale from zero on demand via KEDA triggers.

98. NEVER use PostgreSQL Flexible Server `GeneralPurpose` tier for development environments — use `Burstable` tier (e.g., `Standard_B2ms`). It costs 60-80% less for intermittent workloads.

99. ALWAYS stop PostgreSQL Flexible Server and Redis instances in non-production environments outside business hours — use Container App Jobs with a cron trigger to call `az postgres flexible-server stop` and `az redis start/stop`.

100. ALWAYS use `GPT-4o-mini` for Azure OpenAI tasks that do not require advanced reasoning: classification, keyword extraction, intent detection, simple summarization, JSON structure validation. Reserve `GPT-4o` for generation, analysis, and complex reasoning.

101. NEVER use `o3` for tasks solvable by `o4-mini` — o3 is 10x more expensive per token than o4-mini. Use o4-mini as the default reasoning model and escalate to o3 only when benchmark scores require it.

102. ALWAYS configure Azure OpenAI token-per-minute (TPM) limits on deployments — unbounded TPM on shared deployments allows one runaway request loop to exhaust the quota for all callers. Set TPM limits matching your application's expected peak load.

---

## Local Development

103. ALWAYS use `az login` for local development authentication — `DefaultAzureCredential` picks it up automatically at position 4 in its credential chain. Never use service principal client secrets for developer workstations.

104. ALWAYS create a separate Resource Group and Azure OpenAI deployment for development — share the production deployment only through CI/CD. Developer experiments should not affect production quota.

105. ALWAYS use the `--what-if` flag on `az deployment group create` during development iterations — it shows exactly which resources will be created, modified, or deleted without making any changes.

---

## Container Registry

106. ALWAYS use Azure Container Registry (ACR) with Managed Identity pull access for Container Apps — grant the Container App identity `AcrPull` role on the registry. Never use ACR admin credentials.

107. ALWAYS enable ACR geo-replication to match the regions where Container Apps are deployed — a Container App pulling images from a registry in a different region incurs egress costs and slower deployments.

108. ALWAYS configure ACR Tasks or integrate with GitHub Actions to automatically build and push images on commit — manual `docker push` to production registries is error-prone and bypasses CI checks.

---

## Gap-Check Pass 1

### KEDA Scalers, Dapr, Front Door, CDN, PgBouncer, PTU, Cosmos Change Feed, Azure Monitor Alerts

109. ALWAYS use the KEDA HTTP scaler (`--scale-rule-type http`, `--scale-rule-http-concurrency`) for external-facing Container Apps APIs, and the Service Bus scaler (`--scale-rule-type azure-servicebus`, `--scale-rule-metadata queueName={name} messageCount={N}`) for worker apps — choose the scaler that matches the actual load signal. HTTP scaler cannot detect queue backlog; Service Bus scaler cannot detect concurrent HTTP connections.

110. ALWAYS configure custom KEDA scalers via `ScaledObject` YAML for workloads driven by signals not covered by built-in triggers (Prometheus metrics, Cosmos DB change feed depth, Azure Event Hubs consumer lag) — deploy the KEDA ScaledObject into the Container Apps environment using `az containerapp update --yaml`.

111. ALWAYS set `cooldownPeriod` on KEDA scalers to at least 60 seconds for API apps — an aggressive default of 5 seconds causes replica thrashing when traffic is bursty. For batch workers, 300 seconds is a safer default.

112. ALWAYS enable Dapr on Container Apps via `--enable-dapr --dapr-app-id {name} --dapr-app-port {port}` when the app needs state management, pub/sub, or service invocation building blocks — Dapr sidecars handle retries, circuit breaking, and mTLS between services transparently.

113. ALWAYS configure a Dapr state store component pointing to Azure Cache for Redis or Cosmos DB — never use in-memory Dapr state stores in production. In-memory state is lost when a replica is recycled.

114. ALWAYS configure a Dapr pub/sub component pointing to Azure Service Bus — use Dapr pub/sub rather than the Service Bus SDK directly when the Container Apps environment already has Dapr enabled. This decouples the app from the broker and enables component swaps without code changes.

115. NEVER use Azure Traffic Manager for latency-based routing to Container Apps in April 2026 — use Azure Front Door Standard/Premium instead. Front Door combines global anycast routing, WAF, DDoS protection, and CDN caching in one resource. Traffic Manager is a DNS-only solution without HTTP-layer capabilities.

116. ALWAYS use Azure Front Door Premium (not Standard) for production AI backends that require WAF with OWASP rule sets, bot protection, and private origin connectivity via Private Link — Front Door Standard lacks private origin support and the managed WAF rule sets.

117. NEVER use Azure Application Gateway as a global load balancer — it is a regional resource. Use Application Gateway only when you need regional ingress with WAF and URL-based routing within a single Azure region. For multi-region global routing, place Front Door in front of Application Gateway.

118. ALWAYS use Azure Front Door as the CDN for static assets hosted in Azure Blob Storage in April 2026 — the legacy Azure CDN products (Akamai, Verizon, Microsoft Standard) reached end-of-life retirement. Front Door Standard/Premium replaced them as the unified CDN and global load balancer.

119. ALWAYS use the built-in PgBouncer on PostgreSQL Flexible Server (port 6432) rather than deploying a PgBouncer sidecar container — the built-in pooler is managed, monitored in Azure metrics, and scales with the server. A sidecar adds operational complexity without benefit when the built-in pooler meets your needs.

120. NEVER use the built-in PgBouncer for connections that require `LISTEN`/`NOTIFY`, `PREPARE` statements, advisory locks, or `SET` session variables — PgBouncer in transaction mode resets session state between transactions. Use a direct connection on port 5432 for these specific use cases only.

121. ALWAYS use Azure OpenAI PTU (Provisioned Throughput Units) deployments when your workload requires a latency SLA, consistent throughput above 40K TPM, or predictable monthly cost — PTU provides reserved model capacity billed hourly regardless of usage. Use the PTU capacity calculator in Azure AI Foundry to size PTU units: `PTU = (peak_TPM / tokens_per_PTU_per_minute_for_model)`.

122. ALWAYS implement a PTU overflow pattern: route requests to the PTU deployment first, catch `RateLimitError` (HTTP 429) returned when PTU capacity is saturated, and retry immediately on a standard pay-per-token deployment — this eliminates user-facing rate limit errors while controlling cost.

123. ALWAYS use the Cosmos DB Change Feed processor library (`azure-cosmos` `ChangeFeedProcessor`) rather than polling with `ORDER BY _ts` queries — the change feed processor tracks per-partition lease state in a separate Cosmos container, supports parallel processing across multiple workers, and guarantees at-least-once delivery.

124. NEVER read the Cosmos DB change feed from a single-threaded loop in production — deploy the change feed processor in a Container App that scales based on the lease container's unprocessed item count via a custom KEDA scaler.

125. ALWAYS use dynamic (machine learning-based) alert thresholds for Azure Monitor metric alerts on workloads with daily or weekly seasonality (e.g., API request rates that spike during business hours) — static thresholds produce false positives during off-peak troughs and miss spikes on anomalous peaks. Set `aggregation-type Dynamic` with `sensitivity medium`.

126. ALWAYS use static thresholds for alerts where the breach condition is an absolute safety limit rather than a deviation from normal behavior — DLQ depth > 0, error rate > 5%, replica count = 0 are absolute conditions. Dynamic thresholds are appropriate only for pattern-based anomaly detection.

127. ALWAYS set alert severity correctly: Sev0 (Critical) for revenue-impacting outages, Sev1 (Error) for degraded availability, Sev2 (Warning) for approaching limits, Sev3 (Informational) for operational awareness — mixing severities causes alert fatigue when Sev0 fires for non-critical conditions.

128. ALWAYS configure action groups with email + PagerDuty/OpsGenie webhook for Sev0/Sev1 alerts, and email-only for Sev2/Sev3 — every alert must have at least one action group; alerts with no action group are invisible in production incidents.

---

## Gap-Check Pass 2

### ACR Defender, Event Hubs Kafka, AI Search Hybrid, Embedding Dimensions, Cosmos Vector, Service Bus Sessions, Key Vault Soft-Delete, Azure Policy

129. ALWAYS enable Microsoft Defender for Containers on every Azure Container Registry used in production — Defender scans images on push and on a daily schedule, detecting OS and package CVEs. Without it, vulnerable base images are silently promoted to production.

130. ALWAYS configure Defender for Containers to push scan results to Microsoft Defender for Cloud and set a policy that blocks Container App deployments referencing images with Critical or High CVEs — integrate the scan gate into the CI/CD pipeline using the `az acr task` output.

131. ALWAYS use the Event Hubs Kafka endpoint (`{namespace}.servicebus.windows.net:9093`) to connect existing Kafka producers/consumers to Azure Event Hubs without code changes — the connection string format is `Endpoint=sb://{namespace}.servicebus.windows.net/;SharedAccessKeyName=...;SharedAccessKey=...` formatted as a Kafka SASL/SSL `bootstrap.servers` and `password` pair.

132. NEVER create a separate Kafka cluster (Confluent, MSK, self-hosted) in Azure when Event Hubs with Kafka endpoint meets your throughput requirements — Event Hubs supports up to 40 MB/s per throughput unit and scales to hundreds of TUs. Maintain a single messaging tier.

133. ALWAYS use Azure AI Search hybrid search (BM25 keyword + vector) rather than pure vector search for most RAG retrieval scenarios — hybrid search with `@search.rerankerScore` from semantic reranking produces measurably higher recall than either modality alone. Enable both `vectorSearch` and `semanticConfiguration` in the index definition.

134. ALWAYS create a `vectorSearch` configuration in Azure AI Search using the HNSW algorithm profile — HNSW (`"kind": "hnsw"`) provides lower query latency and no training requirement compared to exhaustive KNN. Set `m=4`, `efConstruction=400`, `efSearch=500` as starting values and tune from benchmark results.

135. ALWAYS use `text-embedding-3-large` with Matryoshka dimension truncation (`dimensions=1536`) in Azure AI Search vector fields when migrating from `text-embedding-ada-002` — this matches the ada-002 dimension count while improving retrieval quality ~10%. Only move to `dimensions=3072` after benchmarking shows recall improvement for your specific corpus.

136. NEVER store raw `text-embedding-3-large` full 3072-dimension vectors in Azure AI Search without evaluating the storage cost impact — 3072 float32 vectors consume 12KB per document. For 10M documents that is 120GB of vector storage. Benchmark `dimensions=256` and `dimensions=512` first.

137. ALWAYS use the Cosmos DB NoSQL vector index (GA as of April 2026) for applications that already use Cosmos DB as their primary store and require ANN vector search — enables `SELECT TOP N c.id, VectorDistance(c.embedding, @queryVector) FROM c ORDER BY VectorDistance(c.embedding, @queryVector)` without a separate search service.

138. NEVER use Cosmos DB vector search as a replacement for Azure AI Search when your workload requires full-text BM25 search, faceting, filtering by multiple fields, or semantic reranking — Cosmos DB vector search is a single-modality index. Use Azure AI Search when hybrid retrieval is required.

139. ALWAYS use Service Bus message sessions (`session_id=entity_id`) for ordered processing of events scoped to a single entity (one user's events, one order's state machine) — without sessions, competing consumers process messages out of order. Sessions guarantee FIFO delivery to a single consumer within the session.

140. ALWAYS use `ServiceBusSessionReceiver` (not `ServiceBusReceiver`) when consuming from a session-enabled queue — the session receiver must call `accept_session(session_id=...)` or `accept_next_session()`. A non-session receiver cannot read from a session-enabled entity.

141. ALWAYS enable soft-delete on Key Vault before any production deployment: `az keyvault update --enable-soft-delete true --soft-delete-retention-days 90` — without soft-delete enabled, a deleted secret, key, or certificate is immediately and permanently gone. This is a prerequisite for most compliance frameworks.

142. ALWAYS enable purge protection on Key Vault before production: `az keyvault update --enable-purge-protection true` — soft-delete alone still allows a `purge` operation to permanently delete a soft-deleted vault object within the retention period. Purge protection blocks this for the entire retention window.

143. NEVER attempt to re-create a Key Vault with the same name in the same subscription after deletion without first purging the soft-deleted vault — Key Vault names are globally unique with a DNS reservation. A soft-deleted vault holds the name for 90 days. Use `az keyvault purge --name {name}` only when purge protection is not enabled.

144. ALWAYS assign the `Azure Policy Contributor` role to the CI/CD service principal and apply the built-in policy set `Azure Security Benchmark` to every subscription — this enforces 200+ security controls automatically including TLS requirements, diagnostic logging, and private endpoint mandates.

145. ALWAYS use Azure Policy `deny` effect for the policies: "Secure transfer to storage accounts should be enabled", "PostgreSQL flexible server should run TLS version 1.2 or higher", and "Public network access should be disabled for PostgreSQL flexible servers" — audit-only policies produce compliance reports but do not prevent misconfiguration.

---

## Gap-Check Pass 3

### Blob Versioning/Soft-Delete, Container Apps Plans, OpenAI Content Filters, Managed Identity Lifecycle, Service Bus DLQ Replay, PostgreSQL Read Replicas, Bicep What-If, Azure Load Testing

146. ALWAYS enable blob versioning (`az storage account blob-service-properties update --enable-versioning true`) on storage accounts that hold user-uploaded content or application state blobs — versioning preserves every write as an immutable version, enabling point-in-time recovery without backup restores.

147. ALWAYS enable blob soft-delete (`--enable-delete-retention true --delete-retention-days 30`) alongside versioning — soft-delete protects against accidental deletion of the current version; versioning protects against accidental overwrites. Both are needed for full accidental-deletion recovery.

148. ALWAYS enable container soft-delete (`--enable-container-delete-retention true --container-delete-retention-days 7`) in addition to blob soft-delete — a deleted container (and all its blobs) is recoverable for the retention period. Without container soft-delete, deleting a container is permanent even with blob soft-delete enabled.

149. ALWAYS use the Consumption workload profile (serverless) for Container Apps when workloads have unpredictable traffic and scale-to-zero is required — Consumption profile charges only for vCPU-seconds and memory-GB-seconds consumed while replicas are running.

150. ALWAYS use Dedicated workload profiles (D4, D8, D16, GPU) for Container Apps when workloads require: GPU compute for local inference, guaranteed minimum CPU/memory not subject to node bin-packing, or egress from a specific set of static IP addresses — Dedicated profiles run on reserved nodes with predictable resource availability.

151. NEVER mix Consumption and Dedicated workload profiles in the same Container Apps Environment without understanding the billing model — Dedicated profile nodes are billed per-hour regardless of replica count. A single Dedicated profile node idle for a month costs the same as one running at full capacity.

152. ALWAYS configure Azure OpenAI content filters in asynchronous annotation mode for streaming endpoints — synchronous filtering blocks the first token until the complete response is evaluated, negating the latency benefit of streaming. Asynchronous annotation annotates completed responses without blocking token delivery.

153. NEVER disable Azure OpenAI content filters in production to reduce latency — the latency impact of content filtering is under 50ms for non-streaming calls and zero for streaming with asynchronous annotation. Disabling filters removes a critical safety layer.

154. ALWAYS use system-assigned Managed Identity for Container Apps, Container App Jobs, and Azure Functions — system-assigned identity is deleted automatically when the resource is deleted, preventing orphaned service principals that accumulate in Entra ID.

155. ALWAYS use user-assigned Managed Identity when: (a) the identity must be pre-provisioned before the compute resource exists (e.g., the identity needs Key Vault RBAC assigned in Bicep before the Container App is deployed), or (b) the same identity authenticates from multiple resources (e.g., multiple Container App revisions sharing a Redis or Storage identity).

156. NEVER delete a user-assigned Managed Identity without first removing it from all resources that reference it — deleting the identity while a Container App still references it causes all Managed Identity authentication to fail silently, producing `401 Unauthorized` with no clear error message.

157. ALWAYS process Service Bus Dead Letter Queue messages by reading from `{queue}/$deadletterqueue` with a `ServiceBusReceiver` pointed at the DLQ sub-queue — to replay a DLQ message, read it, create a new `ServiceBusMessage` from the DLQ message body, send it back to the main queue, then complete the DLQ message. Never resend without inspecting `DeadLetterReason` first.

158. NEVER replay DLQ messages in bulk without understanding the `DeadLetterReason` — `MaxDeliveryCountExceeded` messages failed processing repeatedly and will likely fail again. Fix the processing bug first. `TTLExpiredException` messages may be stale and invalid. Only replay after root cause investigation.

159. ALWAYS create PostgreSQL Flexible Server read replicas in the same region as read-heavy consumers when read traffic exceeds 60% of total query load — read replicas add no additional storage cost (shared base storage) and reduce primary server load.

160. NEVER point write operations (INSERT, UPDATE, DELETE) at a read replica endpoint — read replicas are read-only. Writing to a replica raises `read_only_transaction` errors. Maintain separate SQLAlchemy engine instances: one for the primary, one for the replica pool.

161. ALWAYS use `az deployment group what-if --result-format FullResourcePayloads` for Bicep deployments that modify existing resources — `ResourceIdOnly` result format omits the full property diff, making it impossible to review exactly which properties are changing on an existing Key Vault, PostgreSQL server, or Container App.

162. NEVER approve a Bicep `what-if` output that shows `Delete` for a PostgreSQL Flexible Server, Key Vault, or Storage Account without verifying it is intentional — accidental deletions from Bicep parameterization errors (wrong resource name) are the most common cause of production data loss during infrastructure updates.

163. ALWAYS run Azure Load Testing from a test plan that includes: ramp-up phase (0 to target VUs over 2 minutes), steady-state phase (5 minutes at target VUs), and ramp-down phase (1 minute) — load tests without ramp-up cause an instantaneous connection storm that triggers KEDA autoscaling before the test reaches stable state, producing misleading latency results.

164. ALWAYS set performance baselines in Azure Load Testing after each major deployment — configure pass/fail criteria: P95 response time < 2s, error rate < 1%, average response time < 500ms. Gate CI/CD deployments on load test pass criteria to detect regressions.

---

## Gap-Check Pass 4

### VNet Integration, Private Endpoints, Defender for Containers Runtime, Azure Firewall vs NSG, Cosmos Multi-Region Writes, Service Bus Premium, OpenAI Model Versioning, Container Apps Health Probes

165. ALWAYS deploy Container Apps to a VNet-injected environment (`az containerapp env create --infrastructure-subnet-resource-id {subnet-id}`) when any backend dependency (PostgreSQL, Redis, Service Bus, Key Vault) is restricted to private endpoints — without VNet injection, Container Apps resolve private endpoint DNS but cannot route to the private IP.

166. NEVER use a /27 or smaller subnet for the Container Apps environment infrastructure subnet — Azure recommends /23 (512 addresses) minimum. The environment reserves IP addresses for internal load balancers, DNS, and the control plane. A /27 exhausts in environments with more than 10 apps.

167. ALWAYS create private endpoints for Key Vault, Azure Storage, PostgreSQL Flexible Server, Cosmos DB, and Azure Cache for Redis in production — private endpoints remove all public internet exposure even when public network access is disabled. Public-disable alone can be bypassed by some Azure backbone routing.

168. ALWAYS configure Private DNS Zones linked to the VNet for every private endpoint: `privatelink.vaultcore.azure.net`, `privatelink.blob.core.windows.net`, `privatelink.postgres.database.azure.com`, `privatelink.documents.azure.com`, `privatelink.redis.cache.windows.net` — without Private DNS Zone linkage, private endpoint DNS resolution fails and the client falls back to the public IP.

169. ALWAYS enable Microsoft Defender for Containers runtime threat detection on AKS clusters and Container App environments — Defender detects privilege escalation, container escape attempts, and cryptomining in running containers. Image scanning (Defender for Container Registries) covers only static CVEs; runtime detection is a separate layer.

170. ALWAYS use Network Security Groups (NSGs) as the first line of network perimeter defense for subnet-level traffic control — NSGs are stateful, free, and evaluated before Azure Firewall. Deny all inbound from internet at the NSG level; only allow traffic from Front Door service tag (`AzureFrontDoor.Backend`) on port 443 for public-facing Container App subnets.

171. ALWAYS use Azure Firewall Premium (not NSG) for: egress filtering with FQDN-based rules (blocking LLM exfiltration to non-Azure domains), TLS inspection, IDPS (intrusion detection and prevention), and centralized egress from multiple VNets via hub-spoke topology — NSGs do not support FQDN filtering or TLS inspection.

172. ALWAYS configure Cosmos DB multi-region writes (multi-master) only when write availability in multiple regions simultaneously is a hard requirement — multi-master introduces conflict resolution complexity. For most workloads, single-write-region with read replicas in multiple regions achieves the required availability with no conflict resolution.

173. ALWAYS choose `Last Write Wins` (LWW) conflict resolution policy using `_ts` (system timestamp) for Cosmos DB multi-master when all writes are independent upserts — LWW is the lowest-overhead policy. Use stored procedure-based conflict resolution only when business rules require merge logic.

174. ALWAYS use Service Bus Premium namespace when: the workload requires private endpoints, VNet integration, message sizes over 1MB, geo-disaster recovery (Geo-DR pairing), or dedicated throughput units — Standard namespace supports none of these. Do not start on Standard with a plan to upgrade; namespace tier is set at creation and cannot be changed in-place.

175. ALWAYS configure Service Bus Geo-DR pairing on Premium namespaces for production workloads with RPO < 15 minutes — Geo-DR replicates namespace metadata (queues, topics, subscriptions, policies) to a secondary namespace. In a failover, applications reconnect to the secondary alias endpoint within minutes. Message data in the primary at failover time is not replicated.

176. ALWAYS pin Azure OpenAI model deployment versions using `model_version` in the deployment configuration (e.g., `"2024-11-20"` for `gpt-4o`) rather than `"latest"` — `"latest"` silently rolls to a new model version when Azure retires older versions, changing output behavior, token counts, and cost without any deployment change.

177. NEVER rely on Azure OpenAI `"auto"` model version for production deployments — `"auto"` upgrades happen during Azure's model retirement window, not on your deployment schedule. Pin an explicit version and schedule a controlled migration test when upgrading.

178. ALWAYS configure all three Container App health probe types: startup probe (gives the app time to initialize before liveness takes over), liveness probe (restarts unhealthy replicas), and readiness probe (removes unhealthy replicas from load balancing without restarting) — omitting startup probes on apps with slow initialization causes liveness to kill replicas before they finish starting.

179. ALWAYS implement the `/health/liveness` endpoint as a minimal check (process is alive, no external dependencies) and `/health/readiness` as a dependency check (database connection, cache reachability) — a liveness probe that calls the database will restart healthy replicas when the database is down, compounding the incident.

---

## Gap-Check Pass 5

### Bicep Modules/Registry, Azure Arc, Spring Apps vs Container Apps, OpenAI Assistants API, Cosmos Serverless vs Provisioned, Redis Enterprise RediSearch, App Insights Sampling, DevOps vs GitHub Actions

180. ALWAYS organize Bicep infrastructure into modules: one module per Azure service type (containerApps.bicep, postgreSQL.bicep, keyVault.bicep, openAI.bicep) with a main.bicep orchestrator — monolithic Bicep files over 500 lines are impossible to review and test in isolation.

181. ALWAYS publish reusable Bicep modules to an Azure Container Registry as a Bicep Registry (OCI artifact) — `az bicep publish --file modules/containerApp.bicep --target br:{registry}.azurecr.io/bicep/container-app:1.0.0`. Referencing versioned modules (`br:` scheme) in main.bicep enables updates without copy-pasting.

182. NEVER copy-paste Bicep module files between repositories — publish to a shared Bicep Registry and reference by version. Copy-paste creates N independent copies that diverge over time.

183. ALWAYS use Azure Arc only when you have on-premises Kubernetes clusters, non-Azure cloud VMs, or edge devices that must be managed through Azure Policy, Defender, and GitOps using the same tooling as Azure resources — do not use Azure Arc for pure Azure workloads. It adds management plane complexity with no benefit when all resources are already in Azure.

184. ALWAYS use Azure Container Apps (not Azure Spring Apps) for new Spring Boot workloads unless the workload requires Spring Cloud Config Server, Spring Cloud Service Registry, or Spring Cloud Gateway managed by Azure — Azure Spring Apps Standard tier is being retired. Container Apps provides equivalent compute for Spring Boot jars with simpler pricing.

185. NEVER use the Azure OpenAI Assistants API for stateless one-shot LLM tasks or RAG pipelines that manage their own conversation context — Assistants create persistent threads that consume storage, accumulate cost over time, and require explicit cleanup. Use chat completions with a managed conversation history for stateless workloads.

186. ALWAYS use the Azure OpenAI Assistants API when the use case specifically requires: persistent multi-turn threads that survive client reconnects, file search over uploaded documents (built-in vector store), or code interpreter for data analysis — these capabilities are not available in chat completions and are the correct Assistants use cases.

187. ALWAYS clean up Assistant threads by calling `client.beta.threads.delete(thread_id)` after the conversation is complete — abandoned threads accumulate in Azure OpenAI storage and incur per-GB storage costs. Implement a background cleanup job that deletes threads older than your session TTL.

188. ALWAYS use Cosmos DB serverless tier for: development environments, workloads with fewer than 5,000 RU/s average throughput, and workloads with highly intermittent traffic (scheduled batch jobs, low-traffic APIs) — serverless charges per RU consumed with no hourly minimum. The cost breakeven vs provisioned autoscale is approximately 2,000 RU/s sustained average throughput.

189. NEVER use Cosmos DB serverless for: multi-region geo-replication (not supported on serverless), workloads requiring > 5,000 RU/s sustained, or time-critical workloads where the first-request cold-start latency of serverless is unacceptable — use provisioned throughput with autoscale for these requirements.

190. ALWAYS use Azure Cache for Redis Enterprise (E10, E20, F300 tiers) with the RediSearch module enabled for vector similarity search in scenarios where Redis is already the caching layer — RediSearch `HNSW` vector index in Redis Enterprise avoids adding a separate vector database (Qdrant, Weaviate) to the architecture when query throughput is under 10K QPS.

191. NEVER use Azure Cache for Redis (Basic, Standard, Premium tiers — the non-Enterprise SKUs) for vector search — RediSearch is only available on Redis Enterprise tiers. The standard Redis `ZSCORE` workaround is not an ANN index and does not scale.

192. ALWAYS configure Application Insights adaptive sampling as the default for production FastAPI applications — adaptive sampling automatically reduces telemetry volume during high traffic while preserving 100% of error traces, exception telemetry, and dependency failures. It targets a configurable rate (default 5 events/second per host) rather than a fixed percentage.

193. ALWAYS disable adaptive sampling and use fixed-rate sampling when you need deterministic trace completeness for billing audit or regulatory compliance — adaptive sampling drops requests non-deterministically. Fixed rate (e.g., 10%) ensures a predictable percentage of all request traces are collected.

194. ALWAYS configure telemetry processor filters to exclude health probe requests (`/health/liveness`, `/health/readiness`, `/metrics`) from Application Insights — health probes generate high-frequency, zero-value telemetry that distorts request rate metrics and consumes ingestion quota.

195. ALWAYS use GitHub Actions with Workload Identity Federation (OIDC) for new CI/CD pipelines targeting Azure — Managed Identity OIDC in GitHub Actions is available since 2022 and eliminates stored credentials entirely. Azure DevOps Pipelines also supports Workload Identity Federation as of 2024 and is preferred for enterprises with existing Azure DevOps investment.

196. NEVER store `AZURE_CLIENT_SECRET` as a GitHub Actions or Azure DevOps secret for Azure deployments — use federated credentials (`azure/login@v2` with `client-id`, `tenant-id`, `subscription-id` parameters and no `client-secret`). Stored secrets expire, leak in logs, and require manual rotation.

---

## Gap-Check Pass 6

### Critical NEVER Rules, FastAPI Azure Patterns, Azurite Local Testing, Azure AI Foundry / Phi-4, Cost Anti-Patterns, Well-Architected Framework

197. NEVER deploy a Container App with `--ingress external` and no Front Door or Application Gateway in front — external ingress exposes the Container App directly to the internet without WAF protection, DDoS mitigation, or TLS offloading. Always route public traffic through Front Door Premium with WAF policy attached.

198. NEVER leave Azure OpenAI model deployments at default TPM (Tokens Per Minute) quota without configuring `--capacity` — Azure assigns a default TPM quota that may be shared across all deployments in the resource. A runaway request loop in one deployment can exhaust quota for all callers. Always set explicit TPM per deployment.

199. NEVER use `azure.storage.blob.BlobServiceClient` with a connection string containing the account key in production code — account key connection strings grant full storage account access with no expiry. Use `DefaultAzureCredential` with the storage account URL instead.

200. NEVER provision Azure resources interactively via the Azure Portal for production environments — all production infrastructure must be defined in Bicep or Terraform and deployed via CI/CD. Portal-created resources are invisible to IaC drift detection and cannot be reproducibly destroyed and recreated.

201. ALWAYS implement a FastAPI `/health/liveness` endpoint that returns HTTP 200 with `{"status": "ok"}` within 100ms — Container Apps health probes timeout at 1 second by default; a slow liveness endpoint causes repeated probe failures and unnecessary replica restarts. The liveness endpoint must never call external services.

202. ALWAYS implement a FastAPI `/health/readiness` endpoint that checks: database connection (`SELECT 1`), Redis ping, and any other required dependencies — return HTTP 503 if any dependency is unreachable. Container Apps readiness probe failures remove the replica from the load balancer without restarting it, enabling graceful degradation.

203. ALWAYS configure CORS in FastAPI using `CORSMiddleware` with an explicit `allow_origins` list read from environment variables — never hardcode `["*"]` in production. Load the allowed origins from an env var (`CORS_ORIGINS="https://app.example.com,https://staging.example.com"`) split into a list at startup.

204. ALWAYS use `asynccontextmanager` lifespan in FastAPI to initialize Azure SDK clients (OpenAI, Service Bus, Blob Storage) at application startup and close them at shutdown — initializing clients per-request creates unnecessary TCP connection overhead. Clients should be created once and reused via `app.state`.

205. ALWAYS use Azurite (the Azure Storage emulator) for local development and CI tests that involve Blob Storage, Table Storage, or Queue Storage — `docker run -p 10000:10000 -p 10001:10001 -p 10002:10002 mcr.microsoft.com/azure-storage/azurite`. Configure the SDK with `account_url="http://127.0.0.1:10000/devstoreaccount1"` and `DefaultAzureCredential` falls back to `AzureStorageConnectionString` with the well-known Azurite key.

206. ALWAYS use the Cosmos DB emulator Docker image (`mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:latest`) for local development and CI tests that involve Cosmos DB — the emulator supports the NoSQL API, runs locally without network egress, and avoids development costs. Set `AZURE_COSMOS_EMULATOR_HOST=https://localhost:8081` and disable TLS verification in CI only.

207. ALWAYS disable TLS certificate verification for the Cosmos DB emulator in CI using `CosmosClient(url, credential, connection_verify=False)` — the emulator uses a self-signed certificate. Never disable TLS verification in any non-emulator configuration.

208. ALWAYS use Azure AI Foundry (the renamed Azure AI Studio as of November 2024) as the single portal for: Azure OpenAI model deployment management, Azure AI Search index management, content filter configuration, and AI evaluation runs — do not use the legacy Azure OpenAI Studio or Cognitive Search Portal UIs which are being redirected to AI Foundry.

209. ALWAYS evaluate Phi-4-mini (4B parameter) and Phi-4 (14B parameter) for classification, extraction, and structured output tasks before defaulting to GPT-4o-mini — Phi-4 models run on CPU-only Container Apps instances (`Standard_D4s_v5`) via Ollama sidecar at near-zero variable cost, eliminating Azure OpenAI API call cost for high-frequency low-complexity tasks.

210. NEVER provision a `Standard_D32s_v5` or larger Container App Dedicated workload profile for Phi-4 inference without first benchmarking Phi-4-mini on `Standard_D8s_v5` — Phi-4-mini at 4B parameters achieves comparable benchmark scores to GPT-3.5-turbo at 2-3 tokens/second on D8s_v5. Scale up only if throughput is insufficient.

211. NEVER leave PostgreSQL Flexible Server running in non-production environments continuously — a `Standard_D4s_v5` PostgreSQL server costs approximately $300/month. Stopping it during nights and weekends (16 hours/day, 2 days/week off) reduces cost by 55% at no loss of data.

212. NEVER leave Azure Cache for Redis (Premium P1) running in development environments — P1 costs approximately $160/month. Use Basic C1 ($18/month) for development; it provides the same API surface with reduced throughput and no replication.

213. NEVER over-provision Azure OpenAI PTU without a detailed throughput model — PTU is billed hourly whether used or not. One PTU unit for GPT-4o is approximately $1,800/month. Purchase PTU only after 30 days of pay-per-token usage data establishes a sustained throughput baseline.

214. ALWAYS tag all Azure resources with `environment`, `team`, `cost-center`, and `project` tags via Azure Policy `append` effect — untagged resources make cost allocation to teams impossible. Enforce tagging at resource creation via policy, not via developer convention.

215. ALWAYS apply the Azure Well-Architected Framework Reliability pillar requirements for AI workloads: (a) circuit breaker on all Azure OpenAI calls with fallback to a secondary Azure OpenAI resource in a different region, (b) retry with exponential backoff and jitter on 429 and 503 responses, (c) health model covering all dependencies in the `/health/readiness` endpoint.

216. ALWAYS apply the Azure Well-Architected Framework Security pillar requirements for AI workloads: (a) no API keys — Managed Identity only, (b) all secrets in Key Vault with RBAC, (c) private endpoints for all data services, (d) WAF on Front Door Premium with OWASP CRS 3.2 rule set enabled.

217. ALWAYS apply the Azure Well-Architected Framework Performance Efficiency pillar requirements for AI workloads: (a) embedding cache in Redis keyed by SHA-256 of input, (b) prompt caching via Azure OpenAI `cache_control` breakpoints, (c) KEDA autoscaling on all worker Container Apps, (d) read replicas for PostgreSQL read-heavy query patterns.

218. ALWAYS apply the Azure Well-Architected Framework Cost Optimization pillar requirements: (a) scale-to-zero on all non-external Container Apps, (b) lifecycle management on all Blob Storage containers, (c) auto-pause on non-production databases, (d) right-size model selection (GPT-4o-mini before GPT-4o, Phi-4 before GPT-4o-mini for eligible tasks).

219. ALWAYS apply the Azure Well-Architected Framework Operational Excellence pillar requirements for AI workloads: (a) all infrastructure in Bicep with `what-if` gates in CI, (b) Application Insights with distributed tracing and Langfuse for LLM-specific observability, (c) Azure Monitor alerts with action groups for Sev0/Sev1, (d) load test baseline captured in Azure Load Testing after each major deployment.

220. ALWAYS conduct an Azure Well-Architected Framework review (using the WAF Assessment tool at `aka.ms/waf`) before the first production go-live of any AI workload — the assessment produces a prioritized recommendation list with Azure Advisor integration. Treat Critical and High severity recommendations as deployment blockers.

221. NEVER mix Azure OpenAI deployments across subscription boundaries in a single application without a unified token budget enforcement layer — cross-subscription quota is not automatically aggregated. Each subscription has independent TPM limits. A multi-subscription architecture requires explicit per-subscription quota tracking in Redis.

222. ALWAYS use `azure-ai-inference` SDK (the unified Azure AI model inference SDK as of April 2026) for calling models deployed via Azure AI Foundry model catalog (Phi-4, Mistral, Llama, Cohere) — these models use the `ChatCompletionsClient` from `azure.ai.inference` with a different endpoint format (`https://{endpoint}.models.ai.azure.com`) than Azure OpenAI Service.

223. NEVER confuse Azure OpenAI Service endpoints (`{resource}.openai.azure.com`) with Azure AI Foundry model catalog endpoints (`{endpoint}.models.ai.azure.com`) — they use different SDK clients, different authentication scopes, and different quota systems. Using `AzureOpenAI` client against a Foundry catalog endpoint returns authentication errors.

224. ALWAYS set `Connection: keep-alive` and configure `httpx.AsyncClient` with `http2=True` for Azure Blob Storage and Azure OpenAI SDK calls in high-throughput FastAPI applications — HTTP/2 multiplexing eliminates head-of-line blocking on concurrent requests to the same endpoint, reducing P99 latency for burst workloads.

225. ALWAYS configure the `azure-monitor-opentelemetry` exporter with `disable_offline_storage=False` in production — offline storage buffers telemetry to disk when the Application Insights ingestion endpoint is temporarily unreachable, preventing telemetry loss during network interruptions or ingestion throttling.

226. NEVER deploy a Container App that calls Azure OpenAI without a per-user token budget enforced at the API layer — a single authenticated user in a multi-tenant application can exhaust your entire Azure OpenAI PTU or TPM quota. Implement budget enforcement using atomic Redis `INCR` + `EXPIRE` per `user_id` before every LLM call.

227. ALWAYS use Azure Service Bus topic subscriptions with SQL filter rules to route messages to different consumer Container Apps based on message properties — `az servicebus topic subscription rule create --filter-sql-expression "MessageType = 'EmailNotification'"` eliminates fan-out consumer logic and allows independent scaling per message type.

228. ALWAYS configure Azure Container Apps to pull images using the managed identity `AcrPull` role grant rather than ACR admin credentials — admin credentials are shared, cannot be scoped to specific repositories, and produce a single point of compromise. Managed Identity pull grants are scoped to the specific registry and are audit-logged.

229. ALWAYS enable zone redundancy on Container Apps Environments in production (`az containerapp env create --zone-redundant`) — zone-redundant environments distribute replicas across availability zones automatically. Without zone redundancy, all replicas may land in a single zone and a zone outage takes down the entire app.

230. ALWAYS run `az containerapp logs show --name {app} --resource-group {rg} --follow` for real-time debugging during deployments — Container Apps system logs (`--type system`) show revision activation failures, health probe failures, and KEDA scaling events that are not visible in application logs.

231. NEVER delete a Container Apps Environment without first deleting all Container Apps within it — the environment deletion will fail or leave orphaned DNS and networking resources if apps still exist. Use `az containerapp list --environment {env}` to enumerate and delete all apps before deleting the environment.
