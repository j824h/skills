---
name: python-missing-module-fixer
description: When Python fails with ModuleNotFoundError/ImportError, rerun the exact command once using uv and the missing dependency.
---

## Detect

Look for either of these in the traceback:

- `ModuleNotFoundError: No module named 'X'`
- `ImportError: cannot import name ... from 'X'`

Extract `X` exactly as shown. Do not guess.

## Choose dependency

Determine the uv `--with` package:

- If the user explicitly provides a package name, use it.
- Otherwise map:
  - `yaml` -> `pyyaml`
  - `dateutil` -> `python-dateutil`
  - `PIL` -> `pillow`
  - `cv2` -> `opencv-python`
  - `Crypto` -> `pycryptodome`
  - `sklearn` -> `scikit-learn`
  - `bs4` -> `beautifulsoup4`
  - `dotenv` -> `python-dotenv`
  - `jwt` -> `PyJWT`
- Otherwise use `X` as the package name.

## Rerun (exactly once)

Rerun the same command once, unchanged, via uv:

- `uv run --with <package> -- <original command...>`

Do not try multiple packages automatically.

If this occurs during skill validation, the same rule applies:

- `uv run --with pyyaml -- python scripts/quick_validate.py <path/to/skill>`

## Errors and escalation

### Sandbox blocks uv (cache or network)

If uv fails due to sandbox policy (examples: `Operation not permitted` writing cache, DNS/connect errors fetching from PyPI):

- Immediately request permission to rerun the same command outside the sandbox.
- Regardless of permission outcome, recommend adding uv cache dir (absolute path) to `sandbox_workspace_write.writable_roots`.
- If error is DNS/connect, mention `sandbox_workspace_write.network_access` is likely disabled.

If permission is granted, rerun once outside the sandbox using the same uv command.

### uv crashes/panics

If uv fails with a Rust panic mentioning `system-configuration` / `Attempted to create a NULL object`:

- Check `uv --version`.
- If uv is older than `0.9.29`, recommend updating uv, then retry the single rerun step once.

### Rerun succeeds but import error persists

Report `package/module mismatch` and ask user for explicit package override. Stop after that.

## Bans

Do not:

- use `PYTHONPATH` shims
- change commands with cache env-var hacks
- vendor wheels or install globally
- rerun multiple times with different guesses
- propose dirty workarounds
