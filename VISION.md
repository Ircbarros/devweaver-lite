# Vision

## The Problem

AI coding agents are powerful but undisciplined. In practice they exhibit a consistent
set of failure patterns:

- **Context amnesia**: agents forget earlier decisions mid-task and contradict themselves
  across phases, producing code that drifts from the original architecture.
- **Stale knowledge**: agents rely on training-time snapshots of libraries and frameworks.
  They suggest deprecated APIs, outdated configuration patterns, or security practices
  that the relevant project has since revoked.
- **Standard blindness**: agents skip OWASP checks, write functions without tests, use
  inconsistent naming, and flatten module boundaries -- not because they cannot do better,
  but because nothing enforces these requirements session by session.
- **Silent decisions**: agents make architectural choices (ORM selection, auth strategy,
  data layout) without presenting them for review, creating hidden technical debt.

The result is code that runs on first demo but deteriorates on the second sprint.

---

## The Goal

DevWeaver-Lite exists to make structured, standards-compliant development the default,
not the exception, regardless of which AI provider or IDE a developer uses.

It does this by turning the AI assistant into an engineering partner that:

1. **Follows a fixed workflow** -- the 10-phase sequence (PRE-TASK through POST-TASK)
   is not optional. CONFIRM and GIT GATE checkpoints require explicit human approval
   before code is written or committed.

2. **Retrieves live documentation** -- the CARE-Lite chain queries Context7 for
   up-to-date library docs before generating any implementation, so the agent works
   from current API references, not training snapshots.

3. **Enforces embedded standards** -- OWASP LLM Top 10, naming conventions, test
   coverage thresholds, and documentation requirements are loaded as structured rules
   that govern every phase, not as suggestions the agent can ignore.

4. **Explains decisions** -- every architectural choice is presented with the rationale
   before implementation begins. The developer understands what was built and why.

---

## Target Stacks

DevWeaver-Lite is designed to support production-grade development across modern stacks.
Current standard coverage in the Markdown Librarian includes:

| Domain | Tools and frameworks |
|---|---|
| Frontend | Svelte, SvelteKit, TypeScript |
| AI / ML | LangChain, LangGraph, LangSmith |
| Backend | Python, FastAPI, Pydantic |
| Databases | PostgreSQL, SQLite, Redis |
| Infrastructure | Docker, GitHub Actions |
| Security | OWASP LLM Top 10, bandit, pip-audit |
| Testing | pytest, Playwright |
| Documentation | C4 diagrams, ADR, Markdown standards |

The catalog is extensible -- contributors can add domain indexes and reference guides
for any framework following the librarian contribution guide.

---

## Design Principles

**Zero infrastructure.** The skill itself is a Markdown file. No Docker containers,
no databases, no self-hosted services. Drop one file into your provider and you are running.

**Provider-agnostic.** The same 10-phase workflow runs on GitHub Copilot, Cursor,
Windsurf, Claude, Gemini, OpenAI, and any OpenAI-compatible API. The skill adapts to
the provider; the standards do not change.

**Human-gated.** CONFIRM and GIT GATE are hard stops. The agent cannot bypass them.
No code is written without architectural approval. No commit is made without diff review.

**Teach, do not just build.** The agent surfaces the reasoning behind every decision.
A developer who uses DevWeaver-Lite consistently should understand the architecture
of their project, not just receive files.

**Minimal context footprint.** L3 knowledge files are loaded on demand (max 3 per phase).
The CATALOG.md index keeps the base context small while preserving retrieval depth.

---

## Non-Goals

- DevWeaver-Lite is not a replacement for human code review. It raises the floor; it
  does not replace judgment.
- It is not an autonomous agent with unattended write access. Every file write, commit,
  and destructive git operation requires explicit approval.
- It is not a code generation shortcut. The 10 phases take more turns than asking an
  agent to "just write the code" -- that trade-off is intentional.

---

## Roadmap Direction

- Expand the Markdown Librarian with community-contributed domain indexes.
- Add native support for additional providers as the ecosystem evolves.
- Explore a lightweight session-memory layer that preserves architectural decisions
  across chat sessions without requiring Docker-based infrastructure.
- Publish results from projects built with DevWeaver-Lite to validate and iterate
  on the standards library.
