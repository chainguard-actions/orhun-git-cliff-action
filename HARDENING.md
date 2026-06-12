<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.7.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **orhun--git-cliff-action/v4.7.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The 'Run git-cliff' step in action.yml directly interpolates user-controlled inputs into the run: shell command string without routing through env: variables. Specifically, `${{ inputs.config }}` and `${{ inputs.args }}` are template-substituted directly into the shell command: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. An attacker calling this composite action can supply values like `; malicious-command #` in inputs.args or inputs.config to achieve arbitrary command execution on the runner.

Locations:

- `action.yml:43`

### github-env-injection (severity: high)

run.sh writes multiple unsanitized values to $GITHUB_OUTPUT without the required `printf '%s' ... | tr -d '\n\r'` sanitization step:

(1) Line ~57: `echo "content<<EOF" >> $GITHUB_OUTPUT` followed by `cat "$OUTPUT" >> $GITHUB_OUTPUT` — the full contents of the output file (whose path is derived from the user-controlled --output argument passed via inputs.args) are written verbatim to $GITHUB_OUTPUT. A newline in the content can break the heredoc delimiter and inject arbitrary key=value pairs into the output context.

(2) Line ~63: `echo "changelog=$OUTPUT" >> $GITHUB_OUTPUT` — $OUTPUT is derived from the user-controlled --output argument (via inputs.args) and is written without sanitization.

(3) Line ~66: `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT` — the version string extracted from git-cliff's JSON context output is written without sanitization; a crafted version string containing newlines could inject additional output variables.

Locations:

- `run.sh:57`
- `run.sh:63`
- `run.sh:66`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.config }}" appears directly in run: block of step "Run git-cliff"; move to env: map

Locations:

- `action.yml:46`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.args }}" appears directly in run: block of step "Run git-cliff"; move to env: map

Locations:

- `action.yml:46`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, static-inline-injection, github-env-injection

**Notes:**

Fixed all four findings:

1. action.yml (script-injection + static-inline-injection): Moved `${{ inputs.config }}` and `${{ inputs.args }}` out of the `run:` shell command string into the `env:` block as `GIT_CLIFF_CONFIG` and `GIT_CLIFF_ARGS`. The run block now safely builds a bash array from these env vars and passes them to run.sh.

2. run.sh (github-env-injection, 3 locations):
   - Line ~57: Replaced static 'EOF' heredoc delimiter with a randomly-generated delimiter (`EOF_<random>`) to prevent content-based delimiter injection.
   - Line ~63: Added `printf '%s' "$OUTPUT" | tr -d '\n\r'` sanitization before writing `changelog=` to `$GITHUB_OUTPUT`.
   - Line ~66: Added `printf '%s' "$raw_version" | tr -d '\n\r'` sanitization before writing `version=` to `$GITHUB_OUTPUT`.

