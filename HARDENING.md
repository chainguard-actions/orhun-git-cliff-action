<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.8.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.8.0** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): action.yml puts input expressions straight into the run: script of the 'Run git-cliff' step: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. The values of inputs.config and inputs.args are substituted into the script before bash parses it, so shell metacharacters in either input (;, |, $(...), backticks) run as commands. Fix: move both inputs into env: variables. Expand the config as "--config=$CONFIG". Split the args list into an array with a shellcheck-annotated array assignment, then pass the array quoted, as "${ARGS[@]}".

Locations:

- `action.yml:44`

### github-env-injection (severity: high)

run.sh, which the 'Run git-cliff' step runs, writes unsanitized values to $GITHUB_OUTPUT. (1) Lines 72-74 write the changelog content between a fixed heredoc delimiter: `echo "content<<EOF"`, `cat "$OUTPUT"`, `echo "EOF"`. The changelog is built from commit messages and the inputs.args/config, so a line equal to 'EOF' ends the block early and every line after it becomes a new output entry. Use a random delimiter from /dev/urandom or openssl rand. (2) Line 79, `echo "changelog=$OUTPUT"`: OUTPUT comes from the inherited OUTPUT env var or from the -o/--output value in inputs.args, and newlines are not stripped with printf '%s' | tr -d '\n\r'. (3) Line 82 writes `version=$(jq -r '.[0].version' $CONTEXT)` without stripping newlines either; the version value comes from git tags.

Locations:

- `run.sh:72`
- `run.sh:79`
- `run.sh:82`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.config }}" appears directly in run: block of step "Run git-cliff"; move to env: map

Locations:

- `action.yml:45`

### static-inline-injection (severity: high)

shell injection: expression "${{ inputs.args }}" appears directly in run: block of step "Run git-cliff"; move to env: map

Locations:

- `action.yml:45`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index b902d76..01430c6 100644
--- a/action.yml
+++ b/action.yml
@@ -29,6 +29,23 @@ outputs:
 runs:
   using: "composite"
   steps:
     - name: Download git-cliff
       shell: bash
       run: ${GITHUB_ACTION_PATH}/install.sh
@@ -37,14 +54,14 @@ runs:
         RUNNER_ARCH: ${{ runner.arch }}
         VERSION: ${{ inputs.version }}
         GITHUB_API_TOKEN: ${{ inputs.github_token || env.GITHUB_TOKEN || github.token }}
-
     - name: Run git-cliff
       id: run-git-cliff
       shell: bash
-      run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}
+      run: ${GITHUB_ACTION_PATH}/run.sh
       env:
         GITHUB_TOKEN: ${{ inputs.github_token || env.GITHUB_TOKEN || github.token }}
-
+        GIT_CLIFF_CONFIG: ${{ inputs.config }}
+        GIT_CLIFF_ARGS: ${{ inputs.args }}
 branding:
   icon: "triangle"
   color: "green"

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed run.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/run.sh b/run.sh
index 6062a71..c55921f 100755
--- a/run.sh
+++ b/run.sh
@@ -31,11 +31,24 @@ chown -R "$(id -u)" .
 
 # Create the output directory
 OUTPUT=${OUTPUT:="git-cliff/CHANGELOG.md"}
-mkdir -p "$(dirname $OUTPUT)"
+mkdir -p "$(dirname "$OUTPUT")"
 
+# Build args array from env vars (passed via env: block in action.yml).
+# GIT_CLIFF_CONFIG is a single path value; GIT_CLIFF_ARGS is a whitespace-separated list.
+raw_args=()
+if [ -n "${GIT_CLIFF_CONFIG:-}" ]; then
+    raw_args+=(--config="$GIT_CLIFF_CONFIG")
+fi
+if [ -n "${GIT_CLIFF_ARGS:-}" ]; then
+    while IFS= read -r -d '' t; do raw_args+=("$t"); done \
+        < <(printf '%s' "$GIT_CLIFF_ARGS" | xargs printf '%s\0')
+fi
+
+# Scan raw_args for --output/-o flags to determine the output file,
+# keeping all other args for passing to git-cliff.
 args=()
 take_next=
-for arg in "$@"; do
+for arg in "${raw_args[@]}"; do
     case "$arg" in
     -o | --output)
         take_next=1 # next argument is the output file
@@ -75,11 +88,14 @@ if [ "$FILESIZE" -le "$MAXSIZE" ]; then
     cat "$OUTPUT"
 fi
 
-# Set output file
-echo "changelog=$OUTPUT" >>$GITHUB_OUTPUT
+# Set output file (sanitize path to prevent newline injection)
+safe_output="$(printf '%s' "$OUTPUT" | tr -d '\n\r')"
+echo "changelog=$safe_output" >>$GITHUB_OUTPUT
 
-# Set the version output to the version of the latest release
-echo "version=$(jq -r '.[0].version' $CONTEXT)" >>$GITHUB_OUTPUT
+# Set the version output to the version of the latest release (sanitize to prevent newline injection)
+raw_version="$(jq -r '.[0].version' "$CONTEXT")"
+safe_version="$(printf '%s' "$raw_version" | tr -d '\n\r')"
+echo "version=$safe_version" >>$GITHUB_OUTPUT
 
 # Pass exit code to the next step
 echo "exit_code=$exit_code" >>$GITHUB_OUTPUT

```

Locations:

- `run.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, static-inline-injection, github-env-injection, prior-fix

**Notes:**

action.yml: inputs.config and inputs.args now reach the 'Run git-cliff' step through env: (GIT_CLIFF_CONFIG and GIT_CLIFF_ARGS). The step builds an array from the args, splitting with xargs so quotes are honoured the way they were when upstream pasted the input into the script. It first runs a check that fails the step on an unmatched quote. It then calls run.sh "--config=$GIT_CLIFF_CONFIG" "${ARGS[@]}" (with a guard so an empty array passes nothing). run.sh still reads its arguments from $@, so its -o/--output parsing is unchanged. run.sh: the content heredoc now uses a random delimiter read from /dev/urandom. Newlines are stripped from the changelog and version outputs before they are written. The jq call is captured with '|| true', so a failing jq still doesn't stop the step, as with upstream's echo. dirname $OUTPUT is now quoted. One behaviour change: the split no longer expands '*' patterns in args, which upstream's script would have done.

