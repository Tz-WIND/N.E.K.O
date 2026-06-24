# AGENTS.md

## Cursor Cloud specific instructions

This section captures durable, non-obvious setup/run notes for Project N.E.K.O. (a local desktop AI companion). Standard contributor docs live in `CONTRIBUTING.md` and `README.MD`; the points below are the gotchas that those docs don't make obvious.

### Toolchain
- Python **3.11** is required (the repo pins `requires-python == 3.11.*`). The system Python is 3.12, so always go through `uv`, which manages an isolated 3.11 interpreter.
- Dependencies are managed by **uv** (`uv.lock`). The startup update script runs `uv sync` (main + `dev` group). The `galgame` dependency group (rapidocr/opencv, ~130 MB) is intentionally NOT installed; affected plugin code import-guards it.
- `uv` is installed at `~/.local/bin/uv` (already on `PATH` in interactive shells via `~/.bashrc`). Run everything via `uv run ...` so the `.venv` 3.11 interpreter is used.

### Services (run with `uv run python app/<server>.py`)
- **Memory server** — port `48912`. Start this **before** the main server.
- **Main server** — port `48911`, serves the web UI at `http://127.0.0.1:48911`. This is the entry point for manual testing.
- **Agent/Tool server** — port `48915`, optional; only needed for agent/browser tools and requires `uv run playwright install chromium` first.
- No external database/cache is needed for the core app (it uses local files + SQLite). `launcher.py` is the packaged orchestrator; for dev, start the individual servers above.

### First-run web onboarding (non-obvious)
- On a fresh config the backend boots in **limited-mode** and logs `selection_required`; it stays blocked until the **web UI** completes the storage-location prompt. Open the UI and click "Recommended Storage Location" to release it — there is no CLI equivalent.
- After storage selection the UI shows a skippable tutorial and a skippable personality picker.
- Persisted config/state lives under `~/.local/share/N.E.K.O/` (so onboarding only needs to be done once per VM).

### Provider config / chatting end-to-end
- A zero-config **"Free Version"** provider (hosted at `lanlan.tech`) can be selected for both **Core API** and **Assist API** on the in-app "API Keys" page — no key required, but it needs outbound internet. Enable the "Disable TTS" toggle to avoid voice/audio dependencies when only text chat matters.
- For real providers, set keys in the UI or via env vars (`NEKO_CORE_API` / `NEKO_CORE_API_KEY`).

### Frontend build (needed for full UI + some tests)
- Build with `./build_frontend.sh` (uses `npm ci` + Vite for `frontend/plugin-manager` and `frontend/react-neko-chat`; also unpacks the `yui-origin` Live2D model). Outputs (`static/react/neko-chat`, `frontend/*/dist`, `static/yui-origin`) are git-ignored, so rebuild after pulling frontend changes. Several `tests/unit/*_static.py` contract tests read these built bundles.

### Lint / test
- Lint = `uv run ruff check .` **plus** the custom convention checkers in `.github/workflows/analyze.yml` (`python scripts/check_*.py`, e.g. `check_async_blocking.py`, `check_module_layering.py`). Don't assume ruff alone reflects CI.
- Tests: run by directory, e.g. `uv run pytest tests/unit -q` and `uv run pytest plugin/tests -q`. Bare `pytest` from the repo root hits a cross-plugin collection name clash (duplicate test module name under `plugin/plugins/.../tests`), so always scope to a directory like CI does.
- Heads-up: on a clean `main`, a set of `tests/unit` source-contract / data-ordering tests already fail (e.g. `test_*_static.py`, `test_memory_temporal.py`); these are pre-existing and unrelated to environment setup.
