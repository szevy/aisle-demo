# How Aisle is built

```mermaid
flowchart TB
    U(["Customer"]) --> T1

    T1["Deterministic gates · no model<br/>cart arithmetic · checkout confirm · /stop and documented commands<br/>password reset · order status · rate limits"]
    T1 -->|handled instantly| OUT
    T1 -->|needs language| GUARD

    subgraph GUARD["Guardrails · all run, then one response is chosen"]
        direction LR
        PII[Personal data] --> TOX[Abuse] --> DIS[Distress] --> INJ[Injection]
    end

    INJ -->|"blocked"| STOP(["Refuse and end"])
    INJ -->|clean| CLS

    CLS{{"Intent classifier · ONE call per turn<br/>read by the router and every agent<br/>taught by examples, not keyword rules"}}
    CLS --> RTR{{"LLM router"}}

    RTR -->|policy question| POL["policy_agent"]
    RTR -->|product request| PRO["product_agent"]
    RTR -->|return / cancel| RET["refund_return_agent<br/>state machine · window + status rules in code"]
    RTR -->|"cart · checkout"| TXN["transaction_agent<br/>reservations · totals in code"]
    RTR -->|"greetings · off-topic"| WEB["web_search_agent<br/>bounded tool loop"]
    RTR -->|asks for a person| ESC["human_escalation_agent<br/>offer help once, then hand over"]

    subgraph RAGBOX["RAG · shared retrieval"]
        direction TB
        VDB[("Pinecone<br/>policy + product namespaces")]
        HYB["hybrid retrieve: BM25 + dense<br/>rerank: pinecone → voyage → lexical"]
        VDB --- HYB
    end

    POL --> RAGBOX
    PRO --> RAGBOX
    RAGBOX --> JUDGE{{"Groundedness judge<br/>regenerate once if unsupported"}}
    JUDGE --> OUT
    RET --> OUT
    TXN --> OUT
    WEB --> OUT
    %% Agent-to-agent handoffs: LangGraph edges, capped at 3 hops per turn
    RET -.->|policy question mid-flow| POL
    POL -.->|"[ESCALATE]: not in the documents"| ESC
    ESC -.->|offer accepted, back to self-service| RTR

    ESC --> HUMAN[[Support channel · Slack<br/>replies appear inline · !transfer hands back]]
    HUMAN -->|human reply| OUT

    STATE[(Managed store · Redis<br/>sessions · carts · reservations · transcripts · escalation routes)]
    STATE --- OUT
    OUT(["Answer"]) --> U
    subgraph FALLBACK["Degradation · every stage has a next best, none returns an error page"]
        F1["LLM 429: retries the next provider<br/>cerebras to groq to mistral to google"]
        F2["Reranker quota: circuit breaker opens,<br/>Voyage takes over, then lexical ordering"]
        F3["No BM25 index: hybrid degrades to dense-only"]
        F4["Redis down: in-process memory,<br/>one Slack alert, the shop keeps serving"]
        F5["Artifacts missing: picks fall back<br/>to global popularity"]
    end
    RTR -.->|"any stage fails"| FALLBACK
    classDef guard fill:#fee2e2,stroke:#ef4444,color:#7f1d1d
    classDef llm fill:#ede9fe,stroke:#8b5cf6,color:#4c1d95
    classDef agent fill:#e0f2fe,stroke:#0ea5e9,color:#0c4a6e
    classDef store fill:#dbeafe,stroke:#3b82f6,color:#1e3a8a
    classDef loop fill:#dcfce7,stroke:#22c55e,color:#14532d
    classDef code fill:#fef3c7,stroke:#f59e0b,color:#78350f
    class PII,TOX,DIS,INJ guard
    class CLS,RTR,JUDGE llm
    class POL,PRO,RET,TXN,WEB,ESC agent
    class VDB,STATE,EVAL,TRACE,MET store
    class FB,TRIAGE loop
    class T1,HYB code
```

### How the system improves itself

The system uses feedback loops to monitor performance, trigger alerts, and reallocate resources without manual code changes.

```mermaid
flowchart TB
    OUT(["Answer"]):::io
    PRO["product_agent"]:::agent
    CLS{"intent classifier"}:::llm
    OUT -.-> FB["Not right? · Too slow?"]
    FB -.-> TRIAGE[[Slack triage]]
    TRIAGE -.->|few-shot example + the turn it answered| CLS
    CRON[["GitHub Actions · scheduled<br/>Sun 03:00 UTC: all eval tiers<br/>daily 08:00 UTC: quality digest"]]
    RECRON[["aisle-recommender · scheduled<br/>Sun 01:00 UTC: backtest on the uncapped log,<br/>commits recsys_scores.json here"]]
    RETRAIN["aisle-recommender · Sun 02:00 UTC<br/>retrain segments, PSI drift gate<br/><i>disabled: the Kaggle dataset is static,<br/>so drift never fires and every run is a no-op</i>"]:::off
    RECRON -.->|"publishes the scores the eval reads"| RECEVAL
    RETRAIN -.->|"would publish new segment artifacts"| RECRON
    CRON -.->|"runs, --post alerts on out-of-band"| EVAL
    CRON -.->|"reads Langfuse traces + customer reports"| QUAL[[Quality digest to Slack<br/>what customers reported, what the judges scored]]
    TRIAGE -.->|golden case| EVAL[(Evaluation suite · live<br/>routing · intent · retrieval · groundedness)]
    EVAL -.->|"any metric outside its band"| SLACKALERT[[Slack alert<br/>names the number and what to do]]
    RECEVAL[(Recommender scores · offline<br/>grid differentiation + recall@8 floor<br/>recsys_scores.json)]
    RECEVAL -.->|"read by the recsys tier"| EVAL
    OUT -.-> MET[(Funnel metrics<br/>impressions · clicks · cart-adds per algorithm<br/>order value split by grid-influenced)]
    MET -.->|"cart-add rate per source"| ALLOC[[Slot allocator · automatic<br/>bounded bandit: floor 20% per source,<br/>ignores rates under 200 impressions,<br/>moves at most 10% per 15 min]]
    ALLOC -.->|"how many cards each algorithm gets"| PRO
    OUT -.-> TRACE[("Observability<br/>Langfuse: traces, judge sampling<br/>Prometheus: metrics · Grafana: dashboards")]

    classDef guard fill:#fee2e2,stroke:#ef4444,color:#7f1d1d
    classDef llm fill:#ede9fe,stroke:#8b5cf6,color:#4c1d95
    classDef agent fill:#e0f2fe,stroke:#0ea5e9,color:#0c4a6e
    classDef store fill:#dbeafe,stroke:#3b82f6,color:#1e3a8a
    classDef loop fill:#dcfce7,stroke:#22c55e,color:#14532d
    classDef code fill:#fef3c7,stroke:#f59e0b,color:#78350f
    classDef off fill:#f1f5f9,stroke:#94a3b8,color:#475569,stroke-dasharray:4 3
    classDef io fill:#f8fafc,stroke:#475569,color:#0f172a
    class CLS llm
    class PRO agent
    class EVAL,TRACE,MET store
    class FB,TRIAGE loop
```

