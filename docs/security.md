# Security

The threat model for this shopping assistant addresses three areas: preventing cross-customer data access, mitigating model-driven actions, and securing secrets. This document details data isolation and secrets management. Model action mitigation is documented in [guardrails](guardrails.md).

## Authentication

User accounts use password-based authentication. The application never stores plaintext passwords. The registration process hashes passwords using `werkzeug.security.generate_password_hash` and stores only the resulting digest. Sign-in attempts involve re-hashing the provided password and comparing it against the stored digest.

Sessions use signed cookies via Starlette's SessionMiddleware with `HttpOnly`, `SameSite=Lax`, and `Secure` attributes. The `Secure` flag is configured via `SESSION_COOKIE_SECURE` in the deployment environment. This flag remains disabled in the code default because local development occurs over plain HTTP, where a `Secure` cookie is never transmitted. The cookie stores the user ID and is signed using an environment-supplied key. Rotating this key invalidates all active sessions.

Sign-in attempts are throttled to 10 requests per 5 minutes per account or IP address. This mechanism bounds password guessing attempts by the rate limiter rather than by network latency.

**Email verification is a centralized hard gate.** Unverified accounts are authenticated but restricted to verification and sign-out routes. This restriction is enforced within the single user-resolution function used by all protected routes to prevent omission errors if additional routes are implemented. Page requests redirect to the verification page, while API requests return a structured 403 JSON response.

## Authorization

Of the 23 application routes, 14 require authentication. The remaining 9 public routes include the landing page, registration, sign-in, verification, and the metrics endpoint.

Authorization relies on **derivation, not comparison**. Handlers retrieve the user ID directly from the validated session and scope database queries accordingly, such as `get_user_orders(current_user.id)`. Because no endpoint accepts an arbitrary customer ID from the client, there are no authorization checks to bypass.

This pattern applies to carts, return sessions, vouchers, and transcripts. Each resource is keyed by the session's user ID. Write access for each data store is restricted to a single module, enforced by a static analysis test that scans the codebase.

## Secrets

| Secret | Where it lives | Never |
|---|---|---|
| Provider API keys, Stripe, Langfuse, Slack | Environment variables declared in `render.yaml` without values | Committed, logged, or exposed to users |
| Terraform credentials | Decrypted from a GPG keyring into a single shell environment | Written to `terraform.tfvars` on disk |
| A customer's own model key | In-memory session storage keyed by user ID | Persisted or logged (only the provider name is logged) |
| Password | Not stored | Retained in any recoverable format |

The metrics endpoint requires a bearer token (`METRICS_BEARER_TOKEN`) to satisfy security requirements and Grafana Cloud's scraping specifications. Without this token, traffic volumes, latencies, and token counts would be publicly exposed.

Sensitive input is filtered before reaching the LLM. Credit card numbers and similar patterns are blocked on input and masked on output. Personally identifiable information (PII) like names, addresses, and phone numbers is permitted to facilitate checkout. Implementation details are in [guardrails](guardrails.md).

## Request-level protections

**Rate limiting.** A fixed-window counter per client and route class executes in the request middleware: 20 requests per minute on `/chat` and 10 per 5 minutes on `/login`. These are keyed by session ID when signed in and by IP address otherwise. The `/chat` limit is enforced because each turn consumes LLM provider quota, which is the resource constraint for this deployment. The `/login` limit functions as a brute-force throttle. A request exceeding the limit receives a `429` status code with a `Retry-After` header before any provider is called, ensuring rejected requests consume no quota.

The counters are stored in process memory rather than Redis. Production runs as one uvicorn process on one instance, so there is no secondary worker to share state with, and a Redis-backed counter would consume the free plan's monthly command quota for bookkeeping. A multi-replica deployment would require a shared store. The Kubernetes manifests declare `replicas: 2`, so this transition is a documented prerequisite for scaling.

**CSRF.** A synchroniser token is stored in the signed session and must be provided on every state-changing request. The frontend fetch wrapper attaches this token as an `X-CSRF-Token` header to every same-origin non-GET call. The five HTML forms include the token as a hidden `_csrf` field. `GET`, `HEAD`, and `OPTIONS` requests are exempt because they must not change state. The bearer-token machine endpoints (`/metrics`, health probes) are exempt because they do not use browsers or sessions. `SameSite=Lax` blocks cross-site form POST requests, while the token covers same-site subdomains and clients where the attribute does not apply.

**Dependency scanning.** Dependabot opens weekly grouped pull requests for pip and monthly pull requests for GitHub Actions. A `pip-audit` step in CI reports known CVEs in the pinned dependency set on every push. The audit step is non-blocking to ensure that an advisory without a released fix does not prevent merges until an upstream patch is available.

Both the rate limiter and the CSRF check are covered by tests to ensure bursts are rejected and normal traffic is permitted, and that missing or incorrect tokens are refused. Each mechanism includes a kill switch (`ENABLE_RATE_LIMIT`, `ENABLE_CSRF`) to prevent local tooling and load tests from being throttled by these safeguards.

## What is not implemented

One gap remains open, kept here so the list stays honest.

| Gap | Consequence today | What it would take |
|---|---|---|
| **Rate-limit counters do not survive a restart** | The free instance sleeps after ~15 idle minutes, and limits reset on wake. An attacker pacing requests across sleep cycles is bounded by the instance's own latency rather than the limiter. | A Redis-backed store, at the cost of spending the command quota; worthwhile at the point the app scales past one instance, which is the same point the in-memory counters stop being correct. |

---

[← Back to the README](../README.md)