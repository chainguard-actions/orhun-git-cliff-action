<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.7.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.7.1** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): In the 'Run git-cliff' step of action.yml, the inputs `inputs.config` and `inputs.args` are directly interpolated into the `run:` shell command string via `${{ inputs.config }}` and `${{ inputs.args }}`. Because GitHub Actions performs YAML template substitution before the shell ever sees the string, an attacker who controls these inputs can inject arbitrary shell commands. The offending line is: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. These inputs should be passed via `env:` variables and then referenced as quoted shell variables (e.g., `"$CONFIG"` and `"$ARGS"`).

Locations:

- `action.yml:43`

### github-env-injection (severity: high)

In run.sh, multiple writes to $GITHUB_OUTPUT use values derived from attacker-controlled inputs without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). Specifically: (1) `cat "$OUTPUT" >>$GITHUB_OUTPUT` (line 66) and `echo "changelog=$OUTPUT" >>$GITHUB_OUTPUT` (line 71) write `$OUTPUT`, which is derived from the command-line arguments passed from `${{ inputs.args }}` and `${{ inputs.config }}` in action.yml — an attacker can inject newlines to poison subsequent GITHUB_OUTPUT entries. (2) `echo "version=$(jq -r '.[0].version' $CONTEXT)" >>$GITHUB_OUTPUT` (line 74) writes jq output (from git-cliff's generated context) without sanitization, allowing newline injection if the version string contains newlines. None of these writes are preceded by the required `tr -d '\n\r'` sanitization.

Locations:

- `run.sh:66`
- `run.sh:71`
- `run.sh:74`

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

action.yml: Moved ${{ inputs.config }} and ${{ inputs.args }} from the run: shell string into env: variables (INPUT_CONFIG, INPUT_ARGS). The run: block now uses bash arrays with xargs-based NUL-delimited tokenization to safely reconstruct the --config=... flag and split the args list into separate arguments, preserving argument boundaries and handling quoted values. run.sh: Added sanitization for the 'changelog' output (safe_output via tr -d '\n\r') and the 'version' output (raw_version captured first, then safe_version via tr -d '\n\r') before writing to $GITHUB_OUTPUT. The multiline 'content' heredoc block was left as-is since it uses the correct GitHub Actions multiline output format.

