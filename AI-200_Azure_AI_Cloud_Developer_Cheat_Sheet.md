# AI-200 — Developing AI Cloud Solutions on Azure

> **Study cheat sheet | Exam objectives effective October 7, 2026**  
> **Official scope:** [Microsoft AI-200 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200)  
> **Focus:** Backend services and components that power AI solutions — containers, vector storage, messaging, serverless functions, security, and observability.  
> **Updated:** October 16, 2026

## 0. Exam map — what to prioritize

| Official domain | Weight (overview) | Core services |
|---|---:|---|
| Develop containerized solutions on Azure | 20–25% | ACR, App Service, Container Apps, KEDA, AKS |
| Develop AI solutions with Azure data management services | 30–35% | Cosmos DB for NoSQL, PostgreSQL/pgvector, Azure Managed Redis |
| Connect to and consume Azure services | 20–25% | Service Bus, Event Grid, Azure Functions |
| Secure, monitor, and troubleshoot Azure solutions | 20–25% | Key Vault, App Configuration, OpenTelemetry, KQL |

> **Study-guide discrepancy:** The overview lists data management as **30–35%**, but its detailed section heading lists **25–30%**. Follow the overview for planning, and check the guide again before the exam.

**Big picture:** This is mainly an *AI application backend engineering* exam, not a model-training or prompt-engineering exam. Be able to choose a service, configure it, use its SDK, and diagnose failures. Examples below use Python, SQL, Azure CLI, and KQL.

---

## 1. Develop containerized solutions (20–25%)

### 1.1 Azure Container Registry (ACR)

**Know:**
- **Registry** holds **repositories**, which hold **images** with tags/digests. A digest identifies immutable image content; a tag can be reassigned.
- Use **ACR Tasks** for cloud builds: **quick tasks** (manual/on-demand), automated tasks (source commit/base image updates), and multi-step YAML tasks.
- Prefer **managed identity** and minimal registry permissions when apps pull images; avoid registry admin credentials in production.
- `az acr build` builds and pushes on Azure's build infrastructure; `docker build` normally builds on your machine.

```bash
# Build and push from the current directory; registry must exist
az acr build --registry <registry> --image ai-api:v1 .

# Discover the registry's actual login server (can include a generated suffix)
az acr show --name <registry> --query loginServer -o tsv

# Inspect repositories and image tags
az acr repository list --name <registry> -o table
az acr repository show-tags --name <registry> --repository ai-api -o table
```

**Exam traps:** `az acr login --name` takes the *registry resource name*, not the full login server; avoid assuming `registry.azurecr.io` is always the exact hostname. Changing an image tag is not the same as rebuilding/redeploying running containers.

**Official docs:** [ACR CLI quickstart](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli) · [ACR cloud builds](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-quickstart-task-cli) · [Task automation](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-build-task) · [Multi-step task examples](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tasks-samples)

### 1.2 Deploy containers to Azure App Service

- App Service hosts web APIs as managed web apps, including custom container images.
- Know **app settings** (injected into process environment), **Key Vault references**, the custom container's **listening port**, startup failures, and container logs.
- To pull from private ACR, configure the image plus an identity with the proper pull role/permissions. Managed identity support requires configuring App Service to use it.
- Use deployment slots when the scenario calls for staged deployment and swaps; slots are *not* Container Apps revisions.

```bash
# App settings, not hard-coded values in a Dockerfile
az webapp config appsettings set \
  --resource-group <rg> --name <webapp> \
  --settings APP_ENV=production

# Stream application/container logs (where logging is configured)
az webapp log tail --resource-group <rg> --name <webapp>
```

**Exam traps:** An environment variable containing a literal secret is less desirable than a Key Vault reference. ACR authorization, DNS/network access, and the exposed container port are separate failure points.

