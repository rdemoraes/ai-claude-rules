# Bash Scripts — Coding Standards

Applies to all `*.sh` files. Mirrors the same SOLID + clean code principles used in the Python toolkit.

## Core Principle: Keep It Simple

Write the simplest script that solves the problem. No abstractions without a concrete, immediate need.

## File Header

Every script must start with:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

`set -e` exits on error, `-u` errors on unset variables, `-o pipefail` propagates pipe failures.

## Project Structure

New domain = new directory under `lib/`. New resource = new file under `lib/<domain>/`.
The entry point only sources modules and calls `main`.

```
scripts/
├── main.sh                   # Entry point — sources modules, calls main
├── lib/
│   ├── core/
│   │   ├── logging.sh        # log_info, log_warn, log_error
│   │   └── errors.sh         # error handling helpers
│   └── <domain>/
│       └── <resource>.sh     # domain-specific functions
└── tests/
    └── <resource>_test.sh
```

```bash
# BAD — everything in one file
main() {
  local region="${AWS_REGION:-us-east-1}"
  aws sts get-caller-identity --region "$region" | jq '.Account'
  aws iam list-account-aliases --region "$region" | jq '.AccountAliases[0]'
}

# GOOD — entry point sources modules and delegates
source "$(dirname "$0")/lib/core/logging.sh"
source "$(dirname "$0")/lib/aws/account.sh"

main() {
  local region="${1:-us-east-1}"
  local account_id
  account_id="$(aws_get_account_id "$region")"
  log_info "Account: $account_id"
}
```

## SOLID in Practice

**Single Responsibility** — each function does one thing. Functions that interact with external tools (aws, kubectl, etc.) are pure actions. Functions that format output are pure renderers.

```bash
# BAD — function does too much
set_account_alias() {
  local name="$1"
  local region="${2:-us-east-1}"
  existing=$(aws iam list-account-aliases --region "$region" \
    --query 'AccountAliases[0]' --output text 2>/dev/null)
  if [[ "$existing" != "None" && -n "$existing" ]]; then
    echo "Alias already set: $existing"
    return
  fi
  suffix=$(tr -dc 'a-z0-9' </dev/urandom | head -c 6)
  alias="${name}-${suffix}"
  aws iam create-account-alias --account-alias "$alias" --region "$region"
  echo "Set alias: $alias"
}

# GOOD — each function has one job
get_existing_alias() {
  local region="$1"
  aws iam list-account-aliases --region "$region" \
    --query 'AccountAliases[0]' --output text 2>/dev/null || true
}

build_alias() {
  local name="$1"
  local suffix
  suffix="$(tr -dc 'a-z0-9' </dev/urandom | head -c 6)"
  echo "${name}-${suffix}"
}

create_alias() {
  local alias="$1"
  local region="$2"
  aws iam create-account-alias --account-alias "$alias" --region "$region"
}

render_alias_result() {
  local alias="$1"
  local created="$2"
  local status="created"
  [[ "$created" == "false" ]] && status="already set"
  echo "Account alias (${status}): ${alias}"
}

set_account_alias() {
  local name="$1"
  local region="${2:-us-east-1}"
  local existing
  existing="$(get_existing_alias "$region")"
  if [[ "$existing" != "None" && -n "$existing" ]]; then
    render_alias_result "$existing" "false"
    return
  fi
  local alias
  alias="$(build_alias "$name")"
  create_alias "$alias" "$region"
  render_alias_result "$alias" "true"
}
```

**Open/Closed** — extend by adding new files in `lib/<domain>/`, never by modifying `main.sh` or existing lib files. A new AWS resource = new file `lib/aws/<resource>.sh`.

**Dependency on abstractions** — functions receive all external dependencies (region, profile, config) as arguments. Never read from global variables or environment directly inside domain functions.

```bash
# BAD — implicit global dependency
get_account_id() {
  aws sts get-caller-identity --region "$AWS_REGION" --query Account --output text
}

# GOOD — explicit argument
get_account_id() {
  local region="$1"
  aws sts get-caller-identity --region "$region" --query Account --output text
}
```

## Naming Conventions

Follows the [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html) — the most widely adopted Bash standard.

| Construct | Convention | Example |
|-----------|-----------|---------|
| Local variables | `snake_case` | `account_name`, `existing_alias` |
| Functions | `snake_case` | `get_existing_alias`, `build_alias` |
| Global variables | `UPPER_SNAKE_CASE` | `OUTPUT_FORMAT`, `AWS_PROFILE` |
| Constants (`readonly`) | `UPPER_SNAKE_CASE` | `readonly DEFAULT_REGION="us-east-1"` |
| Environment variables | `UPPER_SNAKE_CASE` | `AWS_REGION`, `KUBECONFIG` |

