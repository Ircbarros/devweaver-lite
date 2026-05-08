# guide_python_advanced_pt2

> Python performance, profiling, and memory patterns — language-agnostic where noted.
> Source: **High Performance Python** (Gorelick & Ozsvald, 2nd ed.)
> Companion: `guide_python_advanced_pt1.md` (idioms) · `guide_python_standards.md` (PEP 8 / SOLID)
> Loaded at: IMPLEMENT (Python scope) · File size rule: **R-PD-02** (≤ 200 lines)

---

## 1. Profiling — Always Profile Before Optimising (PR)

| ID | Rule |
|---|---|
| PR-01 | **Profile first** — never optimise a function without a measured bottleneck; premature optimisation is the root of evil |
| PR-02 | `cProfile` + `pstats` for function-level CPU profiling; `line_profiler` (`@profile`) for line-level hot-path inspection |
| PR-03 | `memory_profiler` (`@profile`) for per-line memory tracking; `tracemalloc` for leak detection |
| PR-04 | `perf_counter_ns()` for micro-benchmarks; never use `time.time()` for performance measurement |
| PR-05 | Profile against **production-representative data** — micro-benchmarks on toy data yield misleading results |
| PR-06 | Fix the algorithm first (O(n²) → O(n log n)) before any implementation micro-optimisation |

---

## 2. Data Structures & Memory (DS)

| ID | Rule |
|---|---|
| DS-01 | `array.array` or `numpy.ndarray` for large homogeneous numeric sequences — ~8× smaller than `list` |
| DS-02 | `collections.deque` for O(1) append/pop from both ends; `list.pop(0)` is O(n) — never use it in hot paths |
| DS-03 | `bisect.insort` / `bisect.bisect_left` for maintaining sorted lists without re-sorting |
| DS-04 | `set` / `frozenset` for O(1) membership testing — never `x in list` in a loop |
| DS-05 | Generator pipelines over intermediate list materialisation — each stage stays lazy |
| DS-06 | `__slots__` on classes used in large collections — eliminates per-instance `__dict__` overhead |
| DS-07 | `sys.getsizeof` + `tracemalloc` to verify memory assumptions — never eyeball object sizes |

---

## 3. NumPy & Vectorisation (NP)

| ID | Rule |
|---|---|
| NP-01 | NumPy vectorised operations over Python loops for numerical work — orders-of-magnitude faster |
| NP-02 | Pre-allocate arrays (`np.zeros`, `np.empty`) — growing arrays with `np.append` in a loop is O(n²) |
| NP-03 | Use boolean indexing and fancy indexing instead of Python-level `if`/`filter` on arrays |
| NP-04 | `np.einsum` for matrix operations when broadcasting rules become hard to read |
| NP-05 | C-contiguous row-major order (NumPy default) — column-major access patterns cause cache misses |

---

## 4. Async I/O & Concurrency (AIO)

| ID | Rule |
|---|---|
| AIO-01 | `asyncio` eliminates thread overhead for I/O-bound work — correct pattern for FastAPI, LangGraph, aiomqtt |
| AIO-02 | `asyncio.Semaphore(N)` to cap concurrency on resource-limited operations (DB connections, external API rate limits) |
| AIO-03 | CPU-bound tasks inside an async budget: `loop.run_in_executor(ProcessPoolExecutor(), fn)` — never block the event loop |
| AIO-04 | `asyncio.Queue` for producer-consumer pipelines; bounded (`maxsize=N`) to apply back-pressure |
| AIO-05 | `asyncio.timeout(seconds)` (3.11+) around external calls — never let I/O hang indefinitely |
| AIO-06 | `aiofiles` for async file I/O — `open()` blocks the event loop thread |

---

## 5. Caching Strategies (CA)

| ID | Rule |
|---|---|
| CA-01 | In-process: `functools.lru_cache` for hot deterministic functions with bounded input space |
| CA-02 | Distributed: Redis (`redis-py async`) for shared cache across processes and workers |
| CA-03 | Cache keys must be **stable and deterministic** — include version hash for ML models and embeddings |
| CA-04 | Always set a TTL on external caches — stale-forever cache entries are a correctness bug, not a performance win |
| CA-05 | Cache at the boundary (service layer) — never inside a repository/ORM method |
| CA-06 | Invalidate explicitly on write — avoid cache-aside with a very long TTL as a substitute for proper invalidation |

---

## 6. Compiled Extensions & JIT (CE)

| ID | Rule |
|---|---|
| CE-01 | `Cython` for tight numerical loops proven to be the bottleneck after profiling; pure Python first |
| CE-02 | `numba` `@jit(nopython=True)` for array-heavy functions without Python object interactions |
| CE-03 | `cffi` / `ctypes` to call existing C libraries — never reimplement low-level algorithms from scratch |
| CE-04 | Profile the Python version first — numpy + vectorisation often makes Cython unnecessary |
| CE-05 | Extensions must have Python fallbacks for portability; gate behind `try/except ImportError` |

---

## 7. Performance Anti-Patterns (PA)

| Anti-pattern | Problem | Fix |
|---|---|---|
| `+` string concatenation in a loop | O(n²) copies | `"".join(parts)` or f-string in list then join |
| `list.pop(0)` in hot path | O(n) shift | `collections.deque.popleft()` |
| Repeated `dict` key hashing in a loop | Wasted work | Cache the value: `val = d[key]` outside loop |
| Loading full file into memory | OOM on large files | Stream with `for line in file` or `mmap` |
| Pickle for IPC between processes | Slow serialisation | `multiprocessing.shared_memory` or `numpy` shared arrays |
| `global` variable for shared state in async | Race condition / GIL false security | `contextvars.ContextVar` for per-task state |

---

## 8. Post-Implementation Performance Checklist

- [ ] Bottleneck identified via `cProfile` or `line_profiler` before any optimisation (PR-01, PR-02)
- [ ] Algorithm complexity addressed before implementation micro-optimisation (PR-06)
- [ ] Hot-path loops: no `list.pop(0)`, no `in list`, no string `+` concatenation (DS-02, DS-04, PA table)
- [ ] Numeric data: `numpy` arrays over Python lists (NP-01, NP-02)
- [ ] Event loop never blocked by CPU-bound or synchronous I/O (AIO-03, AIO-06)
- [ ] All `asyncio` external calls have a timeout (AIO-05)
- [ ] Caches have TTLs; invalidated explicitly on write (CA-04, CA-06)
