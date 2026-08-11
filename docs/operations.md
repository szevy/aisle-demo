# Deployment and operations

The application deploys as a Docker container on Render via `render.yaml` on git push. The repository includes Kubernetes manifests in `k8s/` (a stateless Deployment with health probes, a Service, ConfigMap, Secret slots, and a Redis pod) to run the same container on a cluster. These manifests are not used in production but demonstrate cluster readiness, as all application state resides in Redis.

**The infrastructure is fully declared.** The `terraform/` directory defines the Pinecone index, Upstash Redis database, and Grafana resources (folder, dashboard, alert rules, Slack contact point, and notification policy). In six months nobody remembers which region was chosen or whether Redis eviction was on. These files are that record, and `terraform apply` rebuilds the stack from them.

Two details are critical when integrating Terraform with existing infrastructure:

**The two stateful stores are imported, pinned, and protected.** The Pinecone index (`aisle-hm`) and the Upstash Redis database (`Aisle`) predate the Terraform config and were imported rather than recreated. Their names are pinned to the live resources (neither platform renames in place, so a name Terraform disagrees with means destroy-and-recreate: the catalogue vectors and every order), and both carry `prevent_destroy` as a backstop. A current plan shows no changes to either, so day-to-day applies are routine.

**The whole metrics path is declared, including the scrape connection.** The dashboard, the alert rules, the Slack contact point, the notification policy and the scrape job that pulls `/metrics` from the deployed app are all Terraform, so no step is performed by hand in a console that nobody can reproduce. The endpoint requires a bearer token, which is both correct hygiene and a Grafana Cloud requirement, since it refuses an unauthenticated URL. See [observability](observability.md) for how the metrics reach Grafana.

```bash
cd terraform
source ./tfvars_from_keyring.sh   # exports TF_VAR_* from a gpg keyring, never a file on disk
terraform plan                    # read before applying, always
terraform apply
```

Two separate secret paths, because Terraform and the running app are provisioned by different systems. Terraform runs from a workstation, and its credentials are decrypted from a gpg keyring into that one shell rather than written to `terraform.tfvars`. The deployed app never sees the keyring: `render.yaml` declares 19 secrets as `sync: false`, which Render prompts for once and then holds in its own environment. Render is already declarative through that file, so wrapping it in Terraform would add a layer doing the same job.

**Langfuse configuration is managed via Python scripts.** Because the Langfuse Terraform provider only manages projects and API keys, it cannot configure evaluators or dashboards. Two Python scripts manage these configurations programmatically. They live in the serving repository, which is private, so the paths below are named rather than linked.

| Script | Declares | Applied with |
|---|---|---|
| `scripts/langfuse_sync.py` | Judge and evaluator settings in `config/langfuse_evaluators.json` | `--apply` (no flag prints the diff) |
| `scripts/langfuse_dashboard.py` | Answer-quality dashboard (scores, spread, and costs) | `--apply` (no flag prints the dry run) |

Both scripts default to a dry run to preview changes. Storing configuration in Git ensures version control, auditability, and disaster recovery, which UI-based edits do not provide.

This custom dashboard tracks assistant response quality over time, supplementing the default Langfuse dashboards that only monitor usage, latency, and cost.

The Langfuse dashboard API separates widget creation from layout placement. Because these endpoints are currently unstable, field names were derived directly from API validation errors.

**Stateless application architecture.** Sessions, carts, stock reservations, escalation routes, and transcripts reside in external storage. This design allows instances to restart, sleep, or scale horizontally without disrupting active user sessions. To maintain data integrity, a static analysis test ensures that each shared data store is written to by exactly one designated module.

## Scaling, and what actually limits it

The container runs a single `uvicorn` process hosting the FastAPI application. Synchronous handlers execute within a thread pool of 40 threads (the AnyIO default). However, actual capacity is constrained by a much lower, deliberate limit.

