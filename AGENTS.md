# AGENTS.md

For architecture, plugin contracts, formats, and conventions, read [CLAUDE.md](CLAUDE.md) — it is the primary developer/agent guide. This file only adds Cloud-agent-specific operational notes.

## Cursor Cloud specific instructions

FeedBack is a single deployable product: one FastAPI/uvicorn server (`server.py` + `main.py`) that serves the vanilla-JS SPA in `static/`, the shared libs in `lib/`, and the in-tree plugins in `plugins/`. There is no separate database, queue, or auth service (SQLite lives under `CONFIG_DIR`).

### Dependencies (installed by the startup update script)
- Python deps: `pip install --user -r requirements.txt -r requirements-test.txt` (Python 3.12).
- Node deps (for lint + JS tests + Tailwind build): `npm ci`.
- `pip --user` installs console scripts (`pytest`, `uvicorn`, …) into `~/.local/bin`, which is not on `PATH` by default. Invoke via the module form instead: `python3 -m pytest`, `python3 main.py`.

### Run the server (development)
No Docker needed for core work — run natively:
```
PYTHONPATH=lib:. DLC_DIR=<songs-dir> CONFIG_DIR=<config-dir> HOST=127.0.0.1 PORT=8000 python3 main.py
```
It listens on `http://127.0.0.1:8000`. On first scan it auto-seeds starter songs (Für Elise, Star-Spangled Banner, Ode to Joy, plus a diagnostic chart) into `DLC_DIR`, so the library is playable without supplying your own charts. The library API is `GET /api/library` (not `/api/songs`). Enrichment tries to reach musicbrainz.org and logs a harmless "network unavailable" line when egress is blocked — not an error.

### Lint / test / build
Commands are defined in `package.json` and `.github/workflows/ci.yml`:
- Python tests: `python3 -m pytest` (config in `pyproject.toml`, ~2800 tests).
- JS plugin-API tests: `node --test tests/js/*.test.js 'tests/plugins/*/js/*.test.js' 'plugins/*/tests/*.test.js'`.
- ESLint: `npm run lint` (max-lines warnings are non-blocking; only errors fail CI).
- Tailwind freshness gate: `bash scripts/build-tailwind.sh` must leave `static/tailwind.min.css` unchanged (`git diff --quiet`). Regenerate and commit it whenever you add Tailwind classes to core source.

### Browser playback caveat (important, non-obvious)
Playback position (and therefore the note-highway animation) is driven by the HTML5 `<audio>` element's `currentTime` (see `static/js/transport.js`). The Cloud VM has **no audio output device** (`/dev/snd` absent), and the interactive computer-use Chrome runs on an occluded Xvfb window, so it **throttles `requestAnimationFrame`** — in that browser the highway looks frozen and the top-right timer does not advance even though play/pause toggles. This is an environment limitation, **not an app bug**.

To verify playback actually works, drive a fresh Chrome via the DevTools Protocol where rAF is not throttled — e.g. launch `google-chrome --headless=new --remote-debugging-port=<port> --use-gl=angle --use-angle=swiftshader-webgl --enable-unsafe-swiftshader --autoplay-policy=no-user-gesture-required`, `Page.navigate` to the app, call `window.playSong('starter/beethoven-fur_elise.feedpak')`, wait for the `<audio>` `readyState>=2`, `await audio.play()`, and confirm `audio.currentTime` and `window.highway.getTime()` both advance (~1 s/s). `Page.captureScreenshot` frames from this instance show the 3D highway (WebGL2 via SwiftShader) animating. A PulseAudio null sink (`pulseaudio --start` + `module-null-sink` + `module-x11-publish`) can also be used, but is not required and is not part of the update script.
