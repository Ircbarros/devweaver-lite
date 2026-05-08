# guide_python_standards

> Stream D — Python coding standards, style guide, and security patterns.
> Applies to all `web/`, `ai/`, `iot/`, `storage/`, `observability/` Python code.
> **Advanced patterns**: load `guide_python_advanced_pt1.md` (idioms: Effective Python + Fluent Python)
> and `guide_python_advanced_pt2.md` (performance: High Performance Python) before implementing non-trivial Python.

---

## Table of Contents

1. [PEP 8 Key Rules](#1-pep-8-key-rules)
2. [Google Python Style Guide Deviations](#2-google-python-style-guide-deviations)
3. [SOLID in Python FastAPI Services](#3-solid-in-python-fastapi-services)
4. [Type Hints Best Practices](#4-type-hints-best-practices)
5. [ruff Configuration](#5-ruff-configuration)
6. [Python Security Anti-Patterns](#6-python-security-anti-patterns)
7. [Stable Versions](#7-stable-versions)

---

## 1. PEP 8 Key Rules

| Rule | Detail |
|---|---|
| Line length | 88 chars (black default); configure in `ruff.toml` |
| Indentation | 4 spaces, never tabs |
| Blank lines | 2 between top-level, 1 between methods |
| Imports | stdlib → third-party → local; each group separated by blank line; absolute imports only |
| Naming | `snake_case` (vars, funcs), `PascalCase` (classes), `UPPER_SNAKE` (constants), `_private` prefix |
| Dunder methods | `__all__` at top of public modules |

**Exceptions configured in `ruff.toml`** (see §5).

---

## 2. Google Python Style Guide Deviations

- **Docstrings**: Google format (`Args:`, `Returns:`, `Raises:`); only on public API functions (not private helpers).
- **Type annotations**: Required on all public function signatures; not required on private helpers (< 10 lines).
- **Default arguments**: Never use mutable defaults (`def f(x: list = [])` → `def f(x: list | None = None)`).
- **Comprehensions**: Use for simple transforms; never nest more than 2 levels.
- **Context managers**: Prefer over `try/finally` for resource cleanup.
- **Exceptions**: Never silently catch `Exception`; always log with context.

```python
# ✅ Google-style docstring
def create_user(user_in: UserCreate, db: AsyncSession) -> User:
    """Create a new user in the database.

    Args:
        user_in: Validated Pydantic schema with user fields.
        db: Active async SQLAlchemy session.

    Returns:
        Persisted User ORM object.

    Raises:
        IntegrityError: If email already exists.
    """
```

---

## 3. SOLID in Python FastAPI Services

| Principle | Application |
|---|---|
| **S** — Single Responsibility | Each service class handles one domain (AuthService, StorageService) |
| **O** — Open/Closed | Protocol classes for storage backends; swap Garage for MinIO without changing service |
| **L** — Liskov Substitution | All storage backends implement `StorageProtocol`; FastAPI injects via `Depends` |
| **I** — Interface Segregation | `StorageReadProtocol` vs `StorageWriteProtocol`; read-only code gets read-only interface |
| **D** — Dependency Inversion | FastAPI service receives database session via `Depends(get_db)`; never instantiates directly |

```python
from typing import Protocol

class StorageProtocol(Protocol):
    async def upload(self, bucket: str, key: str, data: bytes) -> str: ...
    async def download(self, bucket: str, key: str) -> bytes: ...

class GarageStorage:  # Implements StorageProtocol without explicit inheritance
    async def upload(self, bucket: str, key: str, data: bytes) -> str: ...
    async def download(self, bucket: str, key: str) -> bytes: ...
```

---

## 4. Type Hints Best Practices

| When to Use | Type |
|---|---|
| Structured immutable read model (from DB/API) | `TypedDict` |
| Mutable in-memory data object | `dataclass` |
| Request/response + validation + serialisation | `pydantic.BaseModel` |
| LangGraph state (enforced immutable reducer) | `TypedDict` with `Annotated` reducers |

```python
# TypedDict: LangGraph state (no methods, used by type checker)
class SkillState(TypedDict):
    phase: str
    confidence: float

# dataclass: mutable service DTO
@dataclass
class ParsedMQTTMessage:
    topic: str
    value: float
    timestamp: int

# Pydantic: API schema with validation
class SensorReading(BaseModel):
    model_config = ConfigDict(strict=True)
    value: float = Field(..., ge=-100.0, le=200.0)
```

---

## 5. ruff Configuration

```toml
# ruff.toml
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = [
  "E",   # pycodestyle errors
  "W",   # pycodestyle warnings
  "F",   # pyflakes
  "I",   # isort
  "B",   # bugbear
  "C4",  # flake8-comprehensions
  "UP",  # pyupgrade
  "S",   # bandit/flake8-bandit security
  "ANN", # type annotations
  "N",   # pep8-naming
]
ignore = ["ANN101", "ANN102", "S101"]  # Allow assert in tests; skip self/cls annotations

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S", "ANN"]  # Relax security/annotation rules in tests
```

---

## 6. Python Security Anti-Patterns

| Anti-Pattern | Risk | Safe Alternative |
|---|---|---|
| `eval(user_input)` | RCE | Never eval user input; use `ast.literal_eval` only for trusted config |
| `subprocess.run(cmd, shell=True)` | Shell injection | Use `shell=False`; pass args as list |
| `open(path)` with user-supplied path | Path traversal | Validate with `Path(path).resolve().is_relative_to(BASE_DIR)` |
| `pickle.loads(data)` | Deserialization RCE | Use `json.loads()` or Pydantic validation |
| Hardcoded secrets | Credential leakage | `os.getenv()` + `pydantic-settings` `BaseSettings` |

```python
# ✅ Safe subprocess
subprocess.run(["git", "log", "--oneline"], shell=False, capture_output=True, timeout=10)

# ✅ Safe path resolution
from pathlib import Path
def safe_path(user_input: str, base: Path) -> Path:
    p = (base / user_input).resolve()
    if not p.is_relative_to(base):
        raise ValueError("Path traversal attempt")
    return p
```

---

## 7. Stable Versions

| Tool / Package | Version | Notes |
|---|---|---|
| Python | `3.12.x` | ✅ Stable; use as default runtime |
| `ruff` | `0.4.x` | ✅ Stable |
| `black` | `24.x` | ✅ Stable (format only; ruff handles lint) |
| `mypy` | `1.10.x` | ✅ Stable |
| `pydantic-settings` | `2.x` | ✅ Stable |
| `asyncpg` | `0.29.x` | ✅ Stable |
| `SQLAlchemy[asyncio]` | `2.0.x` | ✅ Stable |
