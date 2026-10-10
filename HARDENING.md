<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.7.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.7.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): In the 'Run git-cliff' step of action.yml, the expressions `${{ inputs.config }}` and `${{ inputs.args }}` are directly interpolated into the `run:` shell command string: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. An attacker who controls these inputs (e.g. via workflow_dispatch or a calling workflow) can inject arbitrary shell metacharacters (`;`, `|`, `$(...)`, etc.) that are evaluated by bash before any quoting takes effect. Both inputs should be moved to `env:` variables and referenced as quoted `"$VAR"` expansions instead.

Locations:

- `action.yml:43`

### github-env-injection (severity: high)

In run.sh, the variable `$OUTPUT` is written to `$GITHUB_OUTPUT` without sanitization. `$OUTPUT` can be set from the user-controlled `inputs.args` input (parsed via `--output=<value>` or `-o <value>` from `$@`, which comes from `${{ inputs.args }}`). The unsanitized write `echo "changelog=$OUTPUT" >> $GITHUB_OUTPUT` allows a newline character embedded in the output path to inject arbitrary key=value pairs into the GitHub output context. The required sanitization step (`printf '%s' "$OUTPUT" | tr -d '\n\r'`) is absent before this write.

Locations:

- `run.sh:72`

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
1. action.yml: Moved `${{ inputs.config }}` and `${{ inputs.args }}` out of the run: shell string into env: variables (INPUT_CONFIG and INPUT_ARGS). Since `inputs.args` is a whitespace-separated list of arguments, it is tokenized using the xargs/while-read-NUL pattern into a bash array before being passed to run.sh. `inputs.config` is a single value passed as `--config=${INPUT_CONFIG}`.
2. run.sh: Added sanitization of the $OUTPUT variable before writing to $GITHUB_OUTPUT: `safe_output=$(printf '%s' "$OUTPUT" | tr -d '\n\r')` followed by `echo "changelog=$safe_output" >>$GITHUB_OUTPUT`, preventing newline injection attacks.