**Official docs:** [Configure a custom container](https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container?pivots=container-linux) · [App Service app settings](https://learn.microsoft.com/en-us/azure/app-service/configure-common) · [Key Vault references for App Service](https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references)

### 1.3 Azure Container Apps, revisions, and KEDA

**Container Apps = managed container execution and event-driven scaling without managing Kubernetes.**

- **Environment:** shared boundary for networking, logging, and multiple container apps.
- **Revision:** immutable version of a container app template. An image change or scale-rule change typically creates a new revision.
- **Single revision mode:** one active production revision; new ready revision replaces old one.
- **Multiple revision mode:** several active versions; use weights/labels for canary and blue/green rollouts.
- **Application-scope configuration** (such as ingress and secrets) does *not* by itself create a revision. Updating a secret requires a revision restart or redeployment for existing containers to use it.
- **KEDA:** scales replicas using events (e.g., Service Bus queue depth), not only CPU or HTTP traffic. Set min replicas to `0` when scale-to-zero is appropriate; check whether the workload supports it and whether triggers/authentication are configured correctly.

```bash
az containerapp revision list \
  --resource-group <rg> --name <app> -o table

az containerapp logs show \
  --resource-group <rg> --name <app> --follow

az containerapp update \
  --resource-group <rg> --name <app> \
  --image <registry-login-server>/ai-api:v2
```

**Exam traps:** App-scoped secrets ≠ revision-scoped images/scale settings. New revision ≠ automatically receives all traffic when multiple revisions are active. Container Apps manages orchestration; AKS exposes Kubernetes control.

**Official docs:** [Revisions](https://learn.microsoft.com/en-us/azure/container-apps/revisions) · [Manage revisions and traffic](https://learn.microsoft.com/en-us/azure/container-apps/revisions-manage) · [Scaling](https://learn.microsoft.com/en-us/azure/container-apps/scale-app) · [Secrets](https://learn.microsoft.com/en-us/azure/container-apps/manage-secrets)

### 1.4 Azure Kubernetes Service (AKS)

**Know the core manifest objects:**

| Object | Responsibility |
|---|---|
| `Deployment` | Desired replicas, pods, rolling updates |
| `Pod` | Smallest scheduled unit; one or more containers |
| `Service` | Stable network endpoint/load balancing for pods |
| `ConfigMap` | Nonsecret configuration |
| `Secret` | Sensitive Kubernetes values (requires appropriate protection) |
| `Ingress` | HTTP(S) routing into services, with an ingress controller |

```bash
kubectl apply -f deployment.yaml
kubectl get deployments,pods,svc
kubectl describe pod <pod-name>        # state, events, scheduling, image pull
kubectl logs <pod-name> --previous     # previous crashed container, if present
kubectl get events --sort-by=.lastTimestamp
kubectl rollout status deployment/<deployment-name>
```

**Diagnostic pattern:** `Pending` → scheduling/resources; `ImagePullBackOff` → image, tag, registry auth or connectivity; `CrashLoopBackOff` → startup errors, probes, configuration; no external response → Service/Ingress/network/DNS; frequent restarts → logs, resource limits, probes.

**Official docs:** [AKS documentation](https://learn.microsoft.com/en-us/azure/aks/) · [Kubernetes basics on AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-clusters-workloads) · [AKS troubleshooting](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/welcome-azure-kubernetes)

### Hosting decision shortcut

| Requirement | Likely choice |
|---|---|
| Custom containerized web app with App Service operational model | App Service (custom container) |
| Event-driven backend container, revisions, scale-to-zero | Azure Container Apps |
| Full Kubernetes manifests, cluster-level control and ecosystem | AKS |
| Build/store container images | ACR (not a runtime) |

---

## 2. Develop AI solutions with data services (30–35% in overview)

### 2.1 Embeddings and RAG — universal concepts

```text
Source documents → chunk + metadata → embedding model → vector storage/index
User question   → query embedding → filtered vector similarity search
                → top-K relevant chunks → LLM context → grounded answer
```

- **Embedding:** numerical representation of text/images/etc.; semantic neighbors have small distance/high similarity.
- **Dimension:** number of elements in a vector. Your query and stored vectors must have compatible dimensions/model semantics.
- **Distance metrics:** cosine (angle), Euclidean/L2 (geometric distance), inner product (dot product).
- **Exact nearest neighbors:** high recall, potentially more work. **Approximate nearest neighbors (ANN):** lower latency/resource usage, may trade off recall.
- **Metadata filtering:** restrict retrieval by tenant, document type, permissions, date, etc. **Always enforce document authorization**, not just semantic similarity.
- **RAG pattern:** retrieval augments an LLM with up-to-date external context. Vector retrieval alone is not the final answer.

### 2.2 Azure Cosmos DB for NoSQL

**Know:**
- JSON/document store accessed using `azure-cosmos` SDK; SQL-like queries with `SELECT ... FROM c`.
- **Partition key** drives data distribution, query routing, and cost. Point reads using `id` **plus partition key** are efficient. Cross-partition queries can consume significantly more RUs.
- **Request Units (RUs)** account for CPU, I/O, and memory; high RU pressure or insufficient throughput can result in **HTTP 429**. SDK retries, partitioning, efficient queries, and indexing are important.
- **Index policy:** default automatic indexing is convenient but indexing unused large paths increases write cost. Use included/excluded paths and composite indexes appropriately.
- **Consistency levels, strongest → weakest:** `Strong → Bounded staleness → Session → Consistent prefix → Eventual`. **Session** supports read-your-writes for the same session/token.

```python
from azure.cosmos import CosmosClient
from azure.identity import DefaultAzureCredential

client = CosmosClient("https://<account>.documents.azure.com:443/",
                      credential=DefaultAzureCredential())
container = client.get_database_client("appdb").get_container_client("chunks")

item = container.read_item(item="doc-123", partition_key="tenant-a")
```

**Cosmos vector search:**
- Configure **container vector embedding policy** (`path`, `dataType`, `dimensions`, `distanceFunction`) **and vector index** during container creation; vector policy/index changes require a new container in the documented workflow.
- Index options: `flat` = exact search (max **505** dimensions); `quantizedFlat` = quantized vectors (max **4096**); `diskANN` = large-scale ANN (max **4096**).
- Use `VectorDistance(...)` in query expressions; index choice affects RU, latency, and recall.
- Exclude raw vector paths from normal range indexing when configuring vector indexing, per Microsoft's Python vector tutorial.

**Change feed:**
- Captures document changes for downstream work, such as creating/updating embeddings.
- Dedicated **change feed processor** uses a **lease container**, checkpoints and distributes work with at-least-once processing.
- **Important Python caveat:** The change feed *processor library* is supported in **.NET/Java**, not Python. In Python use the **pull model** and manage checkpoints manually, or integrate through an appropriate Azure Functions trigger. Design consumers as idempotent.

**Official docs:** [Cosmos DB query routing](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-query-container) · [Indexing policies](https://learn.microsoft.com/en-us/azure/cosmos-db/index-policy) · [Consistency levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels) · [Vector search](https://learn.microsoft.com/en-us/azure/cosmos-db/vector-search) · [Python vector indexing/query](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-python-vector-index-query) · [Change feed processor](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/change-feed-processor)

### 2.3 Azure Database for PostgreSQL + pgvector

**Know:**
- Relational SQL + JSONB/metadata + embeddings in the same database. Ideal when transactional/relational data and vector retrieval must coexist.
- `pgvector` is the project name; the PostgreSQL extension name is **`vector`**. On Azure, allowlist the extension before `CREATE EXTENSION`.
- `vector(n)` declares the embedding dimension. Index operator class must match the selected distance metric.
- **HNSW:** ANN index with strong search-speed/recall trade-offs but memory/build overhead. **IVFFlat:** ANN index with clustering/lists; tune for data size and recall.
- Evaluate `EXPLAIN (ANALYZE, BUFFERS)`, metadata indexes, filtered queries, memory, CPU, storage IOPS, and connection pressure.
- **PgBouncer:** connection pooling reduces overhead from many short-lived clients/functions.

```sql
-- After allowlisting the Azure extension:
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE chunks (
  id bigserial PRIMARY KEY,
  tenant_id text NOT NULL,
  content text NOT NULL,
  embedding vector(3) NOT NULL
);

CREATE INDEX chunks_tenant_idx ON chunks (tenant_id);
CREATE INDEX chunks_embedding_hnsw_idx
  ON chunks USING hnsw (embedding vector_cosine_ops);

SELECT id, content
FROM chunks
WHERE tenant_id = 'tenant-a'
ORDER BY embedding <=> '[0.1,0.2,0.3]'::vector
LIMIT 5;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM chunks ORDER BY embedding <=> '[0.1,0.2,0.3]'::vector LIMIT 5;
```

**Operators:** `<->` = Euclidean/L2 distance; `<=>` = cosine distance; `<#>` = *negative* inner product. Lower values are better when ordering by distance. With real embeddings use the true model dimension (not the three-element teaching example).

**Exam traps:** Regular B-tree index won't accelerate nearest-neighbor vector search; HNSW doesn't eliminate the need for tenant/authorization filtering; more connections do not necessarily improve throughput; benchmark ANN recall as well as latency.

**Official docs:** [Enable and use pgvector](https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-use-pgvector) · [Optimize pgvector](https://learn.microsoft.com/en-us/azure/postgresql/extensions/how-to-optimize-performance-pgvector) · [PgBouncer](https://learn.microsoft.com/en-us/azure/postgresql/connectivity/concepts-pgbouncer) · [Microsoft sample: PostgreSQL + pgvector (Python)](https://learn.microsoft.com/en-us/samples/azure-samples/azure-postgres-pgvector-python/azure-postgres-pgvector-python/)

### 2.4 Azure Managed Redis: cache + vectors

**Know:**
- Low-latency in-memory operations. Typical **cache-aside**: lookup cache → on miss query primary data source → cache response with TTL → invalidate/update after source changes.
- `SET`/`GET` store/retrieve; `EXPIRE`/`TTL` govern expiration; `DEL` invalidates. **Expiration** (time), **eviction** (memory pressure), and **invalidation** (application-driven) are distinct.
- Managed Redis can act as an AI vector store using **RediSearch** on hashes or JSON documents, with embeddings and metadata.
- **Vector index choices:** `FLAT` = exact; `HNSW` = approximate. Supports nearest-neighbor search, metadata filters, and cosine/L2/inner-product metrics.
- **Provisioning trap:** RediSearch must be enabled at creation, with a supported tier/Enterprise clustering policy; you cannot add the module later to an existing instance. Microsoft's vector guidance recommends **NoEviction** for RediSearch.

```redis
SET ai:summary:doc123 "cached response" EX 300
GET ai:summary:doc123
TTL ai:summary:doc123
DEL ai:summary:doc123
```

**Exam traps:** A cache is not automatically the authoritative system of record. Redis vector search requires appropriate indexing/module configuration; it isn't simply a normal `GET` over embedding arrays.

**Official docs:** [Azure Managed Redis overview](https://learn.microsoft.com/en-us/azure/redis/overview) · [Vector similarity search](https://learn.microsoft.com/en-us/azure/redis/overview-vector-similarity) · [Redis vector storage training](https://learn.microsoft.com/en-us/training/modules/implement-vector-storage-azure-managed-redis/) · [Redis data-loss troubleshooting](https://learn.microsoft.com/en-us/azure/redis/troubleshoot-data-loss)

### Vector storage — choose the service

| Scenario | Typical choice |
|---|---|
| High-scale operational JSON + vectors + change feed | Cosmos DB for NoSQL |
| Relational schema, SQL joins, `pgvector`, transactional metadata | PostgreSQL Flexible Server |
| Ultra-low-latency cache, semantic cache, vector memory | Azure Managed Redis |

---

## 3. Connect to and consume Azure services (20–25%)

### 3.1 Azure Service Bus — durable messaging

| Feature | Meaning |
|---|---|
| **Queue** | Competing consumers; each message processed by one consumer |
| **Topic + subscriptions** | Publish/subscribe; separate subscription receives its own copy |
| **Peek-lock** | Lock message while processing; settle it afterward |
| **Complete** | Processed successfully; remove message |
| **Abandon** | Release for redelivery |
| **Dead-letter** | Move poison/invalid message to DLQ for investigation |
| **Defer** | Retain message for later retrieval using sequence number |
| **Sessions** | Ordered processing of related messages (when designed/configured) |

- Under Peek-lock, repeatedly abandoned messages or expired locks eventually exceed `MaxDeliveryCount` (default **10**) and are dead-lettered.
- Queue/subscription **DLQ is a subqueue**, not the same thing as an Event Grid dead-letter destination.
- Use **idempotent processing** and a stable business/message ID because redelivery can happen even when the handler partially succeeded.
- SDK: `azure-servicebus`; prefer Microsoft Entra authentication with `DefaultAzureCredential` and appropriate RBAC roles.

```python
from azure.identity import DefaultAzureCredential
from azure.servicebus import ServiceBusClient, ServiceBusMessage

with DefaultAzureCredential() as credential:
    with ServiceBusClient(
        fully_qualified_namespace="<namespace>.servicebus.windows.net",
        credential=credential,
    ) as client:
        with client.get_queue_sender(queue_name="work-items") as sender:
            sender.send_messages(ServiceBusMessage("process doc-123"))
```

**Official docs:** [Python queue quickstart](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-python-how-to-use-queues) · [Python topics/subscriptions](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-python-how-to-use-topics-subscriptions) · [Dead-letter queues](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues)

### 3.2 Azure Event Grid — events and notifications

- **Event Grid** routes notifications of *things that happened*; **Service Bus** holds work/messages requiring robust broker processing.
- Know **system/custom topics**, **event subscriptions**, filters by **event type**, **subject prefix/suffix**, and **advanced event data**.
- Default delivery is **at least once**, and ordering is **not guaranteed**. Consumers must tolerate duplicates and out-of-order events.
- Event Grid retries transient delivery failures; a **dead-letter destination** can store undeliverable events (e.g., Blob Storage for relevant push delivery configurations). Without one, permanently undeliverable events can be dropped.

**Scenario:** Blob uploaded → Event Grid → Azure Function → enqueue durable processing in Service Bus.

**Official docs:** [Event filtering](https://learn.microsoft.com/en-us/azure/event-grid/event-filtering) · [Delivery and retry](https://learn.microsoft.com/en-us/azure/event-grid/delivery-and-retry)

### 3.3 Azure Functions — triggers, bindings, deployment

- **Trigger:** one trigger per function (HTTP, timer, Service Bus, Event Grid, etc.).
- **Input/output bindings:** declarative connections to external resources; code can also directly use an Azure SDK.
- **Python v2** programming model: decorators in `function_app.py`. Python v1 uses `function.json`.
- Deployment depends on hosting plan; know function-app **application settings**, `host.json`, managed identity, triggers, logs, and retry/error behavior.

```python
# function_app.py — Azure Functions Python v2
import azure.functions as func
import logging

app = func.FunctionApp(http_auth_level=func.AuthLevel.FUNCTION)

@app.route(route="health", methods=["GET"])
def health(req: func.HttpRequest) -> func.HttpResponse:
    return func.HttpResponse("OK", status_code=200)

@app.service_bus_queue_trigger(
    arg_name="msg", queue_name="work-items", connection="ServiceBusConnection"
)
def process_work_item(msg: func.ServiceBusMessage):
    logging.info("Received work item: %s", msg.get_body().decode("utf-8"))
```

`ServiceBusConnection` above identifies the *name of the connection configuration* (connection string setting or identity-based setting prefix); it is not a hard-coded credential.

**Exam traps:** Trigger ≠ binding; changing the trigger may change delivery/retry semantics. With a broker trigger, investigate repeated failures and DLQ messages rather than assuming an HTTP-style retry response.

**Official docs:** [Python Functions developer guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python) · [Service Bus trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-service-bus-trigger) · [Azure Functions documentation](https://learn.microsoft.com/en-us/azure/azure-functions/)

### Messaging decision shortcut

| Need | Best fit |
|---|---|
| Reliable work queue, competing consumers, DLQ, sessions | **Service Bus queue** |
| Broadcast durable work to independent consumers | **Service Bus topic + subscriptions** |
| Route reactive event notification with filters | **Event Grid** |
| Execute event-triggered code | **Azure Functions** |

---

## 4. Secure, monitor, troubleshoot (20–25%)

### 4.1 Key Vault, identity, and secret rotation

- **Key Vault:** secrets, keys, certificates; retrieve versioned secrets via SDK or service references.
- **Managed identity + RBAC** avoids embedding service credentials in code. Assign the needed **data-plane** role and verify network access separately.
- Use secret **versions**, **rotation**, and dependent application refresh. A newly rotated Key Vault secret is not necessarily immediately used by every running container.
- Rotation designs may use **Key Vault events → Event Grid → Function → update target credential → store new secret version**.

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

vault = SecretClient(
    vault_url="https://<vault-name>.vault.azure.net/",
    credential=DefaultAzureCredential(),
)
api_key = vault.get_secret("third-party-api-key").value
# Never log the value.
```

**Official docs:** [Python Key Vault secrets](https://learn.microsoft.com/en-us/azure/key-vault/secrets/quick-create-python) · [SecretClient API](https://learn.microsoft.com/en-us/python/api/azure-keyvault-secrets/azure.keyvault.secrets.secretclient) · [Dual-credential rotation example](https://learn.microsoft.com/en-us/azure/key-vault/secrets/tutorial-rotation-dual)

### 4.2 Azure App Configuration

**Separation of concerns:**
- **App Configuration:** central nonsecret app settings, labels by environment, feature flags, dynamic refresh. Can reference Key Vault secrets.
- **Key Vault:** secure secret values and their lifecycle. Do not use App Configuration as a substitute secret store.
- Python provider: `azure-appconfiguration-provider`; load via `DefaultAzureCredential`, enable refresh, and explicitly call `config.refresh()` as part of the application's activity path.

```python
from azure.appconfiguration.provider import load
from azure.identity import DefaultAzureCredential

config = load(
    endpoint="https://<store>.azconfig.io",
    credential=DefaultAzureCredential(),
    refresh_enabled=True,
)
# Later, during app activity:
config.refresh()
```

**Exam trap:** `refresh_enabled=True` does not mean changes are pushed instantly without application activity or an explicit refresh call. Labels distinguish configurations, and feature flags need explicit loading/configuration.

**Official docs:** [Python App Configuration provider](https://learn.microsoft.com/en-us/azure/azure-app-configuration/reference-python-provider) · [Feature flags in Python](https://learn.microsoft.com/en-us/azure/azure-app-configuration/quickstart-feature-flag-python)

### 4.3 OpenTelemetry (OTel) and distributed tracing

- **Trace:** full request journey; **span:** timed individual operation within a trace.
- Propagate context across HTTP/messaging to correlate services. Common attributes include **trace ID**, **span ID**, service name, and status/error details.
- **Logs:** individual events; **metrics:** numeric time series; **traces:** distributed causality/latency.
- For new Python apps, Microsoft's **October 2026** guidance points to the **Microsoft OpenTelemetry Distro** (`microsoft-opentelemetry`) for Application Insights export; existing Azure Monitor OpenTelemetry Distro installations are also supported.
- Check sampling, role/service names, exporters, and whether telemetry has actually reached Application Insights.

```python
# Configure APPLICATIONINSIGHTS_CONNECTION_STRING as an environment setting.
from microsoft.opentelemetry import use_microsoft_opentelemetry

use_microsoft_opentelemetry(
    enable_azure_monitor=True,
    logger_name="ai_backend",
)
```

**Exam traps:** Instrumenting a single API gives no automatic end-to-end trace if downstream services fail to propagate context. Logs and metrics alone may not explain which dependency caused one slow user request.

**Official docs:** [OpenTelemetry with Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable?tabs=python) · [Application Insights overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)

### 4.4 KQL — must-know query patterns

**Operator mini-map:**

| Operator | Purpose |
|---|---|
| `where` | Filter rows |
| `project` | Select/rename columns |
| `extend` | Calculate derived columns |
| `summarize` | Aggregate (e.g., count, avg) |
| `bin(TimeGenerated, 5m)` | Group timestamps into intervals |
| `order by` | Sort results |
| `join` | Combine matching rows from different tables |

**Example 1 — failed requests in the last hour** (workspace-based Application Insights tables):

```kusto
AppRequests
| where TimeGenerated >= ago(1h)
| where Success == false
| summarize Failures = count() by Name
| order by Failures desc
```

**Example 2 — average and 95th-percentile latency by interval:**

```kusto
AppRequests
| where TimeGenerated >= ago(24h)
| summarize AvgMs = avg(DurationMs), P95Ms = percentile(DurationMs, 95)
    by bin(TimeGenerated, 15m)
| order by TimeGenerated asc
```

**Example 3 — failed dependencies:**

```kusto
AppDependencies
| where TimeGenerated >= ago(1h)
| where Success == false
| summarize Failures = count() by Target, Name
| order by Failures desc
```

**Example 4 — correlate failures by operation:**

```kusto
AppRequests
| where TimeGenerated >= ago(1h) and Success == false
| project OperationId, RequestName = Name, RequestTime = TimeGenerated
| join kind=leftouter (
    AppDependencies
    | project OperationId, DependencyName = Name, Target, DependencySuccess = Success
  ) on OperationId
| take 50
```

**Exam traps:** `where` filters *rows*, `project` selects *columns*, `extend` computes new ones, `summarize` changes row granularity. Table/field names depend on ingestion mode; classic Application Insights examples often use `requests`/`dependencies` instead of `AppRequests`/`AppDependencies`.

**Official docs:** [Log Analytics tutorial](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-tutorial) · [Azure Monitor log queries](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-query-overview) · [KQL operator reference](https://learn.microsoft.com/en-us/kusto/query/)

---

## 5. Common exam traps — rapid review

1. **ACR ≠ container host:** build and store images in ACR; run in App Service, Container Apps, or AKS.
2. **Container Apps revision ≠ app-wide configuration:** image/scale-rule updates normally make revisions; app-level secrets do not.
3. **KEDA scale-to-zero ≠ CPU autoscale:** event triggers enable scaling of background workers based on queue/event backlog.
4. **AKS CrashLoopBackOff ≠ ImagePullBackOff:** runtime crash vs image retrieval problem.
5. **Cosmos point reads need both `id` and partition key** for efficient routing.
6. **Cosmos RU ≠ storage GB:** inefficient cross-partition scans/index choices can raise RU consumption.
7. **Cosmos vector policy/index is set at creation** in the documented implementation; plan dimensions and index type ahead.
8. **Cosmos change feed processor ≠ Python SDK processor:** .NET/Java processor; Python pull + manual checkpointing.
9. **pgvector extension is called `vector`** and must be allowlisted on Azure PostgreSQL.
10. **Vector distance operator matters:** `<=>` cosine, `<->` Euclidean, `<#>` negative inner product.
11. **Redis expiration ≠ eviction ≠ invalidation**; vector search needs RediSearch configured at provisioning.
12. **Service Bus queue ≠ topic:** competing consumers vs one subscription per independent consumer.
13. **DLQ ≠ automatic repair:** inspect dead-letter reason, fix processing, and deliberately replay.
14. **Event Grid isn't an ordered transactional queue:** duplicates/out-of-order delivery are possible.
15. **Functions trigger ≠ input/output bindings:** Python v2 uses decorators; v1 uses `function.json`.
16. **Managed identity ≠ automatic authorization:** grant RBAC permissions and check networking.
17. **App Configuration ≠ Key Vault:** nonsecret settings/feature flags vs secrets.
18. **A new rotated secret ≠ immediate process refresh**; account for caching and restarts.
19. **OpenTelemetry trace ≠ log:** traces connect spans across services; correlation context must propagate.
20. **KQL `summarize` ≠ `project`:** aggregation versus column selection.

---

## 6. Five focused hands-on labs

Use synthetic data and delete resources afterward to control cost.

- [ ] **Lab 1 — Container delivery:** Build a Python API image using ACR Tasks → deploy to Container Apps → configure environment settings → deploy revision v2 → split traffic or roll back → inspect logs.
- [ ] **Lab 2 — Vector retrieval:** Load 50–200 synthetic snippets into PostgreSQL/pgvector → add HNSW index and tenant metadata index → run cosine search and `EXPLAIN ANALYZE`. Repeat a small subset in Cosmos DB to compare models.
- [ ] **Lab 3 — Cache & vector store:** Implement cache-aside with Azure Managed Redis, TTL and invalidation. If budget permits, provision an instance with RediSearch and try `FLAT`/`HNSW` nearest-neighbor search.
- [ ] **Lab 4 — Events & retries:** Publish a document event to Event Grid → trigger a Function → put work on Service Bus → process it → deliberately fail processing → inspect DLQ, retry/redelivery behavior and idempotency.
- [ ] **Lab 5 — Secure & observe:** Assign a managed identity → read configuration and a Key Vault secret → add OTel tracing → generate successful/failing requests → use four KQL queries to diagnose latency/errors.

**Priority order when short on study time:** PostgreSQL/pgvector + Cosmos vector/change feed → Container Apps revisions/KEDA → Redis vector configuration → Service Bus/Event Grid/Functions distinctions → OpenTelemetry/KQL → ACR and AKS diagnostics.

## 7. Official exam resources

- [AI-200 skills measured — source of truth](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-200)
- [Microsoft certification page](https://learn.microsoft.com/en-us/credentials/certifications/azure-ai-cloud-developer-associate/)
- [AI Skills Navigator — practice assessment](https://aiskillsnavigator.microsoft.com/)
- [Azure documentation homepage](https://learn.microsoft.com/en-us/azure/)
- [Microsoft Learn exam sandbox](https://aka.ms/examdemo)

> **How to use this file:** Read one exam domain, follow its official links, do its corresponding lab, and tick off the objective once you can explain *when to use it*, *how to implement it*, and *how to troubleshoot it* without notes. Recheck the official AI-200 study guide close to your exam date; objectives and guidance can change.
