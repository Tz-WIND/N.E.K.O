# AGENTS.md

## Cursor Cloud specific instructions

Project N.E.K.O. is a Python 3.11 AI-companion server (managed by [`uv`](https://docs.astral.sh/uv/)) plus two Vite frontends (`frontend/react-neko-chat` React, `frontend/plugin-manager` Vue). Standard dev/lint/test/build commands live in `README.MD` (源码开发 section), `CONTRIBUTING.md`, `pyproject.toml`, and `pytest.ini` — refer to those. The notes below are only the non-obvious caveats for running this in the cloud VM.

### Environment
- `uv` is preinstalled in the snapshot at `~/.local/bin/uv` and is on PATH in login shells (`~/.bashrc` sources `~/.local/bin/env`). The startup update script runs `uv sync`, which also provisions the pinned Python 3.11 toolchain. Always run Python via `uv run ...`.
- Node 22 is available and satisfies the `>=20.19` requirement.
- Built frontend assets (`static/react/neko-chat/`, `frontend/plugin-manager/dist/`) and `node_modules/` are git-ignored and persist in the VM snapshot, NOT in git. The update script does not rebuild them. If frontend source changes (or the snapshot is missing them), rebuild once with `./build_frontend.sh` (unpacks the yui-origin Live2D model, then `npm ci && npm run build` for both apps). The main server serves these assets, so the UI needs them present.

### Running the servers (caveat: entry points are packages, not files)
- The README shows `uv run python app/memory_server.py`, but `app/memory_server`, `app/main_server`, and `app/agent_server` are packages. Start them with the module form:
  - `uv run python -m app.memory_server` (memory, port 48912 — required)
  - `uv run python -m app.main_server` (web UI + API, port 48911 — required; open http://localhost:48911)
  - `uv run python -m app.agent_server` (tool/agent features, port 48915 — optional)
- Start `memory_server` before `main_server`. Ports come from `config/network.py` (override with `NEKO_MAIN_SERVER_PORT`, `NEKO_MEMORY_SERVER_PORT`, etc.).

### First-run storage gate
- On a fresh data dir both servers boot in "limited-mode" logging `selection_required` and wait for a one-time storage-location choice made in the web UI (home page onboarding). Complete that selection in the browser to leave limited-mode; until then some APIs return 409. App data lives under `~/Documents/N.E.K.O` and `~/.local/share/N.E.K.O`.

### API keys
- Real conversation needs an AI provider key configured at http://localhost:48911/api_key (Core API must support the Realtime API; a free tier works out-of-the-box for basic use). No external database is required — storage is local SQLite.

### Lint / test
- Lint (CI gate): `uv run ruff check .` (emits `# noqa` warnings but passes).
- Tests: `uv run pytest` (or scope with markers like `-m unit`). A handful of unit tests under `tests/unit/` fail on a clean checkout for reasons unrelated to environment setup (logic/assertion mismatches, e.g. `test_document_parser.py`, `test_music_crawlers.py`, `test_ai_aware_stage1_path_b.py`); treat these as pre-existing, not an env problem.
