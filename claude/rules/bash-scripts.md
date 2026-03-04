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

## Project Conventions

- All scripts respect a `--output` flag (`text` or `json`). Render functions handle both formats.
- Global options (`--region`, `--profile`, `--output`) are parsed once in `main.sh`; never re-parsed inside lib functions.
- `main()` is always the last thing called in an entry-point script.
- Constants use `readonly`: `readonly DEFAULT_REGION="us-east-1"`.
- Boolean flags use `true`/`false` strings, not `0`/`1`, for readability.
