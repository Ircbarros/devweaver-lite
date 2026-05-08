# guide_python_advanced_pt1

> Idiomatic and expressive Python patterns — language/framework agnostic.
> Sources: **Effective Python** (Brett Slatkin, 3rd ed.) · **Fluent Python** (Luciano Ramalho, 2nd ed.)
> Companion: `guide_python_advanced_pt2.md` (performance) · `guide_python_standards.md` (PEP 8 / SOLID)
> Loaded at: IMPLEMENT (Python scope) · File size rule: **R-PD-02** (≤ 200 lines)

---

## 1. Effective Python — Idiomatic Rules (EP)

### 1.1 Strings & Data

| ID | Rule |
|---|---|
| EP-01 | f-strings only — never `%`-format or `.format()` for new code |
| EP-02 | Unpack sequences with assignment unpacking, not index access: `first, *rest = items` |
| EP-03 | `enumerate(seq, start=1)` over `range(len(seq))` — expressive and index-safe |
| EP-04 | Never `else` after `for`/`while` — it executes when the loop finishes normally; confuses readers |
| EP-05 | `dict.get(key, default)` over `key in dict` + conditional; `collections.defaultdict` for accumulation |

### 1.2 Functions & Arguments

| ID | Rule |
|---|---|
| EP-06 | Mutable default arguments must never be used — use `None` sentinel: `def f(x=None): x = x or []` |
| EP-07 | Keyword-only arguments for named options, positional-only for internal numeric params: `def f(a, b, /, *, timeout)` |
| EP-08 | Use generators (`yield`) instead of building and returning full lists for large or unbounded sequences |
| EP-09 | `contextlib.contextmanager` for lightweight context managers; prefer `with` over `try/finally` |
| EP-10 | Closures capture variables by reference — bind at definition with a default arg: `lambda x=x: x` |

### 1.3 Concurrency & Async

| ID | Rule |
|---|---|
| EP-11 | `asyncio` for I/O-bound concurrency; `concurrent.futures.ProcessPoolExecutor` for CPU-bound |
| EP-12 | `asyncio.run()` as the single top-level entry point — never mix sync and async call stacks |
| EP-13 | Never call `time.sleep()` inside an async function — use `await asyncio.sleep()` |
| EP-14 | `asyncio.gather(*coros)` for fan-out; `asyncio.TaskGroup` (3.11+) for structured concurrency |
| EP-15 | `subprocess.run(args_list, shell=False, timeout=N)` — never `shell=True` with user-controlled input |

### 1.4 Caching & Performance

| ID | Rule |
|---|---|
| EP-16 | `@functools.cache` (unbounded) / `@functools.lru_cache(maxsize=N)` only on pure, side-effect-free functions |
| EP-17 | `@functools.lru_cache` must never be applied to instance methods that hold state — use class-level caches |
| EP-18 | Prefer `dataclass` / `TypedDict` / `NamedTuple` over plain tuples for multi-field returns |

---

## 2. Fluent Python — Data Model & Idioms (FP)

### 2.1 Python Data Model

| ID | Rule |
|---|---|
| FP-01 | Implement `__repr__` before `__str__` — `repr` for developer debugging; `str` for end-user output |
| FP-02 | `__len__` + `__getitem__` make any class a sequence — no need to subclass `list` or `abc.Sequence` |
| FP-03 | `__slots__` on data-heavy, high-volume classes reduces per-instance memory ~40%; incompatible with `__dict__` |
| FP-04 | Generator expressions `(x for x in y)` are lazy iterators — never materialise with `list()` unless required |

### 2.2 Typing & Protocols

| ID | Rule |
|---|---|
| FP-05 | `typing.Protocol` for structural subtyping (duck typing with type-checker support) — preferred over ABC in app code |
| FP-06 | `typing.TypeVar`, `Generic[T]` for reusable type-safe containers; `ParamSpec` for decorator typing |
| FP-07 | `typing.TypeAlias` for complex repeated type expressions; `type X = ...` (3.12+) |
| FP-08 | Structural pattern matching (`match/case`, 3.10+) over nested `if-elif` chains for dispatch on data shape |

### 2.3 Functions & Decorators

| ID | Rule |
|---|---|
| FP-09 | `@classmethod` for alternative constructors (`from_dict`, `from_env`); `@staticmethod` for free functions attached to a class |
| FP-10 | Decorators with arguments need three levels: `factory(args)` → `decorator(fn)` → `wrapper(*a, **kw)` |
| FP-11 | `functools.wraps(fn)` is mandatory on every wrapper — preserves `__name__`, `__doc__`, `__wrapped__` |
| FP-12 | `functools.reduce` only for commutative, associative operations — prefer explicit loops for clarity |

### 2.4 Collections & Async

| ID | Rule |
|---|---|
| FP-13 | `dict` is insertion-ordered (3.7+) — never sort a dict when insertion order is the stated contract |
| FP-14 | `collections.ChainMap` to layer config dicts; `collections.Counter` for frequency counting; `heapq` for priority queues |
| FP-15 | `__init_subclass__` for composable class extension hooks — metaclasses only when `__init_subclass__` is insufficient |
| FP-16 | Async generators (`async def` + `yield`) for streaming; wrap with `async with contextlib.asynccontextmanager` |
| FP-17 | `__getattr__` for lazy attribute loading (called only when normal lookup fails); `__getattribute__` only as a last resort |

---

## 3. Post-Implementation Checklist

- [ ] f-strings used throughout — no `%` or `.format()` (EP-01)
- [ ] No mutable default arguments (EP-06)
- [ ] All async functions use `await asyncio.sleep` not `time.sleep` (EP-13)
- [ ] All cached functions are pure and side-effect-free (EP-16)
- [ ] `__repr__` defined on all domain models (FP-01)
- [ ] Structural subtyping via `Protocol` — not forced ABC inheritance (FP-05)
- [ ] All decorator wrappers use `@functools.wraps` (FP-11)
- [ ] `typing.TypeVar` + `Generic` used for reusable containers (FP-06)
