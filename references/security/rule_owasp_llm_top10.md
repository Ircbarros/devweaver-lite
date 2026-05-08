# rule_owasp_llm_top10

> OWASP Top 10 for LLM Applications — v1.1 (2025 Edition)  
> Mandatory compliance reference for the `ai/` module and all LLM-adjacent code.

---

## Table of Contents

1. [LLM01 — Prompt Injection](#llm01--prompt-injection)
2. [LLM02 — Sensitive Information Disclosure](#llm02--sensitive-information-disclosure)
3. [LLM03 — Supply Chain](#llm03--supply-chain)
4. [LLM04 — Data & Model Poisoning](#llm04--data--model-poisoning)
5. [LLM05 — Improper Output Handling](#llm05--improper-output-handling)
6. [LLM06 — Excessive Agency](#llm06--excessive-agency)
7. [LLM07 — System Prompt Leakage](#llm07--system-prompt-leakage)
8. [LLM08 — Vector & Embedding Weaknesses](#llm08--vector--embedding-weaknesses)
9. [LLM09 — Misinformation](#llm09--misinformation)
10. [LLM10 — Unbounded Consumption](#llm10--unbounded-consumption)

---

## LLM01 — Prompt Injection

User input overrides system instructions; attackers bypass guardrails.

**Mitigations:**
- Separate user content from system instructions with structural delimiters.
- Validate user input against known injection keywords before forwarding to LLM.
- Use LangGraph's `interrupt_before` to gate high-privilege actions.

```python
INJECTION_PATTERNS = ["ignore", "forget", "system prompt", "override", "jailbreak"]

def sanitise_prompt(user_input: str) -> str:
    lowered = user_input.lower()
    if any(p in lowered for p in INJECTION_PATTERNS):
        raise ValueError("Potential prompt injection detected")
    return user_input.strip()
```

**Primary module at risk**: `ai/`

---

## LLM02 — Sensitive Information Disclosure

Model leaks PII, API keys, or secrets from training data or context window.

**Mitigations:**
- Redact PII before storing in RAG knowledge base or mem0.
- Never include secrets in LLM context; use references instead.
- Monitor LLM outputs for regex patterns matching secret formats.

```python
import re
PII_PATTERNS = {
    "email": r'\b[\w.%+-]+@[\w.-]+\.[A-Z|a-z]{2,}\b',
    "api_key": r'(?i)(api[_-]?key|secret)["\'\s:=]+([A-Za-z0-9_\-]{20,})',
}
def redact_pii(text: str) -> str:
    for label, pattern in PII_PATTERNS.items():
        text = re.sub(pattern, f"[REDACTED_{label.upper()}]", text)
    return text
```

**Primary module at risk**: `ai/`, `storage/`

---

## LLM03 — Supply Chain

Compromised models, plugins, or training data inject malicious behaviour.

**Mitigations:**
- Pin all ML dependencies to exact versions in `requirements.txt`.
- Generate CycloneDX SBOM on every release; upload signed copy to Garage.
- Verify model artefact hashes before loading; use `cosign` for signed models.

**Primary module at risk**: `ai/` (LangGraph tools, plugin integrations)

---

## LLM04 — Data & Model Poisoning

Training or RAG data is corrupted to insert backdoors or biases.

**Mitigations:**
- Validate RAG documents with entropy checks before embedding (low entropy = suspicious).
- Never accept raw user-submitted documents into the knowledge base without review.
- Monitor model output distribution for unexpected shifts (Grafana alert).

```python
def entropy_check(text: str, min_entropy: float = 3.0) -> bool:
    import math
    probs = [text.count(c)/len(text) for c in set(text)]
    e = -sum(p * math.log2(p) for p in probs if p > 0)
    return e >= min_entropy
```

**Primary module at risk**: `ai/`, `storage/`

---

## LLM05 — Improper Output Handling

LLM outputs executed or rendered without sanitisation; enables XSS and injection.

**Mitigations:**
- Escape LLM text before HTML rendering; never use `{@html}` with unescaped LLM output.
- Never execute LLM-generated SQL directly; use pre-approved parameterised templates.
- Validate Pydantic output model on all LLM-generated structured data.

```python
from html import escape
def safe_render(llm_output: str) -> str:
    return escape(llm_output)  # Before any HTML context
```

**Primary module at risk**: `web/` (rendering), `ai/` (tool execution)

---

## LLM06 — Excessive Agency

Agent granted more tools or permissions than needed; compromised prompt causes damage.

**Mitigations:**
- Each MCP server exposes only the minimum required tools (see `guide_mcp_servers_pt1.md`).
- Destructive MCP operations require explicit user approval (`human_approval_required=True`).
- LangGraph `interrupt_before` gates all irreversible nodes.

**Primary module at risk**: `ai/` (LangGraph tool calls)

---

## LLM07 — System Prompt Leakage

Attackers extract system prompt through crafted queries.

**Mitigations:**
- Filter LLM responses: if output contains system-prompt markers, regenerate.
- Never echo system prompt contents in error messages.

```python
LEAK_MARKERS = ["you are a", "system prompt", "instructions:", "your role is"]
def check_for_leakage(response: str) -> str:
    if any(m in response.lower() for m in LEAK_MARKERS):
        return "[Response redacted for security]"
    return response
```

**Primary module at risk**: `ai/`

---

## LLM08 — Vector & Embedding Weaknesses

RAG stores sensitive docs without access control; embedding manipulation poisons results.

**Mitigations:**
- Enforce `user_id` / role scoping on every Qdrant search filter.
- Remove sensitive metadata from documents before embedding.
- Audit Qdrant payload for secrets via pre-index hook.

**Primary module at risk**: `ai/` (RAG), `storage/` (Qdrant)

---

## LLM09 — Misinformation

LLM generates plausible false information; users trust without verification.

**Mitigations:**
- Attach source citations to every RAG-generated response.
- Implement faithfulness scoring (RAGAS or similar) in the evaluation pipeline.
- Display confidence score and source list in UI.

**Primary module at risk**: `ai/`, `web/` (UI display)

---

## LLM10 — Unbounded Consumption

No rate or token limits; attackers exhaust resources or budget.

**Mitigations:**
- Token cap per request (2 000 tokens max input; configurable via env).
- `slowapi` rate limiting on all LLM endpoints (10 req/min per user).
- Cost monitoring in LangFuse; ntfy alert when daily budget exceeded.

```python
MAX_INPUT_TOKENS = int(os.getenv("LLM_MAX_INPUT_TOKENS", "2000"))
if count_tokens(query) > MAX_INPUT_TOKENS:
    raise HTTPException(400, "Query exceeds token limit")
```

**Primary module at risk**: `web/` (API), `ai/` (inference)
