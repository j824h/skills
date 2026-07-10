---
name: codex-workspace-rebind
description: Rebind Codex global state and saved thread metadata after a project directory moves or is renamed. Use when Codex reports a missing working directory, a known project moves to a new absolute path, or existing threads still resolve to an old workspace root.
---

# Codex Workspace Rebind

Repair stale Codex path bindings after moving a workspace. Keep the mechanism visible so the replacement scope can be reviewed before execution.

## Safety Contract

- Obtain absolute `OLD_PATH` and `NEW_PATH` values.
- Run from a workspace outside `OLD_PATH`.
- Close every thread rooted at `OLD_PATH`; never rewrite an open session.
- Select sessions only when parsed `session_meta.payload.cwd` exactly equals `OLD_PATH`. Ignore diagnostic text and command history when choosing targets.
- Update only global workspace keys that already contain `OLD_PATH`. Never register `NEW_PATH` in an unaffected key.
- Treat paths as fixed strings, preserve unrelated state, and stop on malformed or ambiguous input.
- Do not edit SQLite, delete sessions, or run destructive Git commands.

## Procedure

1. Set `OLD_PATH` and `NEW_PATH` to absolute paths.
2. Review and run the complete Bash block below. Do not run it as zsh.
3. Keep the printed `/tmp` backups until verification succeeds.
4. Restart Codex, reopen the project/thread, and confirm no stale-cwd warning appears.

```bash
set -euo pipefail

: "${OLD_PATH:?Set OLD_PATH}"
: "${NEW_PATH:?Set NEW_PATH}"

case "$OLD_PATH" in /*) ;; *) echo "OLD_PATH must be absolute" >&2; exit 2;; esac
case "$NEW_PATH" in /*) ;; *) echo "NEW_PATH must be absolute" >&2; exit 2;; esac
test "$OLD_PATH" != "$NEW_PATH" || { echo "paths must differ" >&2; exit 2; }

case "$PWD" in
  "$OLD_PATH"|"$OLD_PATH"/*)
    echo "run from outside OLD_PATH" >&2
    exit 2
    ;;
esac

for required in jq perl rg; do
  command -v "$required" >/dev/null 2>&1 || {
    echo "missing required command: $required" >&2
    exit 2
  }
done

NEEDS_MOVE=false
if test -e "$OLD_PATH" && test -e "$NEW_PATH"; then
  echo "both paths exist; refusing ambiguous move" >&2
  exit 1
elif test -e "$OLD_PATH"; then
  NEEDS_MOVE=true
elif ! test -e "$NEW_PATH"; then
  echo "neither path exists" >&2
  exit 1
fi

GLOBAL="$HOME/.codex/.codex-global-state.json"
SESSIONS_DIR="$HOME/.codex/sessions"
SESSION_FILES=()

if test -d "$SESSIONS_DIR"; then
  while IFS= read -r -d '' candidate; do
    jq -c . "$candidate" >/dev/null || {
      echo "invalid JSONL candidate: $candidate" >&2
      exit 1
    }
    if jq -e --arg old "$OLD_PATH" \
      'select(.type == "session_meta" and .payload.cwd == $old)' \
      "$candidate" >/dev/null; then
      SESSION_FILES+=("$candidate")
    fi
  done < <(rg -l -0 --hidden --no-ignore -F -- "$OLD_PATH" "$SESSIONS_DIR" || true)
fi

GLOBAL_AFFECTED=false
GLOBAL_HAD_SAVED=false
GLOBAL_HAD_ACTIVE=false
GLOBAL_HAD_LABEL=false

if test -f "$GLOBAL"; then
  jq empty "$GLOBAL" >/dev/null
  GLOBAL_HAD_SAVED=$(jq -r --arg old "$OLD_PATH" \
    '((.["electron-saved-workspace-roots"] // []) | index($old)) != null' "$GLOBAL")
  GLOBAL_HAD_ACTIVE=$(jq -r --arg old "$OLD_PATH" \
    '((.["active-workspace-roots"] // []) | index($old)) != null' "$GLOBAL")
  GLOBAL_HAD_LABEL=$(jq -r --arg old "$OLD_PATH" \
    '((.["electron-workspace-root-labels"] // {}) | has($old))' "$GLOBAL")

  if test "$GLOBAL_HAD_LABEL" = true && jq -e \
    --arg old "$OLD_PATH" --arg new "$NEW_PATH" '
      (.["electron-workspace-root-labels"] // {}) as $labels |
      ($labels | has($new)) and ($labels[$old] != $labels[$new])
    ' "$GLOBAL" >/dev/null; then
    echo "conflicting labels exist for old and new paths" >&2
    exit 1
  fi

  if test "$GLOBAL_HAD_SAVED" = true || \
     test "$GLOBAL_HAD_ACTIVE" = true || \
     test "$GLOBAL_HAD_LABEL" = true; then
    GLOBAL_AFFECTED=true
  fi
fi

if command -v lsof >/dev/null 2>&1; then
  for ((i = 0; i < ${#SESSION_FILES[@]}; i++)); do
    f=${SESSION_FILES[$i]}
    if lsof -t "$f" >/dev/null 2>&1; then
      echo "close open target session: $f" >&2
      exit 1
    fi
  done
fi

TARGETS=()
test "$GLOBAL_AFFECTED" = false || TARGETS+=("$GLOBAL")
for ((i = 0; i < ${#SESSION_FILES[@]}; i++)); do
  TARGETS+=("${SESSION_FILES[$i]}")
done

if test "${#TARGETS[@]}" -eq 0; then
  test "$NEEDS_MOVE" = false || mv "$OLD_PATH" "$NEW_PATH"
  test ! -e "$OLD_PATH"
  test -e "$NEW_PATH"
  echo "filesystem ready; no stale Codex bindings found"
  exit 0
fi

TS="$(date +%Y%m%d-%H%M%S)-$$"
for f in "${TARGETS[@]}"; do
  safe_name=$(printf '%s' "$f" | sed 's|/|__|g')
  backup="/tmp/${safe_name}.bak.${TS}"
  cp -p "$f" "$backup"
  echo "backup: $f -> $backup"
done

test "$NEEDS_MOVE" = false || mv "$OLD_PATH" "$NEW_PATH"

for ((i = 0; i < ${#SESSION_FILES[@]}; i++)); do
  f=${SESSION_FILES[$i]}
  OLD_PATH="$OLD_PATH" NEW_PATH="$NEW_PATH" \
    perl -0pi -e 's/\Q$ENV{OLD_PATH}\E/$ENV{NEW_PATH}/g' "$f"
done

if test "$GLOBAL_AFFECTED" = true; then
  TMP=$(mktemp "${GLOBAL}.tmp.XXXXXX")
  trap 'rm -f "$TMP"' EXIT
  jq --arg old "$OLD_PATH" --arg new "$NEW_PATH" '
    def rebind_array:
      if index($old) == null then .
      elif index($new) != null then map(select(. != $old))
      else map(if . == $old then $new else . end)
      end;
    if ((.["electron-saved-workspace-roots"] // []) | index($old)) != null
      then .["electron-saved-workspace-roots"] |= rebind_array else . end |
    if ((.["active-workspace-roots"] // []) | index($old)) != null
      then .["active-workspace-roots"] |= rebind_array else . end |
    if ((.["electron-workspace-root-labels"] // {}) | has($old)) then
      .["electron-workspace-root-labels"] as $labels |
      if ($labels | has($new))
        then .["electron-workspace-root-labels"] = ($labels | del(.[$old]))
        else .["electron-workspace-root-labels"] =
          (($labels + {($new): $labels[$old]}) | del(.[$old]))
      end
    else . end
  ' "$GLOBAL" > "$TMP"
  mv "$TMP" "$GLOBAL"
  trap - EXIT
fi

if test "$GLOBAL_AFFECTED" = true; then
  jq empty "$GLOBAL" >/dev/null
  test "$GLOBAL_HAD_SAVED" = false || jq -e --arg new "$NEW_PATH" \
    '.["electron-saved-workspace-roots"] | index($new)' "$GLOBAL" >/dev/null
  test "$GLOBAL_HAD_ACTIVE" = false || jq -e --arg new "$NEW_PATH" \
    '.["active-workspace-roots"] | index($new)' "$GLOBAL" >/dev/null
  test "$GLOBAL_HAD_LABEL" = false || jq -e --arg new "$NEW_PATH" \
    '.["electron-workspace-root-labels"] | has($new)' "$GLOBAL" >/dev/null
fi

for ((i = 0; i < ${#SESSION_FILES[@]}; i++)); do
  f=${SESSION_FILES[$i]}
  jq -c . "$f" >/dev/null
  ! rg -q -F -- "$OLD_PATH" "$f" || {
    echo "old path remains in target session: $f" >&2
    exit 1
  }
  jq -e --arg new "$NEW_PATH" \
    'select(.type == "session_meta" and .payload.cwd == $new)' \
    "$f" >/dev/null
done

test ! -e "$OLD_PATH"
test -e "$NEW_PATH"
echo "workspace rebind completed and verified"
```
