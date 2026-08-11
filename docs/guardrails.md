# Guardrails

Guardrails execute on user input, during tool calls, and on model outputs. Every input check runs before a response is chosen, and every result is recorded, so a trace shows what each check saw rather than stopping at the first one to fire.

```mermaid
flowchart TB
    IN(["Customer message"])

    subgraph INPUT["Input guardrails · all run, then one response is chosen"]
        direction LR
        P["Personal data<br/>Presidio + patterns"]
        T["Abuse<br/>trained classifier"]
        D["Distress<br/>crisis separated from upset"]
        J["Prompt injection<br/>override attempts"]
    end

    IN --> INPUT
    INPUT -->|"any fires"| REF(["Refuse, log, end the turn"])
    INPUT -->|"clean"| AGENT["Agent"]

    subgraph TOOLS["Tool execution guardrails"]
        direction TB
        MAX["max_iterations = 6<br/>a loop cannot run forever"]
        ERR["Tool errors returned as text<br/>the agent re-plans, the graph survives"]
        SCH["Validated tool schemas<br/>arguments checked before execution"]
        UNT["Retrieved content sanitised + fenced<br/>a web page cannot issue instructions"]
        EFF["No side effects in the tool layer<br/>stock and charges are code, outside the loop"]
    end

    AGENT --> TOOLS
    TOOLS --> OUTG

    subgraph OUTG["Output guardrails"]
        direction LR
        OP["PII masking<br/>on what the model wrote"]
        OT["Toxicity<br/>replaced with a safe fallback"]
        OG["Groundedness<br/>regenerate once if unsupported"]
    end

    OUTG --> OUT(["Answer"])

    style INPUT fill:#fef2f2,stroke:#fecaca
    style TOOLS fill:#fef3c7,stroke:#fcd34d
    style OUTG fill:#f0fdf4,stroke:#bbf7d0
    style REF fill:#fee2e2,stroke:#ef4444,stroke-width:2px
    style OUT fill:#dcfce7,stroke:#22c55e,stroke-width:2px
```

| Check | Catches |
|---|---|
| Personal data | Card numbers, passwords, and contact details, intercepted before model inference |
| Abuse | Profanity and hostile intent, including misspelled insults, using a vocabulary classifier and an intent model |
| Distress and crisis | Emotional distress and self-harm signals, triggering a dedicated response |
| Prompt injection | Attempts to override system instructions |
| Groundedness | Claims unsupported by retrieved context passages |

## What each check runs on, and what happens when it is unavailable

Each guardrail defines its library, model, and fallback mechanism. System failures do not return error pages; instead, missing backends degrade to simpler checks, which the system logs in the execution trace.

### What runs in this deployment

| Check | Library | Model | Next-best when it is unavailable |
|---|---|---|---|
| Personal data | Presidio (analyzer + anonymizer) | spaCy `en_core_web_sm` for NER, plus Presidio's validated recognisers (Luhn check on card numbers) | Fallback regex for secrets (e.g., `password: ...`) runs. If Presidio fails to load, entity detection is disabled and the installation command is logged. |
| Abuse | `alt-profanity-check` (sklearn, no torch) | Its bundled linear model, imported on first use | If the backend is missing, toxicity filtering is disabled and the check fails open. |
| Distress and crisis | Regex plus the intent classifier | The turn's LLM, via the intent call already being made | The regex alone detects explicit crisis phrasing without initiating extra model calls. |
| Prompt injection | Regex over a leetspeak-normalised copy | The LLM guardrail call (`ENABLE_LLM_GUARDRAIL=true` here) | The regex runs independently of external providers; an LLM outage degrades the check but does not disable it. |
| Retrieved web content | The same injection patterns applied to customer input | None. Pattern matching only | The fence still wraps the text. A page whose wording matches no pattern is still labelled as data the model must not obey. |
| Return photos | Structured output via pydantic | Gemini `gemini-3.1-flash-lite`, the only configured multimodal provider | If the API key is missing or the call fails, the photo is rejected, and the system prompts the user again. |
| Groundedness | LLM-as-judge: a second model call scores the answer against the passages it was built from | The cheaper utility provider, not the model that wrote the answer | The check fails open. A judge that errors returns a pass, because a flaky judge must never block a legitimate answer. |

### Stronger libraries the memory limit rules out

The libraries listed above are not the most accurate available. The instance provides 512 MB of memory in total, and the application consumes 342 MB after boot and login. Three stronger alternatives are supported by the code and left uninstalled.

