<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.8.0** was hardened automatically. 7 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The 'Run git-cliff' step in action.yml directly interpolates `${{ inputs.config }}` and `${{ inputs.args }}` into the `run:` shell command string. Both are user-controlled inputs that can contain shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.), enabling command injection by any caller of this composite action. Offending line: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`

Locations:

- `action.yml:40`

### script-injection (severity: high)

Sub-rule (a): Two `run:` steps in main.yml directly interpolate `${{ steps.git-cliff.outputs.changelog }}` and `${{ steps.git-cliff.outputs.version }}` into shell commands. Even though these come from a prior step's outputs, they flow through YAML template substitution before the shell sees them and can contain shell metacharacters. Offending lines: `run: cat "${{ steps.git-cliff.outputs.changelog }}"` (line 22) and `run: echo "${{ steps.git-cliff.outputs.version }}"` (line 24).

Locations:

- `.github/workflows/main.yml:22`
- `.github/workflows/main.yml:24`

### github-env-injection (severity: high)

run.sh writes inherited/external values to $GITHUB_OUTPUT without the required sanitization (`printf '%s' ... | tr -d '\n\r'`). (1) Line 68: `echo "changelog=$OUTPUT" >> $GITHUB_OUTPUT` — `$OUTPUT` is an env var inherited from the calling workflow (set via `env: OUTPUT: ...`) and is therefore workflow-controlled/untrusted. (2) Line 71: `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT` — the jq output from git-cliff's JSON context file can contain newlines, which can break the GITHUB_OUTPUT key=value protocol and allow injection of additional output variables. Neither write is preceded by the required sanitization step.

Locations:

- `run.sh:68`
- `run.sh:71`

### unpinned-uses (severity: high)

main.yml uses `actions/checkout@v6.0.2`, which is pinned to a mutable version tag rather than an immutable 40-character commit SHA. A tag can be moved to point to a different (potentially malicious) commit, enabling a supply-chain attack.

Locations:

- `.github/workflows/main.yml:12`

### missing-permissions (severity: medium)

main.yml has no top-level `permissions:` key and no job-level `permissions:` key on the `git-cliff-action` job. Without explicit permissions, the workflow inherits the repository's default token permissions (which may be `write-all`), violating the principle of least privilege.

Locations:

- `.github/workflows/main.yml:1`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.config }}" appears directly in run: block of step "Run git-cliff"; move to env: map

Locations:

- `action.yml:45`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.args }}" appears directly in run: block of step "Run git-cliff"; move to env: map

Locations:

- `action.yml:45`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions, static-inline-injection

**Notes:**

Fixed all 7 findings across 3 files:

1. action.yml (script-injection / static-inline-injection): Moved `${{ inputs.config }}` and `${{ inputs.args }}` from the `run:` shell string into the `env:` block as `GIT_CLIFF_CONFIG` and `GIT_CLIFF_ARGS`. The run block now builds a bash array safely and passes arguments without shell word-splitting risks.

2. main.yml (script-injection): Moved `${{ steps.git-cliff.outputs.changelog }}` and `${{ steps.git-cliff.outputs.version }}` from `run:` shell strings into `env:` blocks (`CHANGELOG` and `VERSION`), then referenced as plain env vars.

3. main.yml (unpinned-uses): Pinned `actions/checkout@v6.0.2` to full SHA `de0fac2e4500dabe0009e67214ff5f5447ce83dd` with tag preserved as comment.

4. main.yml (missing-permissions): Added top-level `permissions: {}` and job-level `permissions: contents: read`.

5. run.sh (github-env-injection): Sanitized both GITHUB_OUTPUT writes — `$OUTPUT` via `printf '%s' "$OUTPUT" | tr -d '\n\r'` and the jq version output via `| tr -d '\n\r'` before writing to GITHUB_OUTPUT.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted $OUTPUT variable inside command substitution on line 34 of run.sh. Changed `mkdir -p "$(dirname $OUTPUT)"` to `mkdir -p "$(dirname "$OUTPUT")"`. The unquoted expansion inside the command substitution allowed word splitting and glob expansion on the OUTPUT variable, which can be controlled by calling workflows. Quoting it properly prevents this attack vector.

