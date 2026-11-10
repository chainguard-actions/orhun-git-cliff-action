<!-- markdownlint-disable -->

# Hardening Report: orhun--git-cliff-action/v4.9.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **orhun--git-cliff-action/v4.9.0** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): GitHub Actions expressions are written directly into the shell command in the 'Run git-cliff' step: `run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}`. `inputs.config` and `inputs.args` are pasted into the script before bash parses it, so shell metacharacters in them (`;`, `$(...)`, backticks) run as commands. Fix: pass them through env vars and quote the expansions.

Locations:

- `action.yml:43`

### github-env-injection (severity: high)

run.sh runs from the 'Run git-cliff' step and writes untrusted values to $GITHUB_OUTPUT. (1) It writes the multi-line changelog content between a fixed heredoc delimiter: `echo "content<<EOF" >>$GITHUB_OUTPUT; awk 1 "$OUTPUT" >>$GITHUB_OUTPUT; echo "EOF" >>$GITHUB_OUTPUT`. The changelog is built from commit messages, which an attacker can control. A line equal to `EOF` ends the block early, and every line after it becomes a new output. The delimiter must come from a secure random source such as /dev/urandom. (2) `echo "changelog=$OUTPUT" >>$GITHUB_OUTPUT` writes a path taken from inputs.args (-o/--output) or the inherited OUTPUT env var, with no `printf '%s' | tr -d '\n\r'` step. (3) `echo "version=$(jq -r '.[0].version' $CONTEXT)" >>$GITHUB_OUTPUT` writes a version string taken from repository tags, also without the newline strip.

Locations:

- `run.sh:75`
- `run.sh:82`
- `run.sh:85`
- `action.yml:43`

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
index 25b3b2e..865ad37 100644
--- a/action.yml
+++ b/action.yml
@@ -29,6 +29,23 @@ outputs:
 runs:
   using: "composite"
   steps:
     - name: Download git-cliff
       shell: bash
       run: ${GITHUB_ACTION_PATH}/install.sh
@@ -37,14 +54,23 @@ runs:
         RUNNER_ARCH: ${{ runner.arch }}
         VERSION: ${{ inputs.version }}
         GITHUB_API_TOKEN: ${{ inputs.github_token || env.GITHUB_TOKEN || github.token }}
-
     - name: Run git-cliff
       id: run-git-cliff
       shell: bash
-      run: ${GITHUB_ACTION_PATH}/run.sh --config=${{ inputs.config }} ${{ inputs.args }}
       env:
         GITHUB_TOKEN: ${{ inputs.github_token || env.GITHUB_TOKEN || github.token }}
-
+        INPUT_CONFIG: ${{ inputs.config }}
+        INPUT_ARGS: ${{ inputs.args }}
+      run: |
+        args=()
+        if [ -n "$INPUT_CONFIG" ]; then
+          args+=(--config="$INPUT_CONFIG")
+        fi
+        if [ -n "$INPUT_ARGS" ]; then
+          while IFS= read -r -d '' t; do args+=("$t"); done \
+            < <(printf '%s' "$INPUT_ARGS" | xargs printf '%s\0')
+        fi
+        "${GITHUB_ACTION_PATH}/run.sh" "${args[@]}"
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
index a68a609..9981792 100755
--- a/run.sh
+++ b/run.sh
@@ -33,7 +33,7 @@ chown -R "$(id -u)" .
 
 # Create the output directory
 OUTPUT=${OUTPUT:="git-cliff/CHANGELOG.md"}
-mkdir -p "$(dirname $OUTPUT)"
+mkdir -p "$(dirname "$OUTPUT")"
 
 args=()
 take_next=
@@ -78,7 +78,8 @@ if [ "$FILESIZE" -le "$MAXSIZE" ]; then
 fi
 
 # Set output file
-echo "changelog=$OUTPUT" >>$GITHUB_OUTPUT
+safe_output=$(printf '%s' "$OUTPUT" | tr -d '\n\r')
+echo "changelog=$safe_output" >> "$GITHUB_OUTPUT"
 
 # Set the version output to the version of the latest release
 echo "version=$(jq -r '.[0].version' $CONTEXT)" >>$GITHUB_OUTPUT

```

Locations:

- `run.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, static-inline-injection, github-env-injection, prior-fix

**Notes:**

Nothing was run or tested; these are code edits only.

action.yml: `inputs.config` and `inputs.args` now reach the step through the env vars INPUT_CONFIG and INPUT_ARGS instead of being pasted into the shell command. Upstream pasted args straight into the script, so the shell honoured quotes inside it. To keep that behaviour I split INPUT_ARGS with the quote-aware xargs pattern: it fails on an unmatched quote, skips an empty value, and uses the shell's built-in printf. The config value is passed as one quoted argument, `--config="$INPUT_CONFIG"`, and I pass it even when it is empty because upstream always did.

run.sh:
- The changelog content block now uses a random heredoc delimiter from /dev/urandom instead of the fixed `EOF`. Its lines are written unchanged, so multi-line content still works.
- Newlines are stripped from the `changelog` and `version` outputs before they are written.
- The `version` command's failure is ignored (`|| true`). That matches upstream, which had no errexit, so a failed jq never stopped the script.
- `$OUTPUT`, `$CONTEXT` and `$GITHUB_OUTPUT` are now quoted, which re-applies the previous harden's `dirname` quoting fix.