The recommender is scored on grid differentiation rather than prediction accuracy. Two
figures are recorded: how many distinct products appear across the four segment grids,
and the overlap between the two most similar segments. Recall@8 is retained as a floor
rather than the objective, because it rewards predicting purchases that would have
happened without a recommendation. The training repository computes both and publishes
them as `recsys_scores.json`, which the weekly evaluation reads.

## The agentic workflow

### Core architectural principle: separation of model and code

The codebase separates deterministic logic from language model interpretation.

**Intent versus action.** The language model identifies user intent, while deterministic code executes the action. Misclassification routes a request to the wrong handler but cannot exceed the handler's explicit permissions.

**Deterministic responsibilities.** Business logic, security, and data integrity reside in code: command execution, checkout confirmation, cart arithmetic, stock reservations, and personal-data detection.

**Model responsibilities.** The model performs semantic interpretation, such as identifying return requests or product search queries. Linguistic variation is managed through prompt examples.

**Structured output enforcement.** Model decisions affecting control flow are constrained to a pydantic schema. The handler receives a validated object. See [Structured output enforcement](structured-output.md) for the four schemas, error handling, and router validation logic.

---

## Customer segmentation

The greeting, login grid, and voucher are determined by the customer's transaction history.

```mermaid
flowchart LR
    T[("H&M transaction log<br/>31.8M rows · 1.36M customers")]
    T --> AGG["Aggregate per customer<br/>one row each"]
    AGG --> F["<b>Seven features</b><br/>recency · frequency · total spend<br/>avg item price · items · online share · age"]
    F --> SC["Standardise<br/>so spend does not drown recency"]
    SC --> KM["<b>k-means</b><br/>fit once, offline"]
    KM --> SEG["Segment label<br/>stored per customer"]

    SEG --> USE1["Greeting<br/>names the segment"]
    SEG --> USE2["Login grid<br/>segment's top sellers,<br/>minus what they own"]
    SEG --> USE3["Voucher minimum<br/>from their own average order"]

    KM -.->|"validated by what segments BUY, not by cluster geometry"| OV["Top-50 article overlap<br/>between every pair"]

    style F fill:#eff6ff,stroke:#3b82f6
    style KM fill:#ede9fe,stroke:#8b5cf6,stroke-width:2px
    style SEG fill:#dbeafe,stroke:#2563eb,stroke-width:2px
    style OV fill:#dcfce7,stroke:#22c55e
```

Segmentation is computed offline and periodically. Login reads the stored label, ensuring no latency for the user.

**Seven features per customer**, computed from the H&M transaction log:

| Feature | What it is |
|---|---|
| Days since last order | Recency |
| Orders placed | Frequency |
| Total spend | Monetary value |
| Average item price | Price sensitivity |
| Items bought | Basket size over time |
| Bought online | Share of online versus in-store orders |
| Age | From the customer record |

This includes the RFM trio (recency, frequency, monetary) plus four features to distinguish between customers with similar spend but different purchasing habits.

**k-means clustering**, fit once and applied at login. This is an unsupervised model as no ground-truth labels exist.

**Cluster evaluation.** Silhouette scores (0.25 at k=2, 0.21 at k=4) and Davies-Bouldin scores are low. These metrics assume data exists as separated blobs. Shopping behavior is a continuum, so partitions naturally score poorly. These metrics describe the data distribution rather than model quality.

Testing 20x more training data (68,185 to 1,362,281 customers), 73 additional product-taste features, and a self-organizing map did not improve silhouette scores.

**Validation.** The model is validated by measuring whether segments purchase different items. Overlap between each pair's 50 most-bought articles:

| Segment pair | Shared articles in top 50 | Overlap |
|---|---|---|
| 0 vs 1 | 1 | 2% |
| 1 vs 3 | 7 | 14% |
| 0 vs 3 | 9 | 18% |
| 1 vs 2 | 12 | 24% |
| 0 vs 2 | 14 | 28% |
| 2 vs 3 | 22 | 44% |

Four of six pairs share under 25% of their top items. Segments with low overlap receive distinct product grids.

**Implementation.** This is a demo surface. A production system would use segments to personalize content without exposing the classification to the user. The segment data is exposed here for inspection, showing the seven features against segment medians.

![The seven features behind the segment, the shopper's values against their segment's medians](../media/profile.png)

---

[← Back to the README](../README.md)