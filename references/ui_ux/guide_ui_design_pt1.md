# guide_ui_design_pt1

> Visual design rules and cognitive UX laws for all UI work — language/framework agnostic.
> Sources: **Refactoring UI** (Wathan & Schoger) · **Laws of UX** (Jon Yablonski)
> Companion: `guide_ui_design_pt2.md` (interaction patterns) · `guide_svelte_patterns.md` (Svelte impl)
> Loaded at: IMPLEMENT (ui/ scope) · File size rule: **R-PD-02** (≤ 200 lines)

---

## 1. Refactoring UI — Visual Design Rules (RUI)

### 1.1 Typography & Hierarchy

| ID | Rule |
|---|---|
| RUI-01 | Use **font weight and color** to establish hierarchy — not only font size |
| RUI-02 | Body: 400–500 weight · Headings: 600–700 · Supporting text: 400 + muted color |
| RUI-03 | Limit sizes to a constrained scale: `12 / 14 / 16 / 20 / 24 / 30 / 36 / 48 px` |
| RUI-04 | Line length: **45–75 chars** for body text — never full-width on large screens |
| RUI-05 | Never pure `#000000` for text; use near-black in HSL e.g. `hsl(220 15% 10%)` |
| RUI-06 | Supporting text on white: minimum `hsl(0 0% 40%)` for 4.5:1 contrast (WCAG AA) |

### 1.2 Color

| ID | Rule |
|---|---|
| RUI-07 | Limit palette: **1 brand · 1–2 accent · 1 neutral scale · 4 semantic** (success/warning/error/info) |
| RUI-08 | Never use grey text on coloured backgrounds — reduce foreground **opacity** instead |
| RUI-09 | Define all colors in **HSL** design tokens — enables systematic lightness/saturation adjustments |
| RUI-10 | Semantic colors: success = green · warning = amber · error = red · info = blue — always consistent |
| RUI-11 | Color reinforces hierarchy; never use it as the **sole** differentiator (WCAG 1.4.1) |

### 1.3 Spacing & Layout

| ID | Rule |
|---|---|
| RUI-12 | **4 px or 8 px base grid** — all spacing must be multiples of the base unit |
| RUI-13 | **More space separates; less space relates** — related elements are closer than unrelated ones |
| RUI-14 | Cards / containers: generous internal padding **16–32 px** — cramped UIs feel broken |
| RUI-15 | Start with too much whitespace and remove; never start dense and try to add space |
| RUI-16 | Align to the grid; any value not on the token spacing scale is a design defect |

### 1.4 Elevation & Depth

| ID | Rule |
|---|---|
| RUI-17 | Use `box-shadow` to signal elevation — not heavy borders; avoid both simultaneously |
| RUI-18 | Only **3 elevation levels**: flat (0) · raised (card/panel) · floating (dialog/dropdown) |
| RUI-19 | Combine background tint + shadow for depth; border alone signals a boundary, not height |

### 1.5 Actions & Components

| ID | Rule |
|---|---|
| RUI-20 | **Button hierarchy**: primary (solid brand) · secondary (outline) · tertiary (ghost) · destructive (red) |
| RUI-21 | **Max one primary action** per screen section — multiple primaries dilute emphasis |
| RUI-22 | Destructive actions require confirmation **or** must be reversible (undo) — never silent delete |
| RUI-23 | **Empty states must be designed**: icon + descriptive message + primary call-to-action |
| RUI-24 | Icons and illustrations must match the visual weight of adjacent text |
| RUI-25 | Every interactive element must have a **:hover, :focus-visible, and :active** state |

---

## 2. Laws of UX — Cognitive & Interaction Laws (LUX)

| ID | Law | Design rule |
|---|---|---|
| LUX-01 | **Fitts's Law** | Touch targets ≥ 44 × 44 px; place related actions close together |
| LUX-02 | **Hick's Law** | ≤ 7 choices per decision; group and progressively disclose to reduce cognitive load |
| LUX-03 | **Jakob's Law** | Prefer familiar patterns; novelty should serve the user goal, not the designer's preference |
| LUX-04 | **Miller's Law** | Chunk information into groups of 5–9; never > 9 ungrouped items in a list or menu |
| LUX-05 | **Postel's Law** | Accept varied input formats (dates, phone numbers); emit consistent, validated data |
| LUX-06 | **Serial Position** | Most important actions: first or last position — middle items are least noticed and recalled |
| LUX-07 | **Von Restorff** | One visually distinct element per screen = primary CTA; overuse eliminates the effect |
| LUX-08 | **Aesthetic-Usability** | A polished interface is perceived as easier to use — visual quality is a functional investment |
| LUX-09 | **Doherty Threshold** | User-initiated actions: respond visually within **400 ms**; show skeleton/spinner if longer |
| LUX-10 | **Zeigarnik Effect** | Multi-step flows need a visible progress indicator — users engage more when they see progress |
| LUX-11 | **Peak-End Rule** | Users remember the best moment and the final moment; make success states and final screens excellent |

---

## 3. Post-Implementation Visual Checklist

- [ ] All colors: HSL design tokens — no hardcoded hex in components (RUI-07 to RUI-09)
- [ ] Typography: sizes from constrained scale · weight differentiation used for hierarchy (RUI-01 to RUI-03)
- [ ] Spacing: all values on 4/8 px grid · related elements visually grouped (RUI-12 to RUI-15)
- [ ] Max one primary button per section · destructive actions confirmation/undo (RUI-20 to RUI-22)
- [ ] Empty states designed: icon + message + CTA (RUI-23)
- [ ] All interactive elements have :hover :focus-visible :active states (RUI-25)
- [ ] Touch targets ≥ 44 × 44 px (LUX-01)
- [ ] ≤ 7 items per menu/list without grouping (LUX-02, LUX-04)
- [ ] Primary action visually distinct — used once per section (LUX-07)
- [ ] Loading > 400 ms: skeleton screen or spinner shown (LUX-09)
- [ ] Multi-step flows: progress indicator visible (LUX-10)