| Not installed | Size | Would replace | Cost of its absence | How to enable |
|---|---|---|---|---|
| Detoxify (`unbiased`) | approximately 1.5 GB, via PyTorch | `alt-profanity-check` for abuse detection | Detoxify scores several dimensions of toxicity from sentence semantics. The sklearn model recognises only its training vocabulary, so hostility phrased without profanity depends on the intent classifier. | Install the package. It is selected automatically at startup if the import succeeds. |
| ONNX injection classifier (`ProtectAI/deberta-v3-base-prompt-injection-v2`) | approximately 440 MB | Nothing. It would be a third layer | Paraphrased injections that match no regular expression depend entirely on the LLM guardrail. | Install the extra, then set `ENABLE_INJECTION_MODEL=true`. |
| spaCy `en_core_web_lg` | approximately 560 MB, against 12 MB for `en_core_web_sm` | `en_core_web_sm` for Presidio's entity recognition | Lower named-entity recall. Card numbers and similar values are unaffected, since Presidio validates those with checksums rather than the language model. | Point the pinned model wheel in `pyproject.toml` at `en_core_web_lg`, then set `SPACY_MODEL=en_core_web_lg`. |

Each is a configuration change rather than a code change, so a deployment with more memory enables it without modifying the application.

**The abuse model is imported on first use rather than at startup** to optimize memory. Unpickling the sklearn model consumes approximately 142 MB of resident memory, which is 25% of the instance capacity. Because abusive input is rare, a word-boundary prescreen determines whether to invoke the model. The prescreen only ever adds work: anything it matches goes to the model, which makes the block or allow decision. Nothing the model would block reaches an agent because the prescreen missed it. Set `TOXICITY_PRESCREEN=false` to bypass the prescreen and run the model on all inputs.

**Names, addresses, and phone numbers are intentionally permitted** to avoid disrupting legitimate checkout flows. Only card numbers, national insurance numbers, cryptocurrency addresses, and IP addresses trigger a hard block. Mentions of phrases like "credit card" are allowed, as Presidio's validated recognizers require an actual numerical value to trigger.

## Guardrails on tool calls

Tool execution is bounded and untrusted. Five constraints mitigate specific failure modes:

**Execution loops are capped.** Both tool-calling agents are configured with `max_iterations=6` to prevent infinite loops and limit token consumption per turn.

**Tool errors are handled as observations rather than exceptions.** Failed tools return error messages as text to the agent, allowing it to re-plan. For example, a timeout results in a retry rather than a system crash. No exceptions propagate from the tool layer into the execution graph.

**Arguments are validated before execution.** Each tool declares a schema, preventing the model from invoking tools with missing or incorrectly typed arguments.

**Retrieved content is sanitized and fenced before model consumption.** Web results are treated as untrusted, external data. Two layers of defense are applied. First, redaction runs the same injection patterns used on customer input against a normalized copy that decodes leetspeak, ensuring `1gn0re 4ll pr3v10us 1nstruct10ns` is detected. Second, the content is wrapped in `<<UNTRUSTED_WEB_RESULTS>>` markers to instruct the model to treat the block as data to summarize rather than instructions to execute. This mitigates paraphrased injections. Sanitization and fencing are managed within the guardrails module, ensuring new patterns protect both customer input and web retrieval paths simultaneously.

**Tools are stateless and cannot cause side effects.** Tools accept inputs as arguments and do not modify shared state, such as carts, sessions, or inventory. State-changing operations, such as reserving stock or processing payments, execute outside the tool layer and require explicit user confirmation. The model cannot modify inventory or process transactions through tool execution alone.

Model outputs undergo the same PII and toxicity checks as inputs to prevent the model from echoing sensitive data, such as card numbers masked in the input phase.

Pasted card numbers are masked before model inference, and abusive input triggers an immediate refusal:

<p>
<img src="../media/guardrail_pii.png" alt="A pasted card number is masked before any model sees it" width="49%">
<img src="../media/guardrail_toxic.png" alt="Abuse is blocked with a firm refusal" width="49%">
</p>

Prompt injection attempts are rejected outright rather than escalated or partially executed:

![A prompt-injection attempt is refused outright](../media/guardrail_injection.png)

Crisis and distress signals bypass standard ticketing and receive direct, supportive responses with crisis resources:

![A distress message gets a direct, human answer rather than a ticket](../media/guardrail_distress.png)

Guardrail precedence is explicitly defined: crisis outranks abuse, and abuse outranks distress. This prioritization ensures appropriate responses; for example, an angry customer using profanity requires a different response than a user in crisis. Distress is addressed immediately rather than escalated to support queues to ensure timely assistance.

Guardrails evaluate each message independently without historical context. A user who sent an abusive message previously will receive a normal response to a subsequent valid query, preventing persistent blocking based on past turns:

![After an abusive message, the next ordinary request is answered normally](../media/guardrail_after_abuse.png)

---

[← Back to the README](../README.md)