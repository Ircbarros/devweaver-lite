# rule_documentation_standards

> **Scope**: Standards for agent-generated documentation of **deployed projects**.
> This is about docs the agent creates FOR the projects it builds - not the AI builder's own docs.
> **Templates**: `templates/template_doc_readme.md` · `templates/template_doc_runbook.md`
> **File limit**: ≤ 200 lines (R-PD-02)

---

## §1 Purpose

The agent produces and maintains living documentation for the three audiences who use every project it builds:

| Audience | Documents | Created when |
|---|---|---|
| **End Users** | `README.md` · `docs/getting-started/` · user guides | SCRATCH · major feature |
| **Developers** | `docs/architecture/overview.md` · `docs/api/reference.md` · `CONTRIBUTING.md` | SCRATCH · interface change |
| **Operators** | `docs/runbooks/{component}-ops.md` · `docs/guides/deployment.md` · `CHANGELOG.md` | Every deploy-affecting change |

---

## §2 Folder Structure (Docs-as-Code)

```
{project_root}/
├── README.md                           ← audience: all · required always
├── CHANGELOG.md                        ← every public behaviour change
├── CONTRIBUTING.md                     ← once; updated on process change
└── docs/
    ├── getting-started/
    │   ├── installation.md             ← OS/container setup steps
    │   ├── quick-start.md              ← working example in < 5 minutes
    │   └── configuration.md            ← env vars · config file reference
    ├── architecture/
    │   └── overview.md                 ← C4 L1+L2 from ArchitectureDocRecord (C-20)
    ├── api/
    │   └── reference.md                ← endpoint table · request/response · auth · errors
    ├── guides/
    │   ├── deployment.md               ← step-by-step deploy to each target env
    │   └── security.md                 ← secrets management · OWASP controls applied
    ├── runbooks/
    │   └── {component}-ops.md          ← from templates/template_doc_runbook.md
    └── troubleshooting.md              ← symptom → cause → fix
```

- **R-DS-01**: Every file in `docs/` uses relative internal links only - no absolute self-referencing URLs.
- **R-DS-02**: Every file in `docs/` is ≤ 200 lines; split with `{name}_pt{N}.md` if exceeded.
- **R-DS-03**: Mermaid diagrams in `docs/architecture/` follow `rule_diagram_standards.md`.

---

## §3 When to Create / Update

| Trigger | Files affected | LangGraph phase |
|---|---|---|
| SCRATCH - new project | Full `docs/` skeleton · `README.md` · `CHANGELOG.md` | `post_task_node` |
| New or changed API endpoint | `docs/api/reference.md` | `post_task_node` |
| Architecture change (ADR approved) | `docs/architecture/overview.md` | `post_task_node` |
| Deployment config change | `docs/guides/deployment.md` · relevant runbook | `post_task_node` |
| Public behaviour change | `CHANGELOG.md` entry | `post_task_node` |
| 3+ IssueRecords with same resolution | `docs/troubleshooting.md` new entry | `post_task_node` |
| Developer explicit request | Any | `post_task_node` (doc-only task) |

**Rule**: Documentation ships in the **same commit as the code change** - never deferred to a future task.
Internal refactors do not appear in `CHANGELOG.md`; they belong in the commit message only.

---

## §4 Writing Style

### §4.1 Audience-First (Atlassian · devdynamics)

Before writing any section, name the reader role: **End User · Developer · Operator**.
Use second person ("you install…", "you run…") for all instructions. Use active voice for decisions and actions.
Tailor vocabulary and example granularity to that role - operators need commands, users need concepts.

### §4.2 Narrative Arc - Need → Obstacle → Solution

Every guide, runbook, and troubleshooting entry follows a three-beat structure:

| Beat | Question answered | Source |
|---|---|---|
| **Need** | Why does the reader need this? What is their goal? | Truby - _The Anatomy of Story_: protagonist + desire |
| **Obstacle** | What goes wrong or blocks them without this? | Truby - opponent; Dicks - "what changed?" |
| **Solution** | Concrete, step-by-step resolution with working example | Lamott - _Bird by Bird_: one task at a time |

**Causality rule** (Storr - _The Science of Storytelling_): connect every step with "because / therefore", never "and then". The brain retains causal chains, not sequential lists.

