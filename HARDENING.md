<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.6.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **orhun--git-cliff-action/v4.6.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): In action.yml's 'Run git-cliff' step, the `run:` shell command directly interpolates `${{ inputs.config }}` and `${{ inputs.args }}` into the shell command string without routing them through env vars. An attacker-controlled value in either input can inject arbitrary shell commands. Offending line: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`

Locations:

- `action.yml:43`

### github-env-injection (severity: high)

In run.sh, the changelog file content (from `$OUTPUT`, which is derived from git history and thus attacker-controllable via commit messages) is written directly to $GITHUB_OUTPUT using a heredoc (`cat "$OUTPUT" >> $GITHUB_OUTPUT`) without sanitization. Additionally, `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT` writes jq-extracted content from git-cliff's JSON context (also derived from git history) to $GITHUB_OUTPUT without the required `printf '%s' ... | tr -d '\n\r'` sanitization. Newlines in these values can inject additional key=value pairs into GITHUB_OUTPUT, potentially overwriting other outputs.

Locations:

- `run.sh:55`
- `run.sh:61`

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

**Fixes applied:** script-injection, github-env-injection, static-inline-injection

**Notes:**

Fixed all four findings:

1. action.yml: Moved `${{ inputs.config }}` and `${{ inputs.args }}` from the `run:` shell string into the `env:` block as `GIT_CLIFF_CONFIG` and `GIT_CLIFF_ARGS`. The run command now uses `--config="$GIT_CLIFF_CONFIG" $GIT_CLIFF_ARGS` referencing plain env vars.

2. run.sh line 55 (heredoc): Replaced the static `EOF` delimiter with a cryptographically random `EOF_$(openssl rand -hex 16)` delimiter to prevent attacker-controlled changelog content (from git commit messages) from escaping the heredoc block.

3. run.sh line 61 (version): Added sanitization of the jq-extracted version string using `printf '%s' "$raw_version" | tr -d '\n\r'` before writing to `$GITHUB_OUTPUT`, preventing newline injection of additional key=value pairs.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed two high-severity findings: (1) In action.yml, replaced unquoted `$GIT_CLIFF_ARGS` expansion in the run: command with `read -ra cliff_args <<< "$GIT_CLIFF_ARGS"` and `"${cliff_args[@]}"` to safely split and pass args as a quoted array, preventing shell metacharacter injection. (2) In run.sh, added `safe_output=$(printf '%s' "$OUTPUT" | tr -d '\n\r')` before writing the changelog output path to $GITHUB_OUTPUT, stripping newlines to prevent injection of additional key=value pairs — consistent with how the version output was already being sanitized.