```bash
# BAD
accountName="prod"           # camelCase — not idiomatic Bash
Account_Name="prod"          # mixed — confusing
EXISTING_alias=""            # mixed case — pick one

# GOOD
account_name="prod"          # local variable: snake_case
readonly DEFAULT_REGION="us-east-1"   # constant: UPPER_SNAKE_CASE
OUTPUT_FORMAT="${OUTPUT_FORMAT:-text}" # global/env: UPPER_SNAKE_CASE

get_existing_alias() { ... } # function: snake_case
```

**Key rule**: `UPPER_SNAKE_CASE` signals "global or exported". `snake_case` signals "local to this function or script". Never use `UPPER_SNAKE_CASE` for `local` variables.

```bash
# BAD — uppercase local looks like a global
set_account_alias() {
  local ALIAS="$1"      # misleading — looks like an env var
}

# GOOD
set_account_alias() {
  local alias="$1"      # clearly local
}
```

## Clean Code Rules

- **Meaningful names**: `get_existing_alias` not `check`. `account_name` not `n`.
- **`local` for all variables**: never pollute the calling scope.
- **Small functions**: if a function needs a comment to explain a section, extract that section into a named function.
- **No dead code**: remove unused functions, variables, and commented-out code.
- **Explicit error handling**: check exit codes, print a clear message, exit with a non-zero code.

```bash
# BAD
aws iam create-account-alias --account-alias "$alias" || echo "failed"

# GOOD
create_alias() {
  local alias="$1"
  local region="$2"
  if ! aws iam create-account-alias --account-alias "$alias" --region "$region"; then
    log_error "Failed to create alias '${alias}' in region '${region}'"
    exit 1
  fi
}
```

## Logging Convention

Use consistent logging functions from `lib/core/logging.sh`:

```bash
log_info()  { echo "[INFO]  $*" >&1; }
log_warn()  { echo "[WARN]  $*" >&2; }
log_error() { echo "[ERROR] $*" >&2; }
```

Never use bare `echo` for status messages in domain functions — only in render functions.

## Testing

### Framework

Use [`bats-core`](https://github.com/bats-core/bats-core) — the industry-standard test framework for Bash. It provides TAP-compliant output, `setup`/`teardown` hooks, and first-class CI support.

Install:

```bash
# macOS
brew install bats-core

# or via git submodule (portable, no system dependency)
git submodule add https://github.com/bats-core/bats-core.git tests/bats
```

Run tests:

```bash
# Run all tests
bats tests/

# Run a single test file
bats tests/account_test.bats -v
```

### Project Structure

```
scripts/
├── main.sh
├── lib/
│   └── aws/
│       └── account.sh
└── tests/
    ├── bats/                        # bats-core submodule (if not installed globally)
    ├── helpers/
    │   └── mock.sh                  # shared mock helpers
    └── aws/
        └── account_test.bats        # tests for lib/aws/account.sh
```

### Test File Structure

Each `.bats` file tests one `lib/<domain>/<resource>.sh` module. Source only the file under test — never `main.sh`.

```bash
#!/usr/bin/env bats

setup() {
  # Source the module under test
  source "${BATS_TEST_DIRNAME}/../../lib/aws/account.sh"
  # Source shared mock helpers
  source "${BATS_TEST_DIRNAME}/../helpers/mock.sh"
}

teardown() {
  # Reset any stubs/mocks
  unset -f aws
}
```

### Mocking External Commands

Override external commands (e.g., `aws`, `kubectl`) by defining a shell function with the same name inside the test. Restore them in `teardown`.

```bash
@test "get_existing_alias returns alias when one exists" {
  aws() {
    echo "my-alias"
  }

  run get_existing_alias "us-east-1"

  [ "$status" -eq 0 ]
  [ "$output" = "my-alias" ]
}

@test "get_existing_alias returns empty when none exist" {
  aws() {
    echo "None"
  }

  run get_existing_alias "us-east-1"

  [ "$status" -eq 0 ]
  [ "$output" = "None" ]
}

@test "create_alias exits 1 on aws failure" {
  aws() {
    return 1
  }

  run create_alias "my-alias" "us-east-1"

  [ "$status" -eq 1 ]
}
```

Use `run` before every function call under test — it captures `$status` and `$output` without aborting the test on failure.

### Required Test Cases

Every function in `lib/<domain>/<resource>.sh` must have tests covering:

1. **Happy path** — correct output and exit code 0.
2. **Idempotent/no-op path** — resource already exists, no action taken.
3. **Error path** — external command fails, function exits with code 1 and logs an error.

### When to Run Tests

- **Before every commit** — `bats tests/` must pass locally before pushing.
- **In CI** — the pipeline must run `bats tests/` and fail on any test failure.
- **When adding a function** — write the test alongside the implementation, not after.

## Project Conventions

- All scripts respect a `--output` flag (`text` or `json`). Render functions handle both formats.
- Global options (`--region`, `--profile`, `--output`) are parsed once in `main.sh`; never re-parsed inside lib functions.
- `main()` is always the last thing called in an entry-point script.
- Constants use `readonly`: `readonly DEFAULT_REGION="us-east-1"`.
- Boolean flags use `true`/`false` strings, not `0`/`1`, for readability.
