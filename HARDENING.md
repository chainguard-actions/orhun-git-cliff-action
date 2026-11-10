<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.9.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.9.1** was hardened automatically. 7 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): the 'Run git-cliff' step puts GitHub expressions straight into a `run:` command: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. The values of inputs.config and inputs.args are pasted into the script before the shell runs it, so characters like `;`, `$(...)` or backticks in them run as commands. Fix: pass them through env (e.g. CONFIG: ${{ inputs.config }}, ARGS: ${{ inputs.args }}) and use "--config=$CONFIG". For ARGS, expand it into an array with a shellcheck SC2086/SC2206 disable comment.

Locations:

- `action.yml:43`

### script-injection (severity: high)

Rule (b): run.sh expands the inherited, workflow-controlled OUTPUT variable without quotes: `mkdir -p "$(dirname $OUTPUT)"`. Quote it: `dirname "$OUTPUT"`.

Locations:

- `run.sh:36`

### github-env-injection (severity: high)

run.sh, which is the 'Run git-cliff' step's script, writes to $GITHUB_OUTPUT unsafely. (f) It writes the changelog content between a fixed heredoc delimiter: `echo "content<<EOF" >>$GITHUB_OUTPUT; awk 1 "$OUTPUT" >>$GITHUB_OUTPUT; echo "EOF" >>$GITHUB_OUTPUT`. The changelog comes from commit messages and inputs.args, so a line equal to 'EOF' ends the block early and injects extra outputs. (c/e) It also writes `echo "changelog=$OUTPUT" >>$GITHUB_OUTPUT` (line 81) and `echo "version=$(jq -r '.[0].version' $CONTEXT)" >>$GITHUB_OUTPUT` (line 84) without the `printf '%s' ... | tr -d '\n\r'` step. OUTPUT comes from inherited env or the user's args, and the version comes from repository tags. Fix: build the delimiter from /dev/urandom and strip newlines from single-line values.

Locations:

- `run.sh:74`
- `run.sh:81`
- `run.sh:84`

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
index a8c717c..b3563ac 100644
--- a/action.yml
+++ b/action.yml
@@ -29,6 +29,23 @@ outputs:
 runs:
   using: "composite"
   steps:
     - name: Download git-cliff
       shell: bash
       run: ${GITHUB_ACTION_PATH}/install.sh
@@ -37,14 +54,20 @@ runs:
         RUNNER_ARCH: ${{ runner.arch }}
         VERSION: ${{ inputs.version }}
         GITHUB_API_TOKEN: ${{ inputs.github_token || env.GITHUB_TOKEN || github.token }}
-
     - name: Run git-cliff
       id: run-git-cliff
       shell: bash
-      run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}
+      run: |
+        args=()
+        if [ -n "$INPUT_ARGS" ]; then
+          while IFS= read -r -d '' t; do args+=("$t"); done \
+            < <(printf '%s' "$INPUT_ARGS" | xargs printf '%s\0')
+        fi
+        "${GITHUB_ACTION_PATH}/run.sh" "--config=${INPUT_CONFIG}" "${args[@]}"
       env:
         GITHUB_TOKEN: ${{ inputs.github_token || env.GITHUB_TOKEN || github.token }}
-
+        INPUT_CONFIG: ${{ inputs.config }}
+        INPUT_ARGS: ${{ inputs.args }}
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
index a68a609..5b06d76 100755
--- a/run.sh
+++ b/run.sh
@@ -33,7 +33,7 @@ chown -R "$(id -u)" .
 
 # Create the output directory
 OUTPUT=${OUTPUT:="git-cliff/CHANGELOG.md"}
-mkdir -p "$(dirname $OUTPUT)"
+mkdir -p "$(dirname "$OUTPUT")"
 
 args=()
 take_next=
@@ -78,10 +78,12 @@ if [ "$FILESIZE" -le "$MAXSIZE" ]; then
 fi
 
 # Set output file
-echo "changelog=$OUTPUT" >>$GITHUB_OUTPUT
+safe_output=$(printf '%s' "$OUTPUT" | tr -d '\n\r')
+echo "changelog=$safe_output" >>"$GITHUB_OUTPUT"
 
 # Set the version output to the version of the latest release
-echo "version=$(jq -r '.[0].version' $CONTEXT)" >>$GITHUB_OUTPUT
+safe_version=$(jq -r '.[0].version' "$CONTEXT" | tr -d '\n\r')
+echo "version=$safe_version" >>"$GITHUB_OUTPUT"
 
 # Pass exit code to the next step
 echo "exit_code=$exit_code" >>$GITHUB_OUTPUT

```

Locations:

- `run.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, static-inline-injection, prior-fix

**Notes:**

action.yml: inputs.config and inputs.args now reach the 'Run git-cliff' step through env (INPUT_CONFIG and INPUT_ARGS) instead of being pasted into the script. The config value is passed as one quoted argument, "--config=$INPUT_CONFIG". The args value is split into an array with the quote-aware xargs method, since upstream pasted it in as an expression. The split has an empty-value guard, a check that fails on an unmatched quote, and uses the shell's built-in printf through sh -c. run.sh: $OUTPUT is now quoted when passed to dirname. The changelog content is written between a random delimiter drawn from /dev/urandom, not a fixed 'EOF'. Newlines are stripped from the changelog and version outputs before they are written. The jq call is captured with '|| true', which keeps upstream's behaviour: there, a failing jq inside echo did not stop the step. These changes also re-apply both prior-fix patches.

