# guide_svelte_patterns

> Stream D — Svelte 5 / SvelteKit 2.x implementation patterns for the full-stack AI app front-end.
> **Design rules**: load `guide_ui_design_pt1.md` (visual design + Laws of UX) and
> `guide_ui_design_pt2.md` (interaction patterns + goal-directed design) before implementing any screen.
> **Advanced patterns**: load `guide_svelte_advanced.md` (runes, TDD, architecture, real-world patterns).
> All component work must satisfy those rules in addition to the Svelte patterns below.

---

## Table of Contents

1. [Svelte 5 Runes](#1-svelte-5-runes)
2. [SvelteKit Routing & Load Functions](#2-sveltekit-routing--load-functions)
3. [Vite Code Splitting](#3-vite-code-splitting)
4. [WCAG 2.2 Accessibility](#4-wcag-22-accessibility)
5. [XSS Prevention & CSP](#5-xss-prevention--csp)
6. [Async UI State Patterns](#6-async-ui-state-patterns)
7. [Design Tokens](#7-design-tokens)
8. [Stable Versions](#8-stable-versions)

---

## 1. Svelte 5 Runes

**`$state`** — mutable reactive variable:
```svelte
<script>
let count = $state(0);
let user = $state({ name: "Alice", role: "admin" });
</script>
<button onclick={() => count++}>{count}</button>
```

**`$derived`** — computed value; recalculates when dependencies change:
```svelte
<script>
let items = $state([]);
let total = $derived(items.reduce((sum, i) => sum + i.price, 0));
</script>
```

**`$effect`** — side effects; runs after DOM updates:
```svelte
<script>
let query = $state("");
$effect(() => {
    if (query.length > 2) fetchSuggestions(query);
});
</script>
```

**When to use which**:
- `$state`: any mutable UI state.
- `$derived`: any computed value derived from state (replaces `$: x = ...`).
- `$effect`: DOM operations, analytics, cleanup-requiring subscriptions.

---

## 2. SvelteKit Routing & Load Functions

**File-system routing**:
```
src/routes/
├── +layout.svelte          # Root layout (nav, auth context)
├── +layout.server.ts       # Server-side layout load (session check)
├── +page.svelte            # Page component
├── dashboard/
│   ├── +page.svelte
│   └── +page.server.ts     # Server-only load (DB queries, secrets)
└── api/
    └── v1/
        └── query/
            └── +server.ts  # API route (replaces FastAPI for simple BFF calls)
```

**Load function choice**:

| Scenario | File | Runs on |
|---|---|---|
| Needs auth session / DB / secrets | `+page.server.ts` | Server only |
| Pure public data, SEO needed | `+page.ts` | Server + client |
| Client-only, no SEO required | `$state` + `fetch` in component | Browser only |

**Never** expose server secrets in `+page.ts`; use `+page.server.ts` for any auth-gated data.

---

## 3. Vite Code Splitting

```typescript
// Lazy-load heavy AI-dashboard routes
const AIChat = () => import("$lib/components/AIChat.svelte");
```

**SvelteKit automatic splitting**: each `+page.svelte` is automatically split.  
**Hard limit**: no route chunk > 150 KB (gzipped); enforce in `vite.config.ts`:

```typescript
import { sveltekit } from "@sveltejs/kit/vite";

export default {
  plugins: [sveltekit()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          if (id.includes("node_modules/chart.js")) return "charts";
          if (id.includes("node_modules/highlight.js")) return "highlight";
        },
      },
    },
  },
};
```

---

## 4. WCAG 2.2 Accessibility

**Required patterns**:
- All interactive elements: explicit `aria-label` or visible label text.
- Focus management on route change: `afterNavigate(() => document.title = pageTitle)`.
- Color contrast ratio ≥ 4.5:1 (normal text), ≥ 3:1 (large text).
- Keyboard navigation: all functionality reachable without mouse.
- Error states: use `aria-describedby` linking input to error message.

```svelte
<input
  id="query"
  aria-label="AI query input"
  aria-describedby={hasError ? "query-error" : undefined}
  aria-invalid={hasError}
/>
{#if hasError}
  <span id="query-error" role="alert">{errorMessage}</span>
{/if}
```

---

## 5. XSS Prevention & CSP

- **Never use `{@html}`** with LLM output or any user-controlled string.
- Use a sanitiser if HTML rendering is unavoidable:

```typescript
import DOMPurify from "dompurify";
const safeHtml = DOMPurify.sanitize(llmOutput, { ALLOWED_TAGS: ["b", "i", "p"] });
```

**CSP in `svelte.config.js`** (SvelteKit 2.x built-in):
```javascript
kit: {
  csp: {
    directives: {
      "script-src": ["self"],
      "style-src":  ["self", "unsafe-inline"],
      "connect-src": ["self", "https://api.yourdomain.internal"],
    },
  },
},
```

---

## 6. Async UI State Patterns

Every async operation must show three states: loading, error, data.

```svelte
<script>
  let result = $state(null);
  let error = $state(null);
  let loading = $state(false);

  async function fetchData() {
    loading = true;
    error = null;
    try {
      result = await api.query(input);
    } catch (e) {
      error = e.message;
    } finally {
      loading = false;
    }
  }
</script>

{#if loading}<Spinner />{/if}
{#if error}<ErrorBanner message={error} />{/if}
{#if result}<ResultCard data={result} />{/if}
```

---

## 7. Design Tokens

All design tokens in `assets/ui_ux/design_tokens.json`; import via CSS custom properties:

```css
/* Generated from design_tokens.json */
:root {
  --color-primary: #1976d2;
  --spacing-md: 1rem;
  --radius-card: 0.5rem;
  --font-body: "Inter", sans-serif;
}
```

**Rule**: No hardcoded hex colors or spacing values in component CSS; always use `var(--token-name)`.

---

## 8. Stable Versions

| Package | Version | Notes |
|---|---|---|
| Svelte | `5.x` | ✅ Stable (Runes are stable in 5.x) |
| SvelteKit | `2.x` | ✅ Stable |
| Vite | `5.x` | ✅ Stable |
| `dompurify` | `3.x` | ✅ Stable |
| TypeScript | `5.4.x` | ✅ Stable |
| `@sveltejs/adapter-node` | `5.x` | ✅ Stable |
