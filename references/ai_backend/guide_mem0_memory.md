# guide_mem0_memory

> AI backend reference — AI agent memory security: attack vectors, defense patterns, and secure architecture.
> Sources: mem0.ai/blog/ai-memory-security-best-practices · OWASP Agentic AI Top 10 (ASI06)
> Related: `guide_rag_qdrant_langfuse.md`, `guide_langfuse_observability.md`

---

## Table of Contents

1. [Why Memory Is a Security Target](#1-why-memory-is-a-security-target)
2. [Attack Vectors](#2-attack-vectors)
3. [Defense Patterns](#3-defense-patterns)
4. [Memory Isolation and Access Control](#4-memory-isolation-and-access-control)
5. [Secure Memory Architecture (6 Stages)](#5-secure-memory-architecture-6-stages)
6. [mem0 Security Features](#6-mem0-security-features)

---

## 1. Why Memory Is a Security Target

### MM-01 — Memory poisoning vs prompt injection

| Attack class | Scope | Persistence | OWASP ref |
|---|---|---|---|
| Prompt injection | Single session only | Gone after session ends | LLM01 |
| Memory poisoning | Long-term memory store | Survives session restart + model updates | ASI06 |

> **MM-02** In a memory-enabled system, a successfully injected instruction sits in the database and is retrieved in future sessions — the agent treats attacker-planted content as its own trusted history.

### MM-03 — Shared state amplifies risk
A memory store is shared state that influences every future interaction. Anything that enters it can shape the agent's reasoning indefinitely. Treat the memory store with the same security rigor as a production database.

---

## 2. Attack Vectors

### MM-04 — MINJA: query-only memory injection
**Attack**: Attacker sends crafted queries that guide the agent to generate malicious reasoning patterns. Bridging steps link the attack to future victims' queries in embedding space. Progressive shortening makes entries appear natural.  
**Result**: 95%+ injection success rate (entries stored), 70%+ attack success rate (agent produces attacker-desired output). No elevated privileges required.  
**Defense**: Input sanitization + composite trust scoring.

### MM-05 — AgentPoison: backdoor via knowledge base
**Attack**: Optimized trigger tokens are embedded in the knowledge base. Whenever a user query contains the trigger pattern, malicious demonstrations are retrieved with high probability.  
**Result**: 80%+ attack success rate across healthcare, autonomous driving, and QA agents. Triggers are indistinguishable from benign queries in embedding space — perplexity-based detection fails.  
**Defense**: Memory isolation + cryptographic integrity checks.

### MM-06 — MemoryGraft: experience grafting for behavioral drift
**Attack**: Fabricated "successful experiences" planted in long-term memory. No explicit trigger. The agent's own similarity search surfaces poisoned content naturally.  
**Result**: Agent replicates malicious patterns while believing it follows its proven playbook.  
**Defense**: Anomaly monitoring on retrieval patterns + periodic memory audits.

---

## 3. Defense Patterns

### MM-07 — Input validation before any write
Every piece of data entering the memory store MUST be validated before persistence:
- Content length limit
- PII and credentials scan (emails, API keys, card numbers)
- Known injection pattern detection
- Redact or reject flagged content

```python
import re

SENSITIVE_PATTERNS = [
    r"(?i)(api[_-]?key|secret|password)\s*[:=]\s*\S+",
    r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
    r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b",
]

def validate_memory_content(text: str, max_length: int = 2000) -> str:
    if len(text) > max_length:
        raise ValueError(f"Memory content exceeds {max_length} chars")
    for pattern in SENSITIVE_PATTERNS:
        if re.search(pattern, text):
            raise ValueError("Sensitive data pattern detected in memory content")
    return text
```

### MM-08 — Composite trust scoring on retrieval
Do not rely on a single LLM-based detector — A-MemGuard research shows single detectors miss 66% of poisoned entries. Combine temporal signals, pattern-based filtering, and content analysis into a unified trust score.

```python
def compute_trust_score(entry: MemoryEntry, now: datetime) -> float:
    age_days = (now - entry.created_at).days
    recency_score = max(0.0, 1.0 - age_days / 90)      # decay over 90 days
    pattern_score = 1.0 if not entry.has_suspicious_pattern else 0.2
    consistency_score = entry.embedding_coherence_score
    return 0.4 * recency_score + 0.3 * pattern_score + 0.3 * consistency_score
```

### MM-09 — TTL-based expiration on all entries
Set `TTL` at write time for every memory entry. TTL bounds the persistence window for poisoned entries and prevents stale data from degrading performance.

```python
# mem0 — every add() call should include expiration policy
client.add(
    content,
    user_id=user_id,
    metadata={"ttl_days": 30, "source": "conversation"},
)
```

### MM-10 — Anomaly detection on write patterns
Monitor memory write rates per user per session. A sudden spike in writes, or writes with unusual embedding characteristics, signals a potential injection attempt.

---

## 4. Memory Isolation and Access Control

### MM-11 — Per-user memory namespacing
Every user's memories MUST be scoped at the storage level, not just at retrieval time. Queries for `user_b` must never access `user_a`'s namespace.

```python
# mem0 — user_id scoped by default
from mem0 import MemoryClient

client = MemoryClient(api_key=settings.MEM0_API_KEY)

# Write — scoped to user
client.add("User prefers dark mode", user_id=user_id, agent_id=agent_id)

# Read — only retrieves this user's memories
results = client.search("preferences", user_id=user_id, agent_id=agent_id)
```

### MM-12 — RBAC on memory operations
- Read / write / delete operations each require different role permissions.
- In multi-agent systems, define which agents can read from and write to which memory scopes.
- Use authentication tokens scoped to specific `user_id + agent_id` pairs.

### MM-13 — Forensic snapshots for rollback
Maintain periodic snapshots of the memory store. If poisoning is detected at a specific point in time, restore to the last known-good snapshot rather than manually auditing every entry.

### MM-14 — GDPR compliance requirements
AI memory stores containing personal data fall under GDPR:
- **Article 15** — users can request a full export of their stored memories.
- **Article 16** — users can request correction of inaccurate memories.
- **Article 17** — users can demand deletion (right to erasure).
- **Article 5** — store only data necessary for the agent's function (data minimization).

```python
# Must implement: enumerate + export + delete per user
async def delete_user_memories(user_id: str):
    memories = client.get_all(user_id=user_id)
    for mem in memories:
        client.delete(mem["id"])
```

---

## 5. Secure Memory Architecture (6 Stages)

> Based on OWASP AI Agent Security Cheat Sheet + Sunil et al. (2026) defense framework.

| Stage | Actions |
|---|---|
| **1. Input sanitization** | Validate content length · scan for PII/credentials · detect injection patterns · redact/reject flagged content |
| **2. Scoped storage** | Per-user namespace · cryptographic hash per entry · set TTL at write time |
| **3. Access control** | RBAC on all memory ops · auth tokens scoped to `user_id + agent_id` · multi-agent write permissions explicit |
| **4. Trust-scored retrieval** | Weight by temporal freshness + source reliability + pattern consistency · deprioritize entries below threshold |
| **5. Output validation** | After generation, check for PII leakage, unexpected behavioral patterns, safety guardrail compliance |
| **6. Continuous monitoring** | Log all ops · anomaly detection on write/retrieval patterns · maintain snapshots · periodic audit |

---

## 6. mem0 Security Features

### MM-15 — Built-in isolation
Every `add()` and `search()` call is scoped by `user_id` + optional `agent_id` at the storage layer. One user's memories are never retrieved in another user's context by default.

### MM-16 — Exclusion rules for proactive data filtering
```python
# Prevent sensitive data from ever entering the memory store
client = MemoryClient(
    api_key=settings.MEM0_API_KEY,
    config={
        "excludes": "financial details, personal identification information, API keys"
    }
)
```

### MM-17 — OpenMemory MCP for self-hosted control
For teams requiring full infrastructure control, OpenMemory MCP runs on-premises. Uses SQLAlchemy with parameter binding (no raw SQL injection risk). Maintains `MemoryStatusHistory` and `MemoryAccessLog` tables for full traceability.

### MM-18 — Managed platform compliance
Mem0 Cloud provides: SOC 2 compliance · encryption at rest and in transit · audit logging for all memory operations.

---

## Post-implementation checklist — mem0 memory security

- [ ] Input validated before every memory write: length + PII + injection patterns (MM-07)
- [ ] Composite trust scoring on retrieval; no single-detector reliance (MM-08)
- [ ] TTL set on every memory entry at write time (MM-09)
- [ ] Anomaly detection active for unusual write rate patterns (MM-10)
- [ ] `user_id` + `agent_id` scoping on all `add()` and `search()` calls (MM-11)
- [ ] RBAC enforced on read / write / delete memory operations (MM-12)
- [ ] GDPR: export, correction, and deletion endpoints implemented for users (MM-14)
- [ ] Memory exclusion rules configured for PII and credentials (MM-16)
- [ ] Periodic memory snapshots for forensic rollback capability (MM-13)
