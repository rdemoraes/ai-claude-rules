# Python CLI — Coding Standards

Applies to all Python CLI projects. Mirrors the same SOLID + clean code principles used in the bash toolkit.

## Core Principle: Keep It Simple

Write the simplest CLI that solves the problem. No abstractions without a concrete, immediate need.

## Stack

- **Framework**: [Typer](https://typer.tiangolo.com/) (`typer[all]`) — type-first, auto-help, subcommand support
- **Python**: 3.9+ with full PEP 484 type hints and PEP 593 `Annotated` syntax
- **Testing**: `typer.testing.CliRunner` + `unittest.mock`
- **Packaging**: `pyproject.toml` with `[project.scripts]` entry point

## Project Structure

New domain = new directory under `commands/`. New resource = new file under `commands/<domain>/`.

```
<pkg>/
├── __init__.py               # __version__ only
├── cli.py                    # Root app, global options, domain registration
├── context.py                # AppContext dataclass (dependency injection)
└── commands/
    └── <domain>/
        ├── __init__.py
        └── <resource>.py     # Commands for one resource
tests/
└── test_<resource>.py        # Integration tests via CliRunner
pyproject.toml                # Build config, dependencies, entry points
```

## Entry Point

`cli.py` is the root. It defines the root app, global options, and registers domain sub-apps. It never contains business logic.

```python
# BAD — command logic in cli.py
@app.command("set-alias")
def set_alias(ctx: typer.Context, name: str) -> None:
    iam = boto3.client("iam")
    iam.create_account_alias(AccountAlias=name)
    typer.echo(f"Created: {name}")

# GOOD — cli.py only registers sub-apps and defines global options
from <pkg>.commands.aws import account as account_cmd

aws_app = typer.Typer(help="AWS operations.")
aws_app.add_typer(account_cmd.app, name="account")
app.add_typer(aws_app, name="aws")
```

## Global Options

Parse global options once in `cli.py` using `@app.callback()`. Never re-parse them inside command or domain modules.

```python
@app.callback()
def root(
    ctx: typer.Context,
    region: Annotated[str, typer.Option(help="AWS region.")] = "us-east-1",
    profile: Annotated[Optional[str], typer.Option(help="AWS named profile.")] = None,
    output: Annotated[str, typer.Option(help="Output format: table or json.")] = "table",
    version: Annotated[
        bool, typer.Option("--version", is_eager=True, callback=version_callback, help="Print version.")
    ] = False,
) -> None:
    ctx.obj = AppContext(region=region, profile=profile, output=output)
```

**Standard global options for all CLIs:**

| Option | Type | Default |
|--------|------|---------|
| `--region` | `str` | `"us-east-1"` |
| `--profile` | `Optional[str]` | `None` |
| `--output` | `str` | `"table"` |
| `--version` | eager flag | — |

## Dependency Injection via AppContext

`context.py` holds the single `AppContext` dataclass. Commands receive it through Typer's `ctx.obj` — never via global variables or direct `os.environ` reads.

```python
# context.py
@dataclass
class AppContext:
    region: str = "us-east-1"
    profile: str | None = None
    output: str = "table"
    _session: Any = field(init=False, repr=False)

    def __post_init__(self) -> None:
        self._session = boto3.Session(
            profile_name=self.profile,
            region_name=self.region,
        )

    def boto3_client(self, service: str) -> Any:
        return self._session.client(service)
```

```python
# BAD — global state, untestable
AWS_REGION = os.environ.get("AWS_REGION", "us-east-1")

def get_existing_alias() -> str | None:
    iam = boto3.client("iam", region_name=AWS_REGION)
    ...

# GOOD — explicit dependency
def get_existing_alias(app_ctx: AppContext) -> str | None:
    iam = app_ctx.boto3_client("iam")
    ...
```

## SOLID in Practice

**Single Responsibility** — separate action, builder, renderer, and orchestrator functions.

```python
# BAD — one function does everything
@app.command("set-alias")
def set_alias(ctx: typer.Context, account_name: str) -> None:
    app_ctx: AppContext = ctx.obj
    iam = app_ctx.boto3_client("iam")
    aliases = iam.list_account_aliases().get("AccountAliases", [])
    if aliases:
        typer.echo(f"Alias already set: {aliases[0]}")
        return
    alias = f"{account_name}-{petname.Generate(2, '-')}"
    iam.create_account_alias(AccountAlias=alias)
    typer.echo(f"Account alias (created): {alias}")


# GOOD — each function has one job
def get_existing_alias(app_ctx: AppContext) -> str | None:           # ACTION
    iam = app_ctx.boto3_client("iam")
    aliases = iam.list_account_aliases().get("AccountAliases", [])
    return aliases[0] if aliases else None

def build_alias(account_name: str) -> str:                           # BUILDER
    return f"{account_name}-{petname.Generate(2, '-')}"

def create_alias(app_ctx: AppContext, alias: str) -> None:           # ACTION
    iam = app_ctx.boto3_client("iam")
    iam.create_account_alias(AccountAlias=alias)

def render_result(output: str, alias: str, created: bool) -> None:   # RENDERER
    data = {"alias": alias, "created": created}
    if output == "json":
        typer.echo(json.dumps(data))
    else:
        status = "created" if created else "already set"
        typer.echo(f"Account alias ({status}): {alias}")

@app.command("set-alias")                                            # ORCHESTRATOR
def set_alias(ctx: typer.Context, account_name: str) -> None:
    app_ctx: AppContext = ctx.obj
    existing = get_existing_alias(app_ctx)
    if existing:
        render_result(app_ctx.output, existing, created=False)
        return
    alias = build_alias(account_name)
    create_alias(app_ctx, alias)
    render_result(app_ctx.output, alias, created=True)
```

**Open/Closed** — extend by adding new files in `commands/<domain>/`, never by modifying `cli.py` or existing command modules.

**Dependency Inversion** — functions receive all dependencies as arguments. Never import and call `boto3.client()` directly inside a command function.

## Error Handling

Catch specific exceptions. Write errors to stderr. Exit with a non-zero code via `typer.Exit`.

```python
# BAD
def get_existing_alias(app_ctx: AppContext) -> str | None:
    iam = app_ctx.boto3_client("iam")
    return iam.list_account_aliases()["AccountAliases"][0]  # KeyError, IndexError silently crash

# GOOD
def get_existing_alias(app_ctx: AppContext) -> str | None:
    iam = app_ctx.boto3_client("iam")
    try:
        aliases = iam.list_account_aliases().get("AccountAliases", [])
        return aliases[0] if aliases else None
    except ClientError as e:
        typer.echo(f"AWS error: {e.response['Error']['Message']}", err=True)
        raise typer.Exit(code=1)
```

- Use `typer.echo(msg, err=True)` for all error messages (goes to stderr).
- Use `raise typer.Exit(code=1)` to exit with failure — never `sys.exit()` or bare `raise`.
- Catch the most specific exception type available (`ClientError`, `FileNotFoundError`, etc.).

## Output Formatting

Every command must support at least `table` (human-readable) and `json` (machine-readable) output via `app_ctx.output`.

```python
def render_result(output: str, alias: str, created: bool) -> None:
    if output == "json":
        typer.echo(json.dumps({"alias": alias, "created": created}))
    else:
        status = "created" if created else "already set"
        typer.echo(f"Account alias ({status}): {alias}")
```

- One `render_*` function per command — handles all formats.
- Never mix rendering with action or builder logic.
- Use `typer.echo()` for all output, never `print()`.

## Naming Conventions

Follows [PEP 8](https://peps.python.org/pep-0008/) — the definitive Python standard.

| Construct | Convention | Example |
|-----------|-----------|---------|
| Variables | `snake_case` | `account_name`, `existing_alias` |
| Functions | `snake_case` | `get_existing_alias`, `build_alias` |
| Constants | `UPPER_SNAKE_CASE` | `DEFAULT_REGION = "us-east-1"` |
| Classes | `PascalCase` | `AppContext`, `AliasResult` |
| Private/internal | `_snake_case` | `_session`, `_build_suffix` |
| CLI command names | `kebab-case` | `set-alias`, `list-accounts` |
| Module/file names | `snake_case` | `account.py`, `app_context.py` |

```python
# BAD
accountName = "prod"           # camelCase — not Python
ACCOUNT_name = "prod"          # mixed — confusing
def GetAlias(): ...            # PascalCase on a function

# GOOD
account_name = "prod"          # variable: snake_case
DEFAULT_REGION = "us-east-1"   # constant: UPPER_SNAKE_CASE
def get_existing_alias(): ...  # function: snake_case
class AppContext: ...           # class: PascalCase
```

## Clean Code Rules

- **Meaningful names**: `get_existing_alias` not `check`. `account_name` not `n`.
- **Full type hints**: every function parameter and return value must be annotated.
- **Small functions**: if a function needs an inline comment to explain a block, extract that block into a named function.
- **No dead code**: remove unused functions, variables, and commented-out code.
- **`Annotated` for CLI options**: always use `Annotated[type, typer.Option(...)]` — never positional `typer.Option()` as a default value.

## Testing

### Framework

- **Test runner**: [`pytest`](https://docs.pytest.org/) — industry standard, required for all projects.
- **Coverage**: `pytest-cov` — enforce a minimum threshold; fail the build below it.
- **Mocking**: `unittest.mock` (stdlib) — use `patch` and `MagicMock`. `pytest-mock` is optional but acceptable.
- **CLI integration**: `typer.testing.CliRunner` — test the full CLI invocation, not individual functions.

Add to `pyproject.toml`:

```toml
[project.optional-dependencies]
dev = ["pytest", "pytest-cov"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=<pkg> --cov-report=term-missing --cov-fail-under=80"
```

Run tests:

```bash
# Run all tests with coverage
pytest

# Run a specific test file
pytest tests/test_account.py -v
```

### Test Structure

Two layers — unit tests for pure functions, integration tests for full CLI invocations:

```
tests/
├── unit/
│   └── test_<resource>_helpers.py   # builder and renderer functions
└── integration/
    └── test_<resource>.py           # full CLI via CliRunner
```

**Unit tests** target pure functions (builders, renderers) with no mocking:

```python
from <pkg>.commands.aws.account import build_alias, render_result

def test_build_alias_format() -> None:
    alias = build_alias("my-account")
    assert alias.startswith("my-account-")

def test_render_result_json(capsys: pytest.CaptureFixture[str]) -> None:
    render_result("json", "my-account-abc123", created=True)
    captured = capsys.readouterr()
    data = json.loads(captured.out)
    assert data == {"alias": "my-account-abc123", "created": True}
```

**Integration tests** test the full CLI invocation via `CliRunner`, mocking only external I/O:

```python
from typer.testing import CliRunner
from unittest.mock import patch, MagicMock
from <pkg>.cli import app

runner = CliRunner()

@patch("<pkg>.commands.aws.account.petname.Generate", return_value="brave-lion")
@patch("<pkg>.context.boto3.Session")
def test_set_alias_creates_new(mock_session: MagicMock, mock_petname: MagicMock) -> None:
    iam_mock = MagicMock()
    iam_mock.list_account_aliases.return_value = {"AccountAliases": []}
    mock_session.return_value.client.return_value = iam_mock

    result = runner.invoke(app, ["aws", "account", "set-alias", "my-account"])

    assert result.exit_code == 0
    assert "my-account-brave-lion" in result.output
    iam_mock.create_account_alias.assert_called_once_with(AccountAlias="my-account-brave-lion")
```

### Required Test Cases

Every command must have tests covering:

1. **Happy path** — resource created/action taken, correct output, exit code 0.
2. **Idempotent path** — resource already exists, no duplicate action taken.
3. **JSON output** — `--output json` returns valid JSON with expected keys.
4. **Error path** — external API failure exits with code 1 and writes message to stderr.

### When to Run Tests

- **Before every commit** — tests must pass locally before pushing.
- **In CI** — the pipeline must run `pytest` and fail on any test failure or coverage drop.
- **When adding a command** — write tests before or alongside the implementation (not after).

## Project Conventions

- `__version__` lives in `<pkg>/__init__.py` only. `cli.py` imports it for `--version`.
- Entry point registered in `pyproject.toml` under `[project.scripts]`: `<cmd> = "<pkg>.cli:app"`.
- `no_args_is_help=True` on every `typer.Typer()` instance, including sub-apps.
- Command names use kebab-case (`set-alias`, not `set_alias`).
- Module names use snake_case (`account.py`, not `Account.py`).
