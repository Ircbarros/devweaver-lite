# guide_ui_design_pt2

> Interaction patterns and goal-directed design principles for all UI work — language/framework agnostic.
> Sources: **Designing Interfaces** (Jenifer Tidwell) · **About Face** (Cooper, Reimann, Cronin, Noessel)
> Companion: `guide_ui_design_pt1.md` (visual design + Laws of UX) · `guide_svelte_patterns.md` (Svelte impl)
> Loaded at: IMPLEMENT (ui/ scope) · File size rule: **R-PD-02** (≤ 200 lines)

---

## 1. Designing Interfaces — Canonical UI Patterns (DI)

### 1.1 Navigation Patterns

| ID | Pattern | When to use |
|---|---|---|
| DI-01 | **Hub & Spoke** | Central dashboard with detail panels; back always returns to hub |
| DI-02 | **Two-Panel Selector** | List left, detail right; item selection populates right panel without navigation |
| DI-03 | **Pyramid / Drill-down** | Hierarchical content; breadcrumb always visible for orientation |
| DI-04 | **Overflow Menu (⋮)** | Secondary actions behind icon; max 7 items; label every item explicitly |

### 1.2 Content & Data Patterns

| ID | Pattern | When to use |
|---|---|---|
| DI-05 | **Card Grid** | Homogeneous items (projects, reports) — scannable; consistent card dimensions |
| DI-06 | **Activity Stream** | Chronological events; newest first; load-more or pagination — never infinite scroll for data tables |
| DI-07 | **Dashboard with Metrics** | Key KPIs + trend spark-lines at top; detail tables and filters below |
| DI-08 | **Accordion** | Long content with distinct named sections; one section open by default |
| DI-09 | **Faceted Search** | Multi-attribute filtering sidebar; results update live; show active filter count |

### 1.3 Input & Interaction Patterns

| ID | Pattern | When to use |
|---|---|---|
| DI-10 | **Inline Edit** | Click-to-edit in context; avoid opening a modal for a single-field change |
| DI-11 | **Wizard / Stepper** | Multi-step forms; progress always visible; back always available and non-destructive |
| DI-12 | **Autocomplete** | Search inputs with bounded vocabulary; trigger after ≥ 2–3 characters |
| DI-13 | **Progressive Disclosure** | Show essential fields first; "Advanced ▾" reveals rare or expert options |
| DI-14 | **Contextual Action Panel** | Floating panel for batch operations on selected items; appears only when selection exists |

### 1.4 Feedback Patterns

| ID | Pattern | When to use |
|---|---|---|
| DI-15 | **Progress Bar** | Long operations (uploads, AI generation); always show % or time estimate when knowable |
| DI-16 | **Skeleton Screen** | Content loading; mirrors the exact layout of the arriving content — never a blank space |
| DI-17 | **Toast / Snackbar** | Non-blocking notifications; auto-dismiss in 4–6 s; error toasts persist until dismissed |
| DI-18 | **Inline Validation** | Field-level errors shown **on blur** — not only on form submit |
| DI-19 | **Confirmation Dialog** | Irreversible or high-consequence actions only; message states exact consequence |

---

## 2. About Face — Goal-Directed Interaction Design (AF)

### 2.1 Design Philosophy

| ID | Principle | Rule |
|---|---|---|
| AF-01 | **Goal-directed design** | Design for user *goals* (what they want to achieve), not task lists (what the system requires) |
| AF-02 | **Mental model ≠ implementation model** | UI must reflect the user's mental model — never expose the system's data model or DB structure |
| AF-03 | **App posture** | Sovereign (full-screen): maximise real estate · Transient (modal): in/out fast · Daemonic: invisible unless summoned |
| AF-04 | **Eliminate modes** | Avoid states where the same UI behaves differently — confuses users and creates traps |

### 2.2 Interaction Principles

| ID | Principle | Rule |
|---|---|---|
| AF-05 | **Safe exploration** | Every action is reversible or confirmation-gated; no user should feel trapped by exploration |
| AF-06 | **Immediate feedback** | Every action triggers visible response within **100 ms** — even if background processing continues |
| AF-07 | **Modals sparingly** | Save dialogs for irreversible/high-stakes actions; use inline undo or toast for reversible ones |
| AF-08 | **Direct manipulation** | Prefer drag-and-drop, resize handles, inline controls over separate settings panels |
| AF-09 | **Forgiveness** | Confirm before delete; provide undo for the last **3 steps minimum**; never auto-delete silently |

### 2.3 Orchestration & Flow

| ID | Principle | Rule |
|---|---|---|
| AF-10 | **Visible system status** | Always show what is happening: loading · saving · processing · done (see also LUX-09) |
| AF-11 | **Orchestration** | Multi-step flows: label the current step and show total ("Step 2 of 4") |
| AF-12 | **Progressive disclosure** | Show only what is needed now; reveal advanced complexity on explicit user demand |
| AF-13 | **Error recovery** | Every error message: what happened · why · how to fix — no "Something went wrong" |
| AF-14 | **Consistent affordances** | Clickable-looking controls must be interactive; non-interactive elements must not look clickable |

### 2.4 Personas & Context Sensitivity

| ID | Principle | Rule |
|---|---|---|
| AF-15 | **Persona-first** | Before building a screen, name the primary persona and their goal |
| AF-16 | **Beginner ↔ Expert** | Defaults + tooltips for beginners; keyboard shortcuts + density toggle for experts |
| AF-17 | **Context sensitivity** | Controls irrelevant to current context: hidden or disabled — never removed (breaks affordance) |

---

## 3. Post-Implementation Interaction Checklist

- [ ] Navigation pattern chosen explicitly (DI-01 to DI-04) and documented in task scope
- [ ] Multi-step flows: stepper / progress bar visible; back is always available (DI-11, AF-11, LUX-10)
- [ ] All destructive actions: confirmation dialog or undo provided (RUI-22, AF-07, AF-09, DI-19)
- [ ] Error messages: specific + actionable — no "Something went wrong" (AF-13)
- [ ] Every action: immediate visual response < 100 ms (AF-06)
- [ ] Loading > 400 ms: skeleton screen or progress bar shown (DI-15, DI-16, LUX-09)
- [ ] Inline validation on all form fields — not only on submit (DI-18)
- [ ] Advanced options behind progressive disclosure — basic fields visible by default (DI-13, AF-12)
- [ ] Direct manipulation preferred over modal dialog for simple single-field edits (AF-08, DI-10)
- [ ] All controls: identical appearance = identical behaviour (AF-14)
- [ ] Primary persona and goal documented before screen was designed (AF-15)
