# Hermes Desktop

Native Electron shell for Hermes, with installer targets for macOS, Windows, and Linux. The packaged app ships only the Electron shell — the Hermes Agent Python runtime is fetched and installed at first launch (see [Runtime Bootstrap](#runtime-bootstrap)), so a desktop-only user ends up with the same on-disk layout as a CLI install.

## Architecture

```
Electron (electron/main.cjs)
 ├─ Vite renderer (src/, React 19)          → the desktop UI
 └─ spawns: hermes dashboard --no-open --tui → the Python backend
            on an open port in 9120–9199
```

The renderer talks to the backend over the same gateway APIs the dashboard/TUI use, and reuses the embedded TUI rather than reimplementing chat in React. Backend resolution, first-launch install, and self-update all live in `electron/main.cjs`.

## Prerequisites

- **Python 3.11+** — agent runtime, backend, and tool execution. Required.
- **Git for Windows** (Windows only) — provides Git Bash, which the `terminal` tool calls directly. macOS/Linux already ship bash. Required.
- **ripgrep** — fast `.gitignore`-aware search for `search_files`. Recommended everywhere; Hermes falls back to slower `grep`/`find` if absent.

The packaged Windows installer (`Hermes-*.exe`) detects all three and offers `winget` installs; the `.msi` skips the prereq page (enterprise deploys handle prereqs out of band). For `npm run dev`, the Electron bootstrapper checks Python and Git Bash at first launch and errors clearly if either is missing. Manual installs:

```powershell
winget install -e --id Python.Python.3.11 --scope user
winget install -e --id Git.Git
winget install -e --id BurntSushi.ripgrep.MSVC --scope user
```

## Development

Install workspace deps from the repo root once (keeps `apps/desktop`, `web`, and `apps/shared` linked):

```bash
npm install
```

Then, from this directory:

```bash
npm run dev          # Vite on 127.0.0.1:5174 + Electron, which boots the backend
npm run dev:fake-boot # deterministic per-phase boot delays to exercise the startup overlay
```

Python is provisioned automatically on first launch (a venv at `HERMES_HOME/hermes-agent/venv` + `pip install -e .`) if you don't already have a CLI install to reuse.

Useful overrides:

```bash
HERMES_DESKTOP_HERMES_ROOT=/path/to/clone npm run dev  # run a specific source checkout
HERMES_DESKTOP_PYTHON=/path/to/python npm run dev
HERMES_DESKTOP_CWD=/path/to/project npm run dev
HERMES_DESKTOP_IGNORE_EXISTING=1 npm run dev           # ignore any `hermes` on PATH (test bootstrap)
HERMES_HOME=/tmp/throwaway npm run dev                 # sandbox config/keys away from the real ~/.hermes
HERMES_DESKTOP_BOOT_FAKE=1 npm run dev                 # fake boot phases (dev:fake-boot wraps this)
```

`HERMES_HOME` defaults to `%LOCALAPPDATA%\hermes` on Windows and `~/.hermes` elsewhere.

On a fresh profile, Desktop shows a first-run setup overlay after boot — it saves the minimum provider credential (e.g. `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`) to the active `.env`, reloads the backend env, and continues without a manual trip to Settings.

## Dashboard Dev

The dashboard (`web/`) is served by the backend; for HMR run the backend and Vite separately:

```bash
hermes dashboard --tui --no-open   # backend on 127.0.0.1:9119
cd web && npm run dev              # Vite; proxies /api, /api/pty, plugin assets to :9119
```

## Build

```bash
npm run build         # write-build-stamp + stage native deps + tsc -b + vite build
npm run pack          # unpacked app under release/ (electron-builder --dir)
npm run dist:mac      # macOS DMG + zip
npm run dist:win      # Windows NSIS + MSI   (also dist:win:nsis / dist:win:msi)
npm run dist:linux    # Linux AppImage + deb + rpm
```

`npm run build` runs `scripts/write-build-stamp.cjs`, which records the git commit + branch into `build/install-stamp.json`. That file ships in the package (`extraResources`) and pins the ref the first-launch bootstrap installs against. Builds fail loudly if no git ref can be resolved.

## Releases

Installers are built and uploaded manually (no CI auto-build). Build per platform with the `dist:*` scripts above, attach the artifacts to a GitHub Release, and point the website's download links at them via `NEXT_PUBLIC_HERMES_DL_*` env.

Signing/notarization credentials are read from the environment at build time when present:

- macOS: `CSC_LINK`, `CSC_KEY_PASSWORD`, `APPLE_API_KEY`, `APPLE_API_KEY_ID`, `APPLE_API_ISSUER`
- Windows: `WIN_CSC_LINK`, `WIN_CSC_KEY_PASSWORD`

## Runtime Bootstrap

Desktop shares its install layout with the CLI installers (`scripts/install.ps1`, `scripts/install.sh`), so a desktop-only and a CLI-only user end up with the same files in the same places.

### Where things live

```text
HERMES_HOME/                       # %LOCALAPPDATA%\hermes (Windows) | ~/.hermes (macOS/Linux)
├── hermes-agent/                  # ACTIVE_HERMES_ROOT — a git checkout
│   ├── .git/
│   ├── pyproject.toml             # dependency source of truth
│   ├── venv/                      # virtualenv (Scripts\ on Windows, bin/ elsewhere)
│   └── .hermes-bootstrap-complete # marker: first-launch install succeeded
├── git/                           # PortableGit (Windows; installed by install.ps1)
├── config.yaml
├── .env
└── logs/                          # desktop.log, agent.log, errors.log, gateway.log
```

The packaged installer ships only the Electron app; the agent is cloned + installed at first launch by running the platform installer (`install.ps1` on Windows, `install.sh` on macOS/Linux) against the ref baked into `install-stamp.json`.

### Backend resolution order

`resolveHermesBackend()` picks the first usable backend:

1. `HERMES_DESKTOP_HERMES_ROOT` — explicit dev override (no bootstrap).
2. Repo source root — only under `npm run dev` from a checkout, so local Python edits win.
3. `HERMES_HOME/hermes-agent` when `.hermes-bootstrap-complete` exists — the trusted canonical install.
4. An existing `hermes` on PATH (smoke-tested with `--version`; skipped when `HERMES_DESKTOP_IGNORE_EXISTING=1`).
5. A pip-installed `hermes_cli` via system Python (import-checked).
6. None of the above → `bootstrap-needed`; the first-launch wizard installs and writes the marker.

### First-launch flow (packaged install)

1. Resolution returns `bootstrap-needed`; the renderer shows the install overlay.
2. Main fetches the platform installer from GitHub at the pinned commit.
3. Main reads the stage list (`--manifest`) then runs each stage (`--stage <name> --json`), streaming progress to the renderer.
4. On success it writes `.hermes-bootstrap-complete` (`{ schemaVersion, pinnedCommit, pinnedBranch, completedAt, desktopVersion }`).
5. The renderer hands off to onboarding (key / model / persona). Subsequent launches see the marker and skip steps 1–4.

### Updates

Once bootstrapped the install is a real git checkout. `applyUpdates()` (or `hermes update` from the CLI) drives it:

- **Windows** hands off to the staged `Hermes-Setup.exe --update` (the desktop must exit to release the venv shim lock); the updater runs `hermes update` and rebuilds the GUI.
- **macOS/Linux** update in-process: `hermes update` + `hermes desktop --build-only`, then an atomic `.app` swap + relaunch on macOS.
- **CLI installs with no staged updater** get the exact `hermes update [--branch <x>]` command to run themselves.

Two transition guards (relevant while desktop tracks the `bb/gui` branch ahead of its `main` merge):

- **Self-healing branch.** If the tracked update branch no longer exists on origin (e.g. `bb/gui` merged + deleted), the updater falls back to `main` and persists it — a read-only `ls-remote` probe that only flips on a definitive "ref absent", never on a network blip.
- **Backend contract guard.** The backend reports `DESKTOP_BACKEND_CONTRACT` in session info; if it predates what this build needs (e.g. a `bb/gui` app pointed at a `main` checkout), the desktop warns with a one-click "Update Hermes" instead of failing cryptically. Bump the integer (both `tui_gateway/server.py` and `src/store/updates.ts`) when the contract changes.

## Testing Install Paths

```bash
npm run test:desktop:existing   # packaged app against an existing PATH `hermes`, real ~/.hermes
npm run test:desktop:fresh      # throwaway sandbox: IGNORE_EXISTING + temp userData + temp HERMES_HOME
npm run test:desktop:dmg        # build + open the DMG
npm run test:desktop:platforms  # bootstrap-path assertions (runtime selection, WSL guards, winpty/ptyprocess)
npm run test:desktop:all
```

Skip the rebuild on reruns with `HERMES_DESKTOP_SKIP_BUILD=1`.

## Icons

Icons live in `assets/` (`icon.icns`, `icon.ico`, `icon.png`); the builder points at `assets/icon`. Replace the files directly to change the app icon.

## Debugging

Boot logs: `HERMES_HOME/logs/desktop.log` (includes backend output + recent Python traceback). Check it first when the UI reports `Desktop boot failed`.

```bash
# Force a fresh first-launch bootstrap
rm "$HOME/.hermes/hermes-agent/.hermes-bootstrap-complete"          # macOS/Linux
Remove-Item "$env:LOCALAPPDATA\hermes\hermes-agent\.hermes-bootstrap-complete"  # Windows

# Rebuild a broken venv
rm -rf "$HOME/.hermes/hermes-agent/venv"                            # macOS/Linux
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\hermes\hermes-agent\venv"        # Windows

# Reset stale macOS mic permission prompts
tccutil reset Microphone com.github.Electron
tccutil reset Microphone com.nousresearch.hermes
```

## Verification

Run before handing off changes (lint may report pre-existing warnings but must exit without errors):

```bash
npm run fix
npm run type-check
npm run lint
npm run test:desktop:all
```
