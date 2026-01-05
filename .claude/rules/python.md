---
paths: "**/*.py"
---

# Python Coding Standards

These rules are automatically applied when working with Python files.

## Import Rules
- **NO inline imports** unless specifically for lazy loading (e.g., avoiding circular dependencies or heavy imports)
- All imports MUST be at the top of the file, organized in this order:
  1. Standard library imports
  2. Third-party imports
  3. Local application imports
- Use absolute imports over relative imports when possible

## Type Hints
- **ALWAYS use lowercase built-in types**: `dict`, `list`, `tuple`, `set`, `frozenset`
- **NEVER use** `Dict`, `List`, `Tuple`, `Set` from `typing` module
- For generic type hints, use:
  - `dict[str, int]` NOT `Dict[str, int]`
  - `list[str]` NOT `List[str]`
  - `tuple[int, ...]` NOT `Tuple[int, ...]`
- Use `typing` module ONLY for:
  - `Optional`, `Union`, `Any`, `Callable`, `TypeVar`, `Protocol`, `Literal`
  - Type aliases for complex types
  - `TypedDict` for structured dictionaries

## Code Style
- Use **f-strings** for string formatting (not `.format()` or `%`)
- Use **pathlib.Path** for file system operations (not `os.path`)
- Use **context managers** (`with` statements) for resource management
- **No bare `except:`** - always specify exception types
- Use **is** for None comparisons: `if x is None:` not `if x == None:`
- Use **in** for membership testing: `if key in dict:` not `if key in dict.keys():`

## Function and Variable Naming
- Functions and variables: `snake_case`
- Classes: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`
- Private attributes/methods: prefix with single underscore `_private_method`

## Documentation
- All public functions/classes MUST have docstrings
- Use Google docstring format consistently
- Include type hints in function signatures, not in docstrings

## Modern Python Practices (3.9+)
- Use `|` for Union types: `str | int` instead of `Union[str, int]`
- Use built-in generics: `list[int]` instead of `List[int]`
- Use `match`/`case` for complex conditionals (Python 3.10+)

## Configuration & Grouping

### Group Related Variables
- **NEVER** have multiple loose related variables - group them into a structure
- Use `@dataclass`, `TypedDict`, or Pydantic `BaseModel` for related config

```python
# Bad - scattered related config
TIMEOUT = 30
TIMEOUT_JOB1 = 60
TIMEOUT_JOB2 = 120
MAX_RETRIES = 3
RETRY_DELAY = 5

# Good - grouped logically
@dataclass
class TimeoutConfig:
    default: int = 30
    job1: int = 60
    job2: int = 120

@dataclass
class RetryConfig:
    max_retries: int = 3
    delay: int = 5
```

### Environment Variables
- Group related env vars into a config class
- Use Pydantic `BaseSettings` or a config dataclass
- Load once at startup, pass config objects around

```python
# Bad
db_host = os.getenv("DB_HOST")
db_port = os.getenv("DB_PORT")
db_user = os.getenv("DB_USER")
db_pass = os.getenv("DB_PASS")

# Good
@dataclass
class DatabaseConfig:
    host: str
    port: int
    user: str
    password: str

    @classmethod
    def from_env(cls) -> "DatabaseConfig":
        return cls(
            host=os.environ["DB_HOST"],
            port=int(os.environ["DB_PORT"]),
            user=os.environ["DB_USER"],
            password=os.environ["DB_PASS"],
        )
```

### Function Parameters
- If a function takes 4+ related parameters, group them
- Prefer a config/params dataclass over long argument lists

```python
# Bad
def connect(host: str, port: int, user: str, password: str, timeout: int, retries: int): ...

# Good
def connect(config: ConnectionConfig): ...
```

## Common Pitfalls

### Mutable Default Arguments
- **NEVER** use mutable defaults: `def foo(items=[])` or `def foo(data={})`
- **ALWAYS** use None and initialize inside:
  ```python
  def foo(items: list[str] | None = None) -> None:
      items = items or []
  ```

### Boolean and Empty Checks
- Use truthiness: `if items:` NOT `if len(items) > 0:`
- Use truthiness: `if not items:` NOT `if len(items) == 0:`
- Use direct bool: `if flag:` NOT `if flag == True:`
- Use direct bool: `if not flag:` NOT `if flag == False:`

### String Operations
- Use `.startswith()/.endswith()`: NOT string slicing `s[:3] == "foo"`
- Use `"sep".join(items)`: NOT loop concatenation
- Use `in` for substring: `if "x" in s:` NOT `s.find("x") != -1`

### Type Checking
- Use tuple for multiple types: `isinstance(x, (int, float))` NOT `isinstance(x, int) or isinstance(x, float)`
- Use `collections.abc` for abstract types: `Mapping`, `Sequence`, `Iterable`

## Data Structures

### Prefer Dataclasses
- Use `@dataclass` for data containers instead of manual `__init__`
- Use `frozen=True` for immutable data
- Use `slots=True` (3.10+) for memory efficiency

### Prefer Enum
- Use `Enum` for fixed sets of values, NOT string constants
  ```python
  # Bad
  STATUS_PENDING = "pending"
  STATUS_DONE = "done"

  # Good
  class Status(Enum):
      PENDING = "pending"
      DONE = "done"
  ```

### Comprehensions
- Prefer list/dict/set comprehensions over `map()`/`filter()`
- Keep comprehensions simple - use loops for complex logic
- Use generator expressions for large sequences: `(x for x in items)`

## Production Code

### Logging
- Use `logging` module, NOT `print()` for production code
- Use appropriate log levels: `debug`, `info`, `warning`, `error`, `critical`

### No Global State
- Avoid `global` keyword
- Pass dependencies explicitly or use dependency injection

## Dependencies

### Type Stubs
- **ALWAYS** install type stubs when available for third-party packages
- Search for stubs: `types-<package>` (e.g., `types-requests`, `types-boto3`, `types-redis`)
- Check typeshed or PyPI for available stubs before concluding they don't exist
- Add stubs to dev dependencies, not production

```bash
# When adding a package, also check for stubs
uv add requests
uv add --dev types-requests
```

### Version Pinning
- Let `uv` or `pip` resolve dependencies first, then pin the resolved versions
- **NEVER** guess or manually specify version numbers
- Use lockfiles (`uv.lock`, `poetry.lock`) for reproducible builds

```bash
# Good - let resolver pick, then pin
uv add requests        # Resolves to latest compatible
uv lock                # Pins exact versions in lockfile

# Bad - guessing versions
uv add requests==2.28.0  # Don't guess versions
```

### Dependency Workflow
1. Add package without version constraint: `uv add <package>`
2. Check for type stubs: `uv add --dev types-<package>` (if available)
3. Let resolver determine compatible versions
4. Commit the lockfile with pinned versions
