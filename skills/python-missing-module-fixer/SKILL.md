---
name: python-missing-module-fixer
description: Use when a traceback shows ModuleNotFoundError / ImportError “No module named …”. Map the missing import to the right package and rerun via uv (prefer no global installs).
---

When you see:
- ModuleNotFoundError: No module named 'X'
- ImportError: No module named X

1) Extract X exactly.

2) Map X -> package:
- yaml -> pyyaml
- dateutil -> python-dateutil
- PIL -> pillow
- cv2 -> opencv-python
- Crypto -> pycryptodome
- sklearn -> scikit-learn
- bs4 -> beautifulsoup4
- dotenv -> python-dotenv
- jwt -> PyJWT

3) Preferred fix (no global install): rerun the exact failing command under uv with an ephemeral dependency:
- uv run --with <package> -- <original command...>

Example for yaml:
- uv run --with pyyaml -- python <your_script.py ...>

4) If this is happening inside Codex skill-creator validation (quick_validate.py), rerun validation the same way:
- uv run --with pyyaml -- python scripts/quick_validate.py <path/to/skill>

5) If a persistent project fix is required instead:
- uv add <package>
- uv run -- <original command...>


Why this directly addresses your case:

The Codex skill-creator workflow explicitly includes running quick_validate.py, and there’s a known failure mode when PyYAML isn’t installed.

uv run --with … -- <cmd> is the supported way to inject per-invocation dependencies without touching global Python.

yaml comes from PyYAML.

If you want this skill to also patch the single failing callsite (skill-creator invoking python scripts/quick_validate.py ...), that’s a separate (small) rewriter—but it should target that one pattern only, not “skill-wide uv migration.”
