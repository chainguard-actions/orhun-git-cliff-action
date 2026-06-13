<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.7.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **orhun--git-cliff-action/v4.7.0** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The 'Run git-cliff' step in action.yml directly interpolates `${{ inputs.config }}` and `${{ inputs.args }}` inside the `run:` shell command string. These are user-controlled inputs that are template-substituted before the shell parses the command, enabling arbitrary shell command injection. The offending line is: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. Both inputs should be passed via `env:` variables and then referenced as quoted shell variables (e.g., `"$CONFIG"` and `"$ARGS"`).

Locations:

- `action.yml:45`

### github-env-injection (severity: high)

run.sh writes the `version` output to $GITHUB_OUTPUT using `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT`. The version value is extracted from git-cliff's JSON context, which is derived from git commit history and tags — data that can be controlled by an attacker via crafted commit messages or tag names containing newline characters. No sanitization (`printf '%s' ... | tr -d '\n\r'`) is applied before the write, allowing a malicious value to inject additional key=value pairs into $GITHUB_OUTPUT and potentially override subsequent outputs.

Locations:

- `run.sh:75`

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

Fixed two files: (1) action.yml: Moved `${{ inputs.config }}` and `${{ inputs.args }}` from the `run:` shell command into the `env:` block as `CONFIG` and `ARGS` environment variables. The run command now uses `--config="$CONFIG" $ARGS` — CONFIG is double-quoted to prevent word-splitting, ARGS is intentionally unquoted to allow multiple space-separated arguments to be passed correctly. (2) run.sh: Replaced the unsafe `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT` with a two-step approach: first extract and sanitize the version via `safe_version=$(jq -r '.[0].version' "$CONTEXT" | tr -d '\n\r')`, then write it safely with `printf 'version=%s\n' "$safe_version" >> "$GITHUB_OUTPUT"`. This prevents newline injection from attacker-controlled git tags/commit messages.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed three security findings in action.yml and run.sh:

1. **script-injection (action.yml)**: Replaced unquoted `$ARGS` in the run command with `read -ra args_array <<< "$ARGS"` to safely split user-controlled input into an array, then passed `"${args_array[@]}"` (properly quoted) to run.sh. This prevents shell word-splitting and glob-expansion injection.

2. **github-env-injection (run.sh line 63)**: Added newline sanitization for `$OUTPUT` before writing to GITHUB_OUTPUT: `safe_output=$(printf '%s' "$OUTPUT" | tr -d '\n\r')` then `echo "changelog=$safe_output"`. This prevents newline-based injection of arbitrary key=value pairs.

3. **github-env-injection (run.sh line 57)**: Replaced the fixed `EOF` heredoc delimiter with a randomized one generated via `/dev/urandom`: `DELIM="EOF_$(head -c 16 /dev/urandom | base64 | tr -dc 'A-Za-z0-9' | head -c 16)"`. This makes it statistically impossible for changelog content to match the delimiter, preventing premature heredoc termination and environment injection.

