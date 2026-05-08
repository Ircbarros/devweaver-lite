# guide_svelte_advanced

> Advanced Svelte 5 / SvelteKit 2.x patterns — architecture, testing, and real-world patterns.
> Sources: **Advanced Svelte** · **Real-World Svelte** · **Svelte with TDD** · **Frontend Development Beyond React: Svelte**
> Companion: `guide_svelte_patterns.md` (Svelte 5 runes, routing) · `guide_ui_design_pt1.md` + `guide_ui_design_pt2.md`
> Loaded at: IMPLEMENT (ui/ scope) · File size rule: **R-PD-02** (≤ 200 lines)

---

## 1. Advanced Svelte 5 Patterns (AS)

### 1.1 Runes — Advanced Usage

| ID | Rule |
|---|---|
| AS-01 | `$state.raw(value)` for large objects that don't need deep reactivity — skips proxy wrapping, ~3× faster mutations |
| AS-02 | `$state.snapshot(value)` to get a plain, non-reactive clone for serialisation/logging — never `JSON.parse(JSON.stringify(x))` |
| AS-03 | `$derived.by(() => { … })` for multi-line derived computations that can't be expressed as a single expression |
| AS-04 | `$effect.root()` for imperative subscriptions that must survive component unmount (e.g., global hotkeys) — always return cleanup |
| AS-05 | `$effect.pre()` to read and write DOM before the browser paints — required for scroll-position preservation |
| AS-06 | Avoid nested `$effect` — extract to a named function; cascading effects are an architectural smell |

### 1.2 Component Composition

| ID | Rule |
|---|---|
| AS-07 | **Snippets** (`{#snippet name(params)}{/snippet}`) replace named slots — pass snippets as props for composable layouts |
| AS-08 | `{@render snippetName(args)}` to project content; snippets can be passed cross-component as typed props |
| AS-09 | Component `interface` in `<script>` block: declare all props with `$props()` — never use the old `export let` |
| AS-10 | `$bindable()` marks a prop two-way bindable — use sparingly; prefer event callbacks for clear data flow |
| AS-11 | `class` directive for conditional styling: `class:active={isActive}` — never build class strings with concatenation |
| AS-12 | `use:action` for DOM-first integrations (drag-and-drop, tooltips, focus traps) — keep actions pure and reusable |

### 1.3 Transitions & Animations

| ID | Rule |
|---|---|
| AS-13 | Built-in `fade`, `fly`, `slide`, `scale` cover 90% of cases — custom transitions only for brand-specific motion |
| AS-14 | `in:` + `out:` allow asymmetric transitions (enter slow, exit fast) — always match user's `prefers-reduced-motion` |
| AS-15 | `animate:flip` for FLIP list reordering — never imperative DOM manipulation for list order changes |
| AS-16 | Wrap any transition in `@media (prefers-reduced-motion: reduce)` CSS guard or `$effect` checking `window.matchMedia` |

---

## 2. SvelteKit Architecture (SK)

### 2.1 Rendering Strategy Decisions

| Scenario | Strategy | Rule |
|---|---|---|
| Marketing / SEO pages | SSR or prerendering | `export const prerender = true` in `+page.ts` for fully static |
| Authenticated dashboards | CSR (no SSR) | `export const ssr = false` — avoids leaking session data in HTML |
| Data-heavy real-time views | SSR initial + client hydration | Default SvelteKit behaviour; keep `+page.server.ts` load fast (< 200 ms) |
| AI streaming responses | Streaming `Response` | `return new Response(readable_stream)` from `+server.ts`; never buffer full LLM output |

### 2.2 Data Loading

