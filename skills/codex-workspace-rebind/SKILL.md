---
name: codex-workspace-rebind
description: Rebind Codex local state from an old absolute workspace path to a new path after a project root move, with backup and verification.
---

# Codex workspace rebind

Repair stale Codex workspace bindings after a workspace folder is moved/renamed.

## Use when
- Codex shows a missing working directory for a known project.
- A project was moved from one absolute path to another.
- Threads/sessions still resolve to the old workspace root.

## Inputs
- `OLD_PATH`: old absolute workspace path
- `NEW_PATH`: new absolute workspace path

## Targets
- `~/.codex/.codex-global-state.json`
- Every `~/.codex/sessions/**/*.jsonl` file containing `OLD_PATH`

## Safety rules
- Do not run destructive git commands.
- Do not delete session files.
- Do not edit SQLite for this operation.
- If replacement scope is ambiguous, stop and confirm with the user.

---

## Procedure

### 1) Preflight
- Require absolute paths for `OLD_PATH` and `NEW_PATH`.
- Check filesystem state with `test -e`.
- If `OLD_PATH` exists and `NEW_PATH` does not, run `mv "$OLD_PATH" "$NEW_PATH"`.
- Fail if `NEW_PATH` does not exist after preflight.

### 2) Discover patch set
- Always include `~/.codex/.codex-global-state.json` as a target (if present).
- Find session files under `~/.codex/sessions` that contain the literal `OLD_PATH`.

### 3) Backup (mandatory)
- Copy each target to `/tmp` with a timestamped suffix.
- Encode source path in backup filename to avoid collisions.
- Print all backup locations.

### 4) Patch (minimal edits)
- Replace the exact `OLD_PATH` string with `NEW_PATH` in every target file.
- Do not modify unrelated fields.

### 5) Verify (must pass)
- Confirm no `OLD_PATH` remains in target files.
- Validate JSON parse for global state file.
- Validate JSONL parse for each session file.
- Confirm global keys include `NEW_PATH`:
  - `electron-saved-workspace-roots`
  - `active-workspace-roots`
  - `electron-workspace-root-labels`
- Confirm session `session_meta.payload.cwd` references `NEW_PATH`.

### 6) User follow-up
- Ask the user to restart Codex.
- Re-open the project/thread and confirm no stale cwd warning.

---

## Quick commands (reference implementation)

```bash
set -euo pipefail

: "${OLD_PATH:?Set OLD_PATH}"
: "${NEW_PATH:?Set NEW_PATH}"

case "$OLD_PATH" in /*) ;; *) echo "OLD_PATH must be absolute"; exit 1;; esac
case "$NEW_PATH" in /*) ;; *) echo "NEW_PATH must be absolute"; exit 1;; esac

# Preflight move (only if OLD exists and NEW doesn't)
if test -e "$OLD_PATH" && ! test -e "$NEW_PATH"; then
  mv "$OLD_PATH" "$NEW_PATH"
fi

if ! test -e "$NEW_PATH"; then
  echo "NEW_PATH does not exist after preflight: $NEW_PATH"
  exit 1
fi

GLOBAL="$HOME/.codex/.codex-global-state.json"
mapfile -t SESSION_FILES < <(rg -l --hidden --no-ignore -S "$OLD_PATH" "$HOME/.codex/sessions" || true)

TARGETS=()
test -f "$GLOBAL" && TARGETS+=("$GLOBAL")
((${#SESSION_FILES[@]})) && TARGETS+=("${SESSION_FILES[@]}")

if ((${#TARGETS[@]} == 0)); then
  echo "No targets found (no global state file and no session files matched)."
  exit 0
fi

TS="$(date +%Y%m%d-%H%M%S)"

# Backup
for f in "${TARGETS[@]}"; do
  test -f "$f" || continue
  safe_name="$(printf '%s' "$f" | sed 's|/|__|g')"
  backup="/tmp/${safe_name}.bak.${TS}"
  cp "$f" "$backup"
  echo "backup: $f -> $backup"
done

# Patch (exact literal replacement)
for f in "${TARGETS[@]}"; do
  test -f "$f" || continue
  OLD_PATH="$OLD_PATH" NEW_PATH="$NEW_PATH" \
    perl -0pi -e 's/\Q$ENV{OLD_PATH}\E/$ENV{NEW_PATH}/g' "$f"
done

# Verify: no OLD_PATH remains
if rg -n --hidden --no-ignore -S "$OLD_PATH" "${TARGETS[@]}"; then
  echo "old path still present after patch"
  exit 1
fi

# Verify: parse
test -f "$GLOBAL" && jq empty "$GLOBAL" >/dev/null
for f in "${SESSION_FILES[@]}"; do
  jq -c . "$f" >/dev/null
done

# Verify: global keys include NEW_PATH (required if GLOBAL exists)
if test -f "$GLOBAL"; then
  jq -e --arg p "$NEW_PATH" '."electron-saved-workspace-roots" | index($p)' "$GLOBAL" >/dev/null
  jq -e --arg p "$NEW_PATH" '."active-workspace-roots" | index($p)' "$GLOBAL" >/dev/null
  jq -e --arg p "$NEW_PATH" '."electron-workspace-root-labels"[$p]' "$GLOBAL" >/dev/null
fi

# Verify: session_meta.payload.cwd references NEW_PATH (required if any session file matched)
for f in "${SESSION_FILES[@]}"; do
  jq -e --arg p "$NEW_PATH" 'select(.type=="session_meta" and .payload.cwd==$p)' "$f" >/dev/null
done

```
