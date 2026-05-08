# rule_clean_code

> **Clean Code** (Robert C. Martin) + **Zen of Python** applied to ALL code,
> regardless of language, framework, or tool.
> Loaded at every `implement_node`. Enforced in §9 self-check.
> Convention: **C-22** · File size rule: **R-PD-02** (≤ 200 lines)

---

## 1. Zen of Python — All 19 Aphorisms (Language-Agnostic)

| ID | Aphorism | Implementation Rule |
|---|---|---|
| Z-01 | Beautiful is better than ugly | Favour clean, elegant solutions — readability over cleverness |
| Z-02 | Explicit is better than implicit | Reveal intent in names; avoid magic values, implicit returns, hidden side effects |
| Z-03 | Simple is better than complex | Use the simplest structure that correctly solves the problem |
| Z-04 | Complex is better than complicated | If complexity is needed, make it controlled and documentable |
| Z-05 | Flat is better than nested | Max 3 levels of nesting; extract deeper logic into named functions |
| Z-06 | Sparse is better than dense | One concept per line; blank lines between logical blocks |
| Z-07 | Readability counts | Code is written once, read many times — optimise for the reader |
| Z-08 | Special cases aren't special enough to break the rules | No one-off hacks; generalise, or document and isolate the exception |
| Z-09 | Practicality beats purity | Pragmatic solutions are fine; document every trade-off explicitly |
| Z-10 | Errors should never pass silently | Every exception must be caught, logged, or re-raised with context |
| Z-11 | Unless explicitly silenced | Suppressed errors must carry an inline comment explaining the reason |
| Z-12 | Refuse the temptation to guess | If ambiguous, ask or research — never assume and proceed |
| Z-13 | One obvious way | Prefer the idiomatic pattern for the language/framework in use |
| Z-14 | Now is better than never | Deliver working solutions over endlessly deferred perfect ones |
| Z-15 | Never is often better than right now | Don't ship unfinished code — complete it properly first |
| Z-16 | Hard to explain = bad idea | If you struggle to explain the design, redesign it |
| Z-17 | Easy to explain = may be a good idea | Clear, explainable solutions are worth pursuing |
| Z-18 | Namespaces are a great idea | Module/package structure matters; never pollute a global namespace |
| Z-19 | Let's do more of those | Prefer well-named, well-scoped modules over monolithic files |

---

## 2. Clean Code — Naming (N-rules)

| ID | Rule |
|---|---|
| N-01 | Names must **reveal intent** — `elapsed_days` not `d` |
| N-02 | No disinformation — `account_list` must actually be a list |
| N-03 | Meaningful distinctions — never use `product` and `product_data` as synonyms |
| N-04 | Pronounceable names — if you can't say it aloud, rename it |
| N-05 | Searchable names — no single-letter variables outside tight loop scopes |
| N-06 | No encodings — no Hungarian notation or type prefixes (`strName`, `iCount`) |
| N-07 | Class names: noun or noun phrase (`Customer`, `WikiPage`, `UserRepository`) |
| N-08 | Function names: verb or verb phrase (`delete_page`, `send_email`, `is_valid`) |
| N-09 | One word per concept — don't mix `fetch`/`retrieve`/`get` for the same operation |
| N-10 | Use domain language — problem domain first, solution domain second |

---

## 3. Clean Code — Functions (F-rules)

| ID | Rule |
|---|---|
| F-01 | **Do one thing** — if you need "and" to describe it, extract a function |
| F-02 | One level of abstraction per function — no mixing high-level steps with low-level detail |
| F-03 | Max **3 arguments** — use a dataclass or config object for more |
| F-04 | No flag arguments (`process(true)`) — split into two explicitly named functions |
| F-05 | No output arguments — functions return values; callers must not pass targets to mutate |
| F-06 | Prefer exceptions to returning error codes |
| F-07 | **No hidden side effects** — `get_user()` must not mutate state |
| F-08 | **DRY** — duplicated logic is a violation; extract to a shared, named function |

---

## 4. Clean Code — Comments & Docs (CC-rules)

| ID | Rule |
|---|---|
| CC-01 | Good code explains itself — a comment often signals a failure to express intent in code |
| CC-02 | Acceptable comments: legal headers · intent (why, not what) · clarification · TODO/FIXME |
| CC-03 | Never redundant — `i++  // increment i` is noise; delete it |
| CC-04 | Never misleading or outdated — update or delete stale comments immediately |
| CC-05 | No commented-out code — use version control history instead |
| CC-06 | Docstrings on **public API only** — not on private or internal implementation details |

---

## 5. Clean Code — Classes & Design (O-rules / SOLID)

| ID | Rule |
|---|---|
| O-01 | **SRP** — one class, one reason to change |
| O-02 | **OCP** — open for extension, closed for modification |
| O-03 | **LSP** — subtypes must fully honour the base-type contract |
| O-04 | **ISP** — prefer small, focused interfaces over fat ones |
| O-05 | **DIP** — depend on abstractions, not concretions |
| O-06 | **Law of Demeter** — talk only to immediate collaborators; no `a.b.c.d()` chains |
| O-07 | High cohesion — most methods use most instance variables |
| O-08 | Small classes — if it requires scrolling to read, it has too much responsibility |

---

## 6. Clean Code — Error Handling (E-rules)

| ID | Rule |
|---|---|
| E-01 | Prefer exceptions to error codes; let callers decide how to handle |
| E-02 | Provide **context** with every exception — message, cause, affected value |
| E-03 | Don't return `None` from functions that can fail — raise or use `Optional` with docs |
| E-04 | **Never swallow exceptions silently** (enforced by Z-10 / Z-11) |
| E-05 | Define custom exception classes per domain area — never bare `Exception` |

---

## 7. Clean Code — Format & Structure (R-rules)

| ID | Rule |
|---|---|
| R-01 | **Newspaper rule** — high-level abstractions at top; detail below |
| R-02 | Related code stays together — caller above callee; related methods grouped |
| R-03 | Blank lines as paragraph breaks between logical blocks |
| R-04 | Max line length: 100 chars (Python 88 per Black/Ruff; JS/TS 100 per Prettier) |
| R-05 | Indentation reflects scope — never use whitespace to align arbitrary values |
| R-06 | Configured formatter takes precedence — `ruff` · `black` · `prettier` · `gofmt` |

---

## 8. Emergence Rules (Kent Beck — Four Rules of Simple Design)

In priority order:

1. **Passes all tests** — correctness before elegance
2. **No duplication** — every piece of knowledge has one authoritative location
3. **Expresses intent** — names, structure, and comments communicate purpose clearly
4. **Minimises** — fewest classes, methods, and lines that satisfy the above three rules

---

## 9. Post-Write Self-Check (Mandatory Before Marking a Unit Done)

- [ ] Names: N-01 to N-10 applied
- [ ] Functions: F-01 to F-08 applied — each function does one thing
- [ ] Comments: CC-01 to CC-06 — no noise, no dead code, public API documented
- [ ] Classes/Design: O-01 to O-08 (SOLID + Demeter) applied
- [ ] Error handling: E-01 to E-05 applied — no silent failures
- [ ] Format: R-01 to R-06 applied — formatter run
- [ ] Emergence rules §8: design passes all four checks
- [ ] All 19 Zen aphorisms (Z-01 to Z-19) verified against this unit
