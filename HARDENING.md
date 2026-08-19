<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.7.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.7.0** was hardened automatically. 7 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct expression interpolation in a run: shell command. In action.yml, the 'Run git-cliff' step interpolates `${{ inputs.config }}` and `${{ inputs.args }}` directly into the shell command string: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. An attacker-controlled calling workflow can supply arbitrary shell metacharacters via these inputs, enabling command injection.

Locations:

- `action.yml:43`

### script-injection (severity: high)

Sub-rule (a): Direct expression interpolation in run: shell commands in the workflow. The 'Print the changelog' step uses `run: cat "${{ steps.git-cliff.outputs.changelog }}"` and the 'Print the version' step uses `run: echo "${{ steps.git-cliff.outputs.version }}"`. Step outputs are workflow-controllable values and must not be interpolated directly into shell commands — they should be passed via env: variables with double-quoted expansions.

Locations:

- `.github/workflows/main.yml:18`
- `.github/workflows/main.yml:20`

### github-env-injection (severity: high)

In run.sh, the inherited process env var $OUTPUT (set by the calling workflow via `env: OUTPUT: fixtures/CHANGELOG.md`) is written to $GITHUB_OUTPUT without sanitization: `echo "changelog=$OUTPUT" >> $GITHUB_OUTPUT`. Additionally, `echo "version=$(jq -r '.[0].version' $CONTEXT)" >> $GITHUB_OUTPUT` writes externally-derived content (from a temp file populated by git-cliff processing of the repo) to $GITHUB_OUTPUT without sanitization. Neither write is preceded by the required `printf '%s' ... | tr -d '\n\r'` sanitization step, allowing newline injection to poison the output file.

Locations:

- `run.sh:63`
- `run.sh:66`

### unpinned-uses (severity: high)

The workflow uses `actions/checkout@v6` — a mutable tag reference rather than a pinned 40-character commit SHA. A tag can be moved to point to a different (potentially malicious) commit, enabling a supply-chain attack. It should be pinned to a full SHA, e.g. `actions/checkout@<40-hex-sha> # v6`.

Locations:

- `.github/workflows/main.yml:10`

### missing-permissions (severity: medium)

The workflow file .github/workflows/main.yml has no top-level `permissions:` key and the single job `git-cliff-action` also has no job-level `permissions:` key. Without explicit permissions, the workflow inherits the repository default (typically `contents: write` for push events), granting broader access than necessary. A minimal `permissions:` block (e.g. `contents: read`) should be added.

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

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions, static-inline-injection

**Notes:**

Fixed all 7 findings across 3 files:

1. action.yml (script-injection / static-inline-injection): Moved `${{ inputs.config }}` and `${{ inputs.args }}` out of the `run:` shell string into an `env:` block as GIT_CLIFF_CONFIG and GIT_CLIFF_ARGS. The run script now builds a bash array and passes arguments safely without shell word-splitting risks.

2. .github/workflows/main.yml (script-injection): Moved `${{ steps.git-cliff.outputs.changelog }}` and `${{ steps.git-cliff.outputs.version }}` into `env:` blocks (CHANGELOG, VERSION) and referenced them as plain shell variables.

3. .github/workflows/main.yml (unpinned-uses): Pinned `actions/checkout@v6` to full commit SHA `d23441a48e516b6c34aea4fa41551a30e30af803 # v6`.

4. .github/workflows/main.yml (missing-permissions): Added top-level `permissions: contents: read`.

5. run.sh (github-env-injection): Sanitized both the `$OUTPUT` path and the jq-derived version string with `printf '%s' ... | tr -d '\n\r'` before writing to `$GITHUB_OUTPUT`.

