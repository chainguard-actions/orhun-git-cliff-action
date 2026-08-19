<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.7.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.7.1** was hardened automatically. 6 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The 'Run git-cliff' step in action.yml directly interpolates `${{ inputs.config }}` and `${{ inputs.args }}` into the `run:` shell command string: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. An attacker-controlled caller can supply values containing shell metacharacters (`;`, `|`, `$(...)`, etc.) in these inputs, achieving arbitrary command execution on the runner.

Locations:

- `action.yml:38`

### script-injection (severity: high)

Sub-rule (a): Two `run:` steps in the workflow directly interpolate `${{ steps.git-cliff.outputs.changelog }}` and `${{ steps.git-cliff.outputs.version }}` into shell commands. Line 22: `run: cat "${{ steps.git-cliff.outputs.changelog }}"`. Line 24: `run: echo "${{ steps.git-cliff.outputs.version }}". Step outputs flow through YAML template substitution before the shell sees them, making them susceptible to injection. These should be passed via env vars and referenced as `"$VAR"` instead.

Locations:

- `.github/workflows/main.yml:22`
- `.github/workflows/main.yml:24`

### unpinned-uses (severity: high)

The workflow references `actions/checkout@v6` using a mutable tag instead of a full 40-character commit SHA. A tag can be moved to point to a different (potentially malicious) commit, enabling a supply-chain attack. Pin to a specific SHA, e.g. `actions/checkout@<40-char-sha> # v6`.

Locations:

- `.github/workflows/main.yml:10`

### missing-permissions (severity: medium)

The workflow file `.github/workflows/main.yml` has no top-level `permissions:` key and the single job `git-cliff-action` also has no `permissions:` key. Without explicit permissions, the workflow inherits the repository default (often `write-all` for private repos or `read-all` for public repos), granting broader access than necessary. Add a top-level `permissions: {}` block and grant only the minimum required scopes (e.g. `contents: read`).

Locations:

- `.github/workflows/main.yml:1`

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

**Fixes applied:** script-injection, static-inline-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all 6 findings across action.yml and .github/workflows/main.yml:

1. action.yml script-injection (inputs.config, inputs.args): Moved both inputs to env vars (GIT_CLIFF_CONFIG, GIT_CLIFF_ARGS) in the 'Run git-cliff' step. The run script now builds a bash array safely: --config is only added when non-empty using ${VAR:+...} pattern, and extra args are split via read -ra to keep each token separate while preventing shell metacharacter injection.

2. main.yml script-injection (steps.git-cliff.outputs.changelog, steps.git-cliff.outputs.version): Moved both step outputs to env vars (CHANGELOG, VERSION) and referenced them as plain shell variables in the run commands.

3. main.yml unpinned-uses: Pinned actions/checkout@v6 to full SHA d23441a48e516b6c34aea4fa41551a30e30af803 with # v6 comment for readability.

4. main.yml missing-permissions: Added top-level `permissions: {}` and job-level `permissions: contents: read` (minimum needed for checkout).

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Fixed github-env-injection in run.sh at line 79. The $OUTPUT variable (which can be inherited from the calling workflow) was written directly to $GITHUB_OUTPUT without sanitization. Fixed by adding a sanitization step: `safe_output=$(printf '%s' "$OUTPUT" | tr -d '\n\r')` and then writing `safe_output` instead of `$OUTPUT` to $GITHUB_OUTPUT. This prevents newline injection attacks that could poison the output file with arbitrary key=value pairs.

