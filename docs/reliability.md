# Reliability

To handle free-tier API limitations, the application degrades gracefully in four stages:

```mermaid
flowchart TB
    CALL([Model call]) --> P1

    P1["Provider 1 · Cerebras"] -->|success| OK([Answer])
    P1 -->|"429 or 20s timeout"| P2

    P2["Provider 2 · Groq"] -->|success| OK
    P2 -->|"429 or 20s timeout"| P3

    P3["Provider 3 · Mistral"] -->|success| OK
    P3 -->|"429 or 20s timeout"| P4

    P4["Provider 4 · Google"] -->|success| OK
    P4 -->|"all four exhausted"| DEG

    subgraph DEG["Degraded operation · no model available"]
        direction TB
        RT["Routing falls back to keywords"]
        IC["Intent falls back to a regex"]
        TPL["Returns, cancellations, escalation<br/>answer from templates · no model needed"]
        NOM["Product search, policy, cart<br/>need a model to compose text"]
    end

    DEG --> BAN["Banner names what still works<br/>and what does not, until the quota resets"]

    P1 -.->|"a failed provider is recorded here"| BRK[("Circuit breaker<br/>per-MINUTE 429: benched 90s<br/>per-DAY quota: benched until the reset")]
    BRK -.->|"the next call skips it and starts at one that answers"| CALL

    style OK fill:#dcfce7,stroke:#22c55e,stroke-width:2px
    style DEG fill:#fef3c7,stroke:#f59e0b
    style BAN fill:#fef2f2,stroke:#fca5a5
    style BRK fill:#dbeafe,stroke:#3b82f6
```

1. **Provider failover:** The system routes requests across four providers, immediately falling over to the next when encountering a rate limit.
2. **Circuit breaker:** Per-minute limits trigger retries within seconds, while daily quota exhaustion pauses requests until the provider's specific reset boundary.
3. **Fail fast:** Request timeouts prevent hung requests by converting stalled provider calls into immediate failovers. Non-retryable errors fail immediately.
4. **Honest degradation:** If all providers are exhausted, the UI displays a banner explaining the limitation rather than serving degraded or silent failures.

## Why there is no exponential backoff on the model call

All provider clients are initialized with `max_retries=0`. While standard practice favors exponential backoff, backoff assumes retrying the same endpoint will eventually succeed. This assumption holds for transient network errors but fails for free-tier rate limits. When one provider is rate-limited, other providers remain idle with available quotas. **Switching is faster than waiting**; the failover chain acts as the retry mechanism. Backoff would only introduce unnecessary latency before an inevitable failure.

This design choice was validated by measurement. Gemini's 429 error includes a `retry_delay` (observed at 57 seconds) that the SDK silently honors twice. With default retries enabled, a rate-limited request stalled for two minutes before failing over. Consequently, product searches took 50 to 61 seconds on 2 August 2026 when all providers were rate-limited. Setting `max_retries=0` ensures an immediate failover to the next provider.

The application pauses in only one scenario: if **all four** providers are rate-limited in a single pass. In this case, the application sleeps once and retries the chain, as per-minute limits resolve in seconds, making a brief delay preferable to a hard failure. Daily quota exhaustion is raised immediately because it cannot resolve quickly.

| Failure | Response | Why not backoff |
|---|---|---|
| One provider 429s | Immediate hop to the next provider | Another quota is available now |
| A provider stalls | 20-second timeout, then hop | A hung socket never recovers on its own |
| All four 429 in one pass | Sleep once, walk the chain again | A per-minute cap clears in seconds |
| A daily quota is exhausted | Raise now, mark the provider down until its reset boundary | Waiting cannot help before the reset |

## Circuit breakers, and what each one protects

Circuit breakers prevent repetitive, predictable failures from wasting user latency.

**Provider breaker:** When a daily quota is exhausted, the system flags the provider as inactive until its specific reset boundary. Subsequent requests bypass the inactive provider, avoiding redundant failure checks.

**Reranker breaker:** Pinecone's monthly rerank limit is 500 requests org-wide. Once exhausted, subsequent calls return 429 errors. The breaker then opens, routing queries directly to Voyage. Measurements show a query takes 0.822s with the breaker closed (probing the exhausted service) versus 0.318s with it open, saving 0.50s per query by avoiding the exhausted provider.

Because the limit resets monthly, a short cooldown is inefficient. A 5-minute cooldown would close the breaker 288 times daily, repeatedly costing users 0.50s to check an un-reset quota. Setting `RERANK_BREAKER_COOLDOWN_SEC` to 6 hours limits probes to 4 times daily, which is sufficient to detect a reset while minimizing latency overhead.

The two reranking stages have distinct failure modes and limits:

| Stage | Limit | Recovers |
|---|---|---|
| Pinecone rerank | 500 per month, org-wide | When the month rolls over |
| Voyage rerank | 3 requests per minute on the free tier | Within seconds |

Voyage enforces a rate limit rather than a hard quota, allowing normal, spaced user traffic to pass. However, concurrent requests from the evaluation suite exceed this limit. Because the suite executes retrieval queries back-to-back, most queries fall back to lexical ordering. The suite logs these occurrences and labels the evaluation scores accordingly, ensuring burst-induced fallbacks do not skew the measured quality of the ranking stack.

Fallback and recovery paths are thoroughly tested to ensure system stability during active failures.

When all LLM providers are exhausted, the application degrades to a defined subset of features. Routing falls back to keyword matching, and intent classification falls back to regular expressions. Common workflows (returns, cancellations, and human agent routing) use static templates and bypass the LLM entirely. Product search retains its full retrieval stack: the vector index embeds queries server-side, the BM25 keyword search runs locally, and the reranker operates as an independent service. The exhausted quota only disables the LLM-generated conversational response and conversational follow-up resolution. Consequently, the product grid displays under a static "Closest matches" label instead of custom prose. Policy questions, which require LLM generation, display a message indicating quota exhaustion and the expected reset time. The UI banner explicitly details this limited-service state.

The screenshot below demonstrates the application operating under complete daily quota exhaustion, showing the active limited-service banner above a functional, interactive product grid.

![The limited-service banner over a working product grid, captured under a real exhausted quota](../media/degraded_banner.png)

---

[← Back to the README](../README.md)