---
name: codex-workspace-rebind
description: Repair stale Codex workspace and thread path bindings after a project folder rename by updating local Codex state files and verifying the new path is fully bound.
---

# Codex Workspace Rebind

Rebind Codex state from an old absolute workspace path to a new path after a folder rename.

## Workflow

1. Collect inputs.
- Collect old absolute path.
- Collect new absolute path.

2. Verify filesystem truth first.
- Check both paths with `test -e` or `ls -ld`.
- If `OLD_PATH` exists and `NEW_PATH` does not, run `mv "$OLD_PATH" "$NEW_PATH"`.
- Re-check both paths after `mv`.
- Stop and confirm with the user if `NEW_PATH` still does not exist.

3. Identify files to patch.
- Target `~/.codex/.codex-global-state.json`.
- Target session files in `~/.codex/sessions/**/*.jsonl` that contain the old path.

4. Back up before edits.
- Copy each target file to `/tmp` with a timestamped suffix.
- Preserve unique names in `/tmp` by encoding the full source path.
- Report backup paths.

5. Patch bindings.
- Replace the exact old absolute path string with the new absolute path string in selected files.
- Keep edits minimal and avoid unrelated key changes.

6. Verify patch.
- Confirm no old path remains in patched files via `rg`.
- Confirm global keys include the new path:
  - `electron-saved-workspace-roots`
  - `active-workspace-roots`
  - `electron-workspace-root-labels`
- Confirm session `session_meta.payload.cwd` uses the new path.
- Validate edited JSON/JSONL entries parse cleanly.

7. Finalize.
- Ask the user to restart Codex.
- Re-check for stale project entry and cwd warning.

## Safety Rules

- Do not run destructive git commands.
- Do not delete session files.
- Do not edit SQLite for this operation; workspace/thread binding is in JSON/JSONL state.
- Stop and ask for confirmation when replacement scope is ambiguous.

## Quick Commands

```bash
rg -n --hidden --no-ignore -S "$OLD_PATH" ~/.codex/.codex-global-state.json ~/.codex/sessions

if test -e "$OLD_PATH" && ! test -e "$NEW_PATH"; then
  mv "$OLD_PATH" "$NEW_PATH"
fi

TS="$(date +%Y%m%d-%H%M%S)"
FILES=(
  "$HOME/.codex/.codex-global-state.json"
)
while IFS= read -r f; do
  FILES+=("$f")
done < <(rg -l --hidden --no-ignore -S "$OLD_PATH" "$HOME/.codex/sessions")

for f in "${FILES[@]}"; do
  safe_name="$(printf '%s' "$f" | sed 's|/|__|g')"
  backup="/tmp/${safe_name}.bak.${TS}"
  cp "$f" "$backup"
  echo "$f -> $backup"
done

# Patch each selected file.
for f in "${FILES[@]}"; do
  perl -0pi -e 's|\Q'$OLD_PATH'\E|'"$NEW_PATH"'|g' "$f"
done

# Verify no old path remains.
rg -n --hidden --no-ignore -S "$OLD_PATH" "${FILES[@]}"

# Validate edited JSON and JSONL parse cleanly.
jq empty "$HOME/.codex/.codex-global-state.json"
for f in "${FILES[@]}"; do
  case "$f" in
    *.json) jq empty "$f" >/dev/null ;;
    *.jsonl) jq -c . "$f" >/dev/null ;;
  esac
done
```