**Specificity rule** (Goldberg - _Writing Down the Bones_): write concrete specifics - "run `docker compose up -d api`", not "start the service". First draft first, then prune.

**5-second moment** (Dicks - _Storyworthy_): every doc worth writing delivers one insight the reader didn't have before - name it in the opening sentence.

**Start in action** (The Moth · Nieman Storyboard): open with the reader already in their context - "When your container fails to start…" - not with background history.

### §4.3 Deep Modules - Interface over Implementation (Ousterhout)

- Document the **interface**: what the user can do, how to invoke it, what it returns, what errors it throws.
- Never expose implementation details in user-facing docs - that is ADR territory.
- "Why" belongs in ADRs; "What I can do" belongs in API reference; "How to do it" belongs in guides.
- A correct abstraction at the right level is worth more than exhaustive detail at the wrong level.

### §4.4 Conciseness Rules (Google · Write the Docs)

| Rule | Applies to |
|---|---|
| One accurate page > ten stale pages (Minimum Viable Documentation) | All docs |
| Delete dead documentation immediately at `post_task_node` | All docs |
| No TODO / PLACEHOLDER in published sections | All docs |
| Break prose at 4+ consecutive sentences without a list, code block, or header | All prose |
| Every abstract statement has a concrete example within 3 lines | Guides · runbooks |
| First sentence of every section answers "so what?" for the reader | All docs |
| No passive voice in instructions ("It can be configured..." -> "Configure it by...") | All docs |
| No emojis anywhere - not in headings, prose, comments, or docstrings | All text artefacts |
| No em dash (`—`): use a hyphen `-` for a parenthetical break, a colon `:` to introduce a clause, or rewrite as two sentences | All text artefacts |

### §4.5 CHANGELOG Format (Keep a Changelog)

```
## [version] - YYYY-MM-DD

### Added
- One-line user-visible new capability

### Changed
- What changed · why it matters to the user

### Fixed
- Bug resolved · observed symptom

### Removed
- Feature removed · migration path
```

---

## §5 Architecture Documentation in the Deployed Project

When architecture is undocumented or mode = SCRATCH, the agent creates **two documents**:

1. `references/architecture/{component}.md` - internal ADR record per C-20 (ai builder's knowledge base)
2. `docs/architecture/overview.md` - project-facing architecture document for developers using the project

Both contain C4 L1+L2 diagrams using `flowchart LR` (R-DS-03). The project-facing doc (`docs/architecture/overview.md`) is diagram-primary - the C4 diagrams ARE the architecture explanation; prose annotates, it does not replace. Keep prose to ≤ 3 sentences per diagram. Link from `README.md` Architecture section.

---

## §6 Node & Pipeline Integration

| LangGraph node | Documentation action |
|---|---|
| `pre_task_node` | Load `docs_context` from Librarian - identify existing doc files to determine update scope |
| `architecture_node` | Draft `docs/architecture/overview.md` when `ArchitectureDocRecord.status = DRAFT` |
| `post_task_node` | Diff committed files → create/update all affected `docs/` files + `README.md` + `CHANGELOG.md` |
| `post_task_node` | If architecture docs updated → trigger `adr_index_pipeline` (ADR-001 §5.3) |
| `file_watcher_pipeline` | Does NOT watch project `docs/` - that is the user's project repo, not the AI builder's workspace |

The **documentation sweep** in `post_task_node` runs in parallel with the code review and architecture
sync sweeps defined in ADR-002 §2.4. It is not a separate phase.

---

## §7 Quality Gates

Gates enforced at `post_task_node` before any docs file is committed:

| Gate | Check |
|---|---|
| **R-DS-04** | No broken relative links (`markdown-link-check` or equivalent) |
| **R-DS-05** | All Mermaid blocks render without errors (`mmdc --validate`) |
| **R-DS-06** | No TODO / PLACEHOLDER markers in Critical sections |
| **R-DS-07** | Every guide and runbook section has a working code example |
| **R-DS-08** | No stale version references - cross-check against `rule_standards_versions.md` |
| **R-DS-09** | Writing follows §4: audience identified · N->O->S arc in guides · conciseness rules applied |
| **R-DS-10** | No emojis and no em dashes (`—`) in any text artefact: docs, comments, docstrings, commit messages, log strings, or headings |