| ID | Rule |
|---|---|
| SK-01 | `+layout.server.ts` for auth/session; `+page.server.ts` for page data; never fetch in `onMount` what can be loaded server-side |
| SK-02 | `load` functions must be **pure** — no side effects; they may be called multiple times (invalidation) |
| SK-03 | `depends('app:key')` to declare custom invalidation groups; `invalidate('app:key')` to trigger re-load client-side |
| SK-04 | `Promise.all([fetch1, fetch2])` inside `load` for parallel data fetching — sequential awaits add latency directly |
| SK-05 | `error(404, 'Not found')` from `load` — never `throw new Error` untyped; maps to `+error.svelte` automatically |
| SK-06 | SvelteKit form actions for mutations — avoids duplicating API endpoints; progressively enhances with JS off |

---

## 3. Testing with Vitest + Svelte Testing Library (TDD)

### 3.1 Unit Testing Components

| ID | Rule |
|---|---|
| TDD-01 | `@testing-library/svelte` for component testing — test what the user sees, not implementation internals |
| TDD-02 | `render(Component, { props: { … } })` — always pass props explicitly; never rely on default prop values in tests |
| TDD-03 | `getByRole`, `getByLabelText`, `getByText` — never `getByTestId` unless the element has no semantic role |
| TDD-04 | `userEvent` for interactions (click, type) — not `fireEvent`; `userEvent` simulates real browser behaviour |
| TDD-05 | `await screen.findBy…` for async elements (after data loads, after animations) — `getBy` throws synchronously |
| TDD-06 | Test reactive state by asserting the rendered DOM — never inspect `$state` values directly |

### 3.2 Mocking SvelteKit

| ID | Rule |
|---|---|
| TDD-07 | Mock `$app/navigation` (`goto`, `invalidate`) with `vi.mock('$app/navigation')` for navigation assertions |
| TDD-08 | Mock `+page.server.ts` load functions by passing the expected `data` prop directly to the component under test |
| TDD-09 | `msw` (Mock Service Worker) for API mocking in integration and E2E tests — avoids real network calls |
| TDD-10 | Test the **form action** handler in isolation with a plain `Request` object — no running SvelteKit server needed |

---

## 4. Real-World Architecture Patterns (RW)

| ID | Rule |
|---|---|
| RW-01 | **Module barrel files** (`$lib/index.ts`) for clean public API; internal components are not re-exported |
| RW-02 | `$lib/stores/` for cross-component state; never `$state` in a module that is imported by multiple components (shared reference bug) |
| RW-03 | **Universal stores** (plain `$state` object exported from a `.svelte.ts` file) replace Svelte 4 writable stores |
| RW-04 | Split large components: < 200 lines per `.svelte` file; extract data logic to `{name}.svelte.ts` files |
| RW-05 | Lazy-load heavy routes: `const HeavyPage = () => import('./HeavyPage.svelte')` — chunk per feature |
| RW-06 | SvelteKit i18n via `paraglide-js` (Inlang) or `svelte-i18n` — never concatenate translated strings |
| RW-07 | API client in `$lib/api/` — never call `fetch` directly in components or `+page.svelte` files |
| RW-08 | Environment variables: public via `$env/static/public` (compile-time); private via `$env/static/private` (server only) |

---

## 5. Post-Implementation Svelte Checklist

- [ ] Deep reactivity not needed: `$state.raw()` used for large objects (AS-01)
- [ ] All `$effect` calls have a cleanup return; no nested effects (AS-04, AS-06)
- [ ] Named slots replaced with `{#snippet}` + `{@render}` (AS-07, AS-08)
- [ ] Transitions respect `prefers-reduced-motion` (AS-16)
- [ ] Rendering strategy chosen explicitly and documented in task scope (SK section)
- [ ] `load` functions are pure — no side effects (SK-02)
- [ ] Parallel fetches inside `load` with `Promise.all` (SK-04)
- [ ] Tests use `getByRole` / `getByLabelText` — not `getByTestId` (TDD-03)
- [ ] `userEvent` used for interactions (TDD-04)
- [ ] API calls encapsulated in `$lib/api/` — not in components (RW-07)