**Single-concurrency chat execution.** A semaphore limits execution to one active graph turn. Up to 3 additional requests can queue for up to 30 seconds; subsequent requests are immediately rejected with a busy message. This limit is dictated by memory constraints: the application consumes 342 MB at boot, and lazy-loading during the first turn increases usage to the instance limit of 512 MB. Concurrent execution would trigger out-of-memory (OOM) failures, as verified during testing where two concurrent users crashed the worker twice before this limit was implemented.

**Bounded queue prevents thread starvation.** Because queued requests block worker threads, an unbounded queue during traffic spikes previously exhausted the thread pool. This caused health check failures and triggered Render to restart the instance after a 5-second timeout. Restricting the queue size avoids thread starvation and provides immediate feedback to excess requests.

**Single-process execution.** Running multiple worker processes is unfeasible within the 512 MB memory limit, as each process requires its own interpreter and model instances. The guardrail model alone consumes approximately 142 MB of resident memory when loaded.

**Thread pool utility.** Lightweight, I/O-bound operations (orders, cart actions, static files, and health checks) run concurrently in the thread pool while the resource-intensive graph execution is serialized. This concurrency model maximizes CPU efficiency but cannot bypass the physical memory limit of one concurrent graph turn.

In summary, the current instance handles one active conversation turn at a time due to free-tier memory limits rather than architectural constraints.

**FastAPI migration preserved the concurrency model.** Routes were migrated using a frozen 25-probe acceptance baseline (verifying status codes, JSON structures, and deterministic responses) while keeping handler bodies synchronous to maintain identical client behavior. The legacy Flask codebase was removed. Rather than a wholesale rewrite, `async def` is introduced selectively where network I/O bottlenecks exist. This targeted async adoption releases threads during external API calls without risking the stability of CPU-bound routes.

**External API quotas limit scaling.** Scaling instances horizontally increases requests against shared free-tier API keys, leading to HTTP 429 rate limits rather than CPU exhaustion. Consequently, optimizing [token economics](token-economics.md) directly increases system capacity by reducing API calls per request.

**Proven horizontal scalability.** The Kubernetes manifests specify `replicas: 2` to verify stateless operation across multiple instances. A HorizontalPodAutoscaler scales the deployment from 2 to 6 replicas at 70% CPU utilization. Resource requests are set to 200m CPU and 512Mi memory, with limits of 1 CPU and 1Gi.

| Layer | Scales by | Ceiling |
|---|---|---|
| Web tier | Threads, then replicas (HPA, 2 to 6) | Provider rate limits |
| State | Managed Redis | Free-plan command quota |
| Retrieval | Hosted vector index | Monthly rerank limits (with fallback chain) |
| Large artifacts | SQLite disk paging | Disk I/O (no impact on resident memory size) |

**Production omissions.** The current architecture lacks a global request queue, model response caching (exact or semantic), and database read replicas or connection pooling. These omissions are deliberate for demo workloads. Implementing a semantic cache is the highest priority optimization, as it would reduce redundant LLM queries and lower API usage.

**Dataset optimization for constrained memory.** The raw order history and features for 1,362,281 customers totaled 808 MB, exceeding the 512 MB instance limit. Compressing this data into single-row blobs reduced the size to 160 MB with a 0.07 ms read time. Using SQLite prevents memory overhead by paging data from disk. Similarly, storing 105,100 product thumbnails as a single database reduced storage from 465 MB (where 84% of loose files were smaller than one 4 KB block) to 227 MB.

**Observability stack.**

| Layer | Answers |
|---|---|
| Distributed tracing | Execution details per conversation (model calls, tokens, costs, retrieved context) |
| Metrics and dashboards | System behavior across traffic (latency percentiles, failovers, engagement) |
| Judge scoring | Quality of live answers (groundedness and relevance sampled from production) |

Detailed cost metrics and budget caps are documented in [token economics](token-economics.md).

---

[← Back to the README](../README.md)