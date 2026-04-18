# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This is a meta-repository containing two git submodules:

- `reachy_mini/` — Python SDK, FastAPI daemon, kinematics, media
- `reachy-mini-desktop-app/` — Tauri (Rust) + React desktop application

For app development guidance (creating/running Reachy Mini apps), read `reachy_mini/AGENTS.md`.

---

## Python SDK (`reachy_mini/`)

### Setup

```bash
cd reachy_mini
uv sync --frozen --all-extras --group dev
```

### Run daemon

```bash
uv run reachy-mini-daemon
```

### Tests

```bash
# Standard unit tests (excludes hardware-dependent tests)
uv run pytest -vv -m 'not audio and not video and not wireless' --tb=short

# With MuJoCo simulation disabled
MUJOCO_GL=disable uv run pytest -vv -m 'not audio and not video and not wireless' --tb=short

# Run a single test file
uv run pytest tests/test_motion.py -vv
```

Test markers: `audio`, `video`, `wireless`, `ipc_resolution`

### Lint & Format

```bash
uv run ruff check src/ --select I --select D
uv run ruff format src/
```

### Type check

```bash
mypy src/ --strict
```

### Generate OpenAPI spec (CI drift check)

```bash
uv run python scripts/generate_openapi.py
```

---

## Desktop App (`reachy-mini-desktop-app/`)

### Prerequisites

Node.js 24.4.0+ LTS, Yarn 1.22+, Rust (latest stable), and [Tauri v2 system dependencies](https://v2.tauri.app/start/prerequisites/).

### Setup

```bash
cd reachy-mini-desktop-app
yarn install
```

### Development

```bash
yarn dev          # Vite frontend only
yarn tauri:dev    # Full Tauri app (requires sidecar built first)
yarn tauri:dev:fresh  # Kill daemon, rebuild sidecar, then dev
```

### Build sidecar (required before tauri:dev/build)

```bash
yarn build:sidecar-macos    # macOS
yarn build:sidecar-linux    # Linux
yarn build:sidecar-windows  # Windows
```

### Build production bundle

```bash
yarn tauri:build
```

### Tests

```bash
yarn test:app      # Vitest unit tests
yarn test:e2e      # WebdriverIO end-to-end
yarn test:all      # All tests (except update-prod)
```

### Lint & Format

```bash
yarn lint          # ESLint
yarn lint:fix      # Auto-fix
yarn format        # Prettier
yarn format:check  # Check only
```

### Utilities

```bash
yarn kill-daemon       # Stop all daemon processes
yarn check-daemon      # Check daemon status
yarn clean             # Clean build artifacts
```

---

## Architecture

### Data flow (real robot)

```
User Python code
  → ReachyMini SDK (TCP/WebSocket)
  → Daemon (FastAPI :8000, RobotBackend)
  → Serial connection
  → Robot hardware (motors, camera, audio)
```

### Data flow (desktop app)

```
Desktop App (React UI)
  → Zustand store + hooks
  → Tauri IPC (Rust)
  → Python sidecar daemon (spawned process)
  → Robot (USB or WiFi/mDNS)
```

### Python SDK internals (`reachy_mini/src/reachy_mini/`)

| Module | Role |
|--------|------|
| `reachy_mini.py` | Main SDK entry point (`ReachyMini` class) |
| `daemon/daemon.py` | Daemon class, pluggable backends, WebSocket server |
| `daemon/app/main.py` | FastAPI app, route registration |
| `io/protocol.py` | Command/response definitions, WebSocket protocol |
| `motion/move.py` | Motion planning, minimum-jerk interpolation |
| `media/` | Camera (GStreamer/WebRTC), audio (DOA), backend abstraction |
| `kinematics/` | IK solvers (Rust-based + optional ONNX/placo) |
| `apps/` | App assistant, HF Spaces discovery, lifecycle management |

Daemon backends are pluggable: `RobotBackend` (real hardware), `MujocoBackend` (physics sim), `MockupSimBackend` (minimal sim).

REST API at `http://localhost:8000/api` (Lite) or `http://reachy-mini.local:8000/api` (Wireless). Interactive docs at `/docs`.

WebSocket pushes robot state at 20 Hz.

### Desktop app internals (`reachy-mini-desktop-app/`)

| Area | Key files |
|------|-----------|
| React entry | `src/App.jsx` (Tauri), `src/WebApp.jsx` (web-only) |
| Views | `src/views/` — priority-based router: PermissionsRequired → UpdateView → FindingRobot → WiFiSetup → Starting → ActiveRobot |
| State | `src/store/` — Zustand with slices, cross-window sync |
| Hooks | `src/hooks/` — organized by domain: `daemon/`, `robot/`, `system/`, `media/` |
| 3D viewer | `src/components/viewer3d/` — Three.js URDF visualization at 20 Hz |
| Rust backend | `src-tauri/src/lib.rs` — IPC handlers; submodules: `daemon/`, `usb/`, `discovery/`, `wifi/`, `update/`, `python/` |
| Local proxy | `src-tauri/src/local_proxy.rs` — TCP/UDP proxy for Private Network Access bypass |
| Platform configs | `src-tauri/tauri.{macos,windows,linux}.conf.json` |
