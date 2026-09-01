<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.9.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.9.0** was hardened automatically. 7 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

The 'Run git-cliff' step in action.yml directly interpolates `${{ inputs.config }}` and `${{ inputs.args }}` into the `run:` shell command string: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. An attacker-controlled caller can supply shell metacharacters via these inputs, enabling command injection. These values must be passed via `env:` variables and then double-quoted in the shell script.

Locations:

- `action.yml:38`

### script-injection (severity: high)

Two `run:` steps in main.yml directly interpolate `steps.*` output expressions into shell commands: `run: cat "${{ steps.git-cliff.outputs.changelog }}"` and `run: echo "${{ steps.git-cliff.outputs.version }}". Any `${{ ... }}` expression inside a `run:` block is a script-injection risk (sub-rule a), as the value is substituted by the template engine before the shell parses it. These should be passed via `env:` and referenced as quoted shell variables.

Locations:

- `.github/workflows/main.yml:20`
- `.github/workflows/main.yml:22`

### unpinned-uses (severity: high)

The workflow uses `actions/checkout@v7.0.1`, which is pinned to a mutable version tag rather than an immutable 40-character commit SHA. A tag can be moved to point to a different (potentially malicious) commit. It should be pinned to a full SHA, e.g. `actions/checkout@<40-char-sha> # v7.0.1`.

Locations:

- `.github/workflows/main.yml:10`

### missing-permissions (severity: medium)

The workflow file main.yml has no top-level `permissions:` key and no job-level `permissions:` key on the `git-cliff-action` job. Without explicit permissions, the workflow inherits the repository's default token permissions, which may be overly broad. A minimal `permissions:` block (e.g. `contents: read`) should be added.

Locations:

- `.github/workflows/main.yml:1`

### github-env-injection (severity: high)

In sync-git-cliff-version.yml, the 'Get latest git-cliff release' step writes `VERSION` (fetched from an external GitHub API response) to `$GITHUB_OUTPUT` without sanitization: `echo "version=${VERSION}" >> ${GITHUB_OUTPUT}`. Similarly, the 'Read current default version' step writes a `VERSION` value parsed from action.yml to `$GITHUB_OUTPUT` without sanitization. A malicious release tag name containing newlines could inject additional key-value pairs into GITHUB_OUTPUT. The value should be sanitized with `printf '%s' "$VERSION" | tr -d '\n\r'` before writing.

Locations:

- `.github/workflows/sync-git-cliff-version.yml:19`
- `.github/workflows/sync-git-cliff-version.yml:25`

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

**Fixes applied:** script-injection, unpinned-uses, missing-permissions, github-env-injection, static-inline-injection

**Notes:**

Fixed all findings:
1. action.yml (script-injection/static-inline-injection): Moved inputs.config and inputs.args from inline run: interpolation to env: vars (INPUT_CONFIG, INPUT_ARGS). INPUT_CONFIG is used with a guard and double-quoted. INPUT_ARGS is tokenized via xargs into a bash array to safely handle multi-argument lists.
2. main.yml (script-injection): Moved steps.git-cliff.outputs.changelog and steps.git-cliff.outputs.version to env: blocks (CHANGELOG, VERSION) and referenced as quoted shell variables.
3. main.yml (unpinned-uses): Pinned actions/checkout@v7.0.1 to full SHA 3d3c42e5aac5ba805825da76410c181273ba90b1 with tag comment.
4. main.yml (missing-permissions): Added top-level permissions: contents: read.
5. sync-git-cliff-version.yml (github-env-injection): Sanitized both VERSION values with printf '%s' | tr -d '\n\r' before writing to GITHUB_OUTPUT.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed two issues in hardened/action/run.sh: (1) Quoted $OUTPUT inside the dirname command substitution at line 36 to prevent word-splitting and glob expansion: `mkdir -p "$(dirname "$OUTPUT")"`. (2) Sanitized $OUTPUT before writing to $GITHUB_OUTPUT at line 81 by first stripping newlines with `safe_output=$(printf '%s' "$OUTPUT" | tr -d '\n\r')` and then writing `echo "changelog=$safe_output" >> "$GITHUB_OUTPUT"` to prevent newline injection attacks.

