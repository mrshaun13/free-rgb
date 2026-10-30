# Free RGB — Complete Build Specification

This document is the authoritative specification for building the Free RGB lighting control system. It is designed to be handed to an AI coding assistant along with the user's hardware details. The AI should produce a complete, working, production-quality codebase by following this spec precisely.

> **To the AI reading this:** This spec is detailed and opinionated on purpose. Follow it closely. When the spec says "must", it means "must". **This repository contains only documentation.** You are expected to generate every file listed in [Section 3](#3-folder-structure) — Python, HTML, `docker-compose.yml`, `docker-compose.wsl2.yml`, and both Dockerfiles — from this spec. Where this spec shows a file in full (compose files, Dockerfiles), reproduce it, substituting only the values the user supplies. If anything in README.md or CONTRIBUTING.md disagrees with this file, **this file wins**.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Architecture Overview](#2-architecture-overview)
3. [Folder Structure](#3-folder-structure)
4. [Configuration (Environment Variables)](#4-configuration-environment-variables)
5. [Docker Configuration](#5-docker-configuration)
6. [Backend: Web Server (rgb_server.py)](#6-backend-web-server)
7. [Backend: Effects Engine (rgb_effects.py)](#7-backend-effects-engine)
8. [Frontend: Single-Page Application (index.html)](#8-frontend-single-page-application)
9. [Effect Catalog](#9-effect-catalog)
10. [Theme System](#10-theme-system)
11. [1337 Mode (Easter Egg)](#11-1337-mode-easter-egg)
12. [Error Handling](#12-error-handling)
13. [Security](#13-security)
14. [Platform-Specific Setup](#14-platform-specific-setup)
15. [Extension Points](#15-extension-points)

---

## 1. Prerequisites

### Step 0: Compatibility Check (MANDATORY)

Before writing any code, the user **must** confirm that OpenRGB can detect and control their RGB devices. This is not optional. If OpenRGB cannot see the hardware, nothing else in this project will work.

**Instructions for the user:**

1. Download and install [OpenRGB](https://openrgb.org/releases.html) for your platform.
2. Launch OpenRGB with administrative privileges (required for SMBus/I2C access), or install OpenRGB's udev rules on Linux.
3. Go to the **Devices** tab.
4. Confirm your motherboard controller (or other RGB controllers) appear in the device list.
5. Click on a device, manually change a color, and verify the physical LEDs respond.
6. For addressable (ARGB) headers, use the zone **Resize** button to set the number of LEDs physically connected, and confirm the whole strip/fan chain lights up.
7. Go to the **Information** tab — note down:
   - Device name(s) (e.g., `Example Board X1`, `Example RGB DRAM`)
   - Zone name(s) and LED counts per zone (e.g., `ARGB Header 1: 30 LEDs`, `ARGB Header 2: 40 LEDs`)
   - Total LED count

> Addressable headers cannot report how many LEDs are physically attached — OpenRGB only knows the size you give it. The counts the user provides here become the **configured** zone sizes (Section 7). Everything downstream (preview ring, power estimate) uses the sizes OpenRGB reports **after** resizing, never a hard-coded number.

**If this fails:** Check the [OpenRGB supported devices list](https://openrgb.org/devices.html), ensure required kernel modules are loaded (Linux, Step 3), and consult the [OpenRGB FAQ](https://openrgb.org/faq.html).

### Step 1: Gather Hardware Information

The user must provide the AI with:

| Information | Example (placeholder) | Why It's Needed |
|-------------|-----------------------|-----------------|
| Motherboard or controller name | `Example Board X1` | Device discovery (substring match) |
| Zone names and LED counts | `ARGB Header 1: 30`, `ARGB Header 2: 40` | Zone sizing and effect rendering |
| Physical layout description | "3 front fans on header 1; 3 top fans + AIO pump head (8 LEDs, last in chain) on header 2" | Zone targeting, optional pump-head-aware effects |
| Operating system + run mode | "Linux, Docker" / "Linux, native" / "Windows 11 + WSL2" | Platform-specific setup (Section 14) |

If the user has only **one** ARGB zone, map it to `bottom` and leave the top zone unset (Section 7, "Hardware Configuration").

### Step 2: Software Requirements

| Requirement | Version | Notes |
|-------------|---------|-------|
| OpenRGB | 0.9+ (1.0 release candidate recommended) | Must detect the user's hardware |
| Docker Engine + Docker Compose plugin | 24.x+ / 2.x+ | Only for the Docker run mode |
| Python | 3.11+ | Bundled in the Docker image; needed on the host only for the native run mode |
| Git | Any | Clone the repo |
| Web browser | Any modern | Chrome, Firefox, Safari, Edge |

### Step 3: Linux Kernel Modules (Linux only)

For motherboard RGB control via SMBus/I2C, these kernel modules must be loaded on the **host** (not inside Docker). Load `i2c-dev` plus the SMBus driver for your chipset:

| Chipset | SMBus driver module |
|---------|---------------------|
| Intel (most desktop boards) | `i2c-i801` |
| AMD (most desktop boards) | `i2c-piix4` |
| Other | See the OpenRGB documentation for your platform |

```bash
# Load immediately — Intel
sudo modprobe i2c-dev
sudo modprobe i2c-i801

# Load immediately — AMD
sudo modprobe i2c-dev
sudo modprobe i2c-piix4

# Persist across reboots (use i2c-piix4 instead of i2c-i801 on AMD)
echo "i2c-dev"  | sudo tee /etc/modules-load.d/i2c-dev.conf
echo "i2c-i801" | sudo tee /etc/modules-load.d/i2c-smbus.conf
```

Some AMD boards also need the kernel parameter `acpi_enforce_resources=lax` for OpenRGB to access the SMBus; follow the OpenRGB docs if your controller is not detected. USB-based RGB controllers do not need these modules.

---

## 2. Architecture Overview

### Components

There are three runtime pieces. How they are packaged depends on the run mode (Section 14):

| Component | What it does | Docker mode (Linux) | WSL2 mode | Native mode |
|-----------|--------------|---------------------|-----------|-------------|
| OpenRGB SDK server | Owns the hardware, listens on TCP 6742 | `openrgb` container (privileged) | OpenRGB on Windows | OpenRGB on the host |
| Web server (`rgb_server.py`) | Serves UI + REST API on port 7777, manages effect subprocesses | `rgb-lights` container | `rgb-lights` container in WSL2 | `python3` on the host |
| Effect subprocess (`rgb_effects.py`) | Renders frames and sends them to OpenRGB | Inside `rgb-lights` | Inside `rgb-lights` | Child of the native web server |

### Docker Mode (Linux)

```
┌──────────────────────────────────────────────────────────────────┐
│  Docker network: rgb-net  (6742 is NOT published to the host)    │
│                                                                  │
│  ┌──────────────────┐   TCP 6742    ┌──────────────────────────┐ │
│  │ openrgb          │◀──────────────│ rgb-lights               │ │
│  │ OpenRGB SDK      │               │ rgb_server.py            │ │
│  │ server           │               │   └─ spawns ─▶           │ │
│  │ (privileged)     │               │      rgb_effects.py      │ │
│  └────────┬─────────┘               │      (subprocess)        │ │
│           │                         └────────────┬─────────────┘ │
└───────────┼──────────────────────────────────────┼───────────────┘
            │                                      │ published as
     /dev/i2c-*, USB                               │ 127.0.0.1:7777 (default)
       (hardware)                                  ▼
                                          Browser http://localhost:7777
```

### Key Design Decisions

1. **OpenRGB is the only component that touches hardware.** The web app never accesses hardware directly.
2. **The effect subprocess communicates with OpenRGB via the SDK protocol** (raw TCP, default port 6742) — not via CLI, not via REST, not via direct hardware access.
3. **Effects run as subprocesses** — the web server spawns `rgb_effects.py` as a child process with an argument list (never a shell string). This isolates crashes, simplifies state management, and makes it trivial to kill/restart effects.
4. **Zero external Python dependencies** — the entire backend uses only Python stdlib. The OpenRGB SDK protocol is implemented from scratch using raw sockets. No `pip install` required, which is also what makes the native run mode trivial.
5. **Single-file frontend** — `index.html` contains all HTML, CSS, and JavaScript inline. No build tools, no npm, no bundler, no framework. Vanilla JS only.
6. **State persistence** — the last-active effect, zone, speed, brightness, static flag, and color are saved to `state.json` in the state directory and auto-restored when the web server starts.
7. **Nothing user- or machine-specific is hard-coded.** Hostname label, timezone, ports, bind addresses, and hardware names are configuration (Section 4 and Section 7).

### Request Flow

1. User opens `http://localhost:7777` (or the LAN/proxy address they configured) in a browser.
2. Browser fetches `GET /api/effects` (catalog), `GET /api/info` (zone sizes), and `GET /api/status` (current state + display label).
3. Browser renders the effect grid with animated mini-previews (canvas-based) and sizes the preview ring from the reported zone sizes.
4. User clicks an effect → browser sends `POST /api/effect` with `Content-Type: application/json` and body `{ effect, zone, speed, brightness, static, color }`.
5. `rgb_server.py` validates the request (Section 6 and Section 13), kills any existing effect subprocess, and spawns a new one:
   ```
   python3 effects/rgb_effects.py --effect fire --zone both --speed 1.5 --brightness 0.8 --host openrgb --port 6742
   ```
6. The subprocess connects to the OpenRGB SDK server via TCP, discovers the device, resizes zones if needed, and enters a render loop at 30 FPS.
7. Each frame: `effect.render(t, zone.led_count)` per target zone → build the full-device color array → send an `UPDATELEDS` packet to OpenRGB.
8. Browser polls `GET /api/status` every 3 seconds to stay synced.

---

## 3. Folder Structure

Everything except the four documentation files and `.gitattributes` is **generated by the AI from this spec**; those generated files are not checked into this repository.

```
free-rgb/
├── docker-compose.yml              # (generated) Linux Docker mode: openrgb + rgb-lights
├── docker-compose.wsl2.yml         # (generated) WSL2 mode: rgb-lights only
├── Dockerfile                      # (generated) OpenRGB SDK server image
├── Dockerfile.lights               # (generated) Web app image
├── rgb_server.py                   # (generated) Web server + subprocess manager
├── effects/
│   └── rgb_effects.py              # (generated) Effects engine + OpenRGB SDK protocol
├── static/
│   └── index.html                  # (generated) Complete SPA (HTML + CSS + JS, single file)
├── .gitattributes                  # Forces LF line endings (important for WSL2/Windows checkouts)
├── SPEC.md                         # This file — the source of truth
├── CONTRIBUTING.md                 # Contributor guide
├── README.md                       # Project overview
└── LICENSE                         # MIT
```

Generated runtime directories (add them to `.gitignore` if the user commits their generated code): `openrgb-config/` (OpenRGB profiles in Docker mode) and `data/` (only if the user points `STATE_DIR` there).

**Why this structure:**
- Flat top level — no unnecessary nesting. The two Python files and the HTML file are the entire application.
- `effects/` is a directory to allow future modularization (splitting effects into separate files).
- `static/` follows the convention of static file serving and keeps the HTML separate from Python.
- Docker files at the root — standard convention, easy to find.
- `.gitattributes` with `* text=auto eol=lf` prevents Windows checkouts from converting scripts and compose files to CRLF, which breaks shebangs and shell snippets inside WSL2 and containers.

---

## 4. Configuration (Environment Variables)

All configuration is via environment variables. **Nothing machine- or user-specific is hard-coded** — only generic defaults.

### Web Server (`rgb_server.py`)

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENRGB_HOST` | `127.0.0.1` | Hostname/IP of the OpenRGB SDK server |
| `OPENRGB_PORT` | `6742` | Port of the OpenRGB SDK server |
| `PORT` | `7777` | Port the web server listens on (read from the environment — never a hard-coded constant) |
| `BIND_HOST` | `127.0.0.1` | Address the web server binds to. The Docker images set `0.0.0.0` *inside the container*; exposure is then controlled by the compose port binding (Section 13). |
| `STATE_DIR` | *(auto, see below)* | Directory for `state.json` |
| `HOSTNAME_LABEL` | `Free RGB` | Text shown in the UI footer to identify this machine. Set it to any friendly name. |
| `CORS_ORIGINS` | *(empty)* | Comma-separated list of extra origins allowed to call the API cross-origin. Empty = same-origin only (Section 13). |
| `TZ` | `UTC` | Timezone for log timestamps |

**`STATE_DIR` resolution** (first match wins; the chosen directory is created with `mkdir(parents=True, exist_ok=True)` and logged at startup):
1. `STATE_DIR` if set.
2. `/data` if it exists and is writable (the Docker volume).
3. `$XDG_STATE_HOME/free-rgb` (default `~/.local/state/free-rgb`) on Linux/macOS, or `%LOCALAPPDATA%\free-rgb` on Windows.

If the directory cannot be created or written, log a warning and keep running without persistence — never crash because state can't be saved.

**`HOSTNAME_LABEL` note:** inside a container `socket.gethostname()` returns a random container ID, so the default is the generic `Free RGB`, not the machine hostname. Users who want the real hostname can set `HOSTNAME_LABEL=$(hostname)` in their shell or `.env` file. Never bake a specific machine name into generated code.

### Effects Engine (`rgb_effects.py`)

Optional overrides for the hardware constants in Section 7 — useful so users can retune without editing code:

| Variable | Overrides | Example |
|----------|-----------|---------|
| `RGB_DEVICE_NAME` | `DEVICE_NAME` | `Example Board X1` |
| `RGB_ZONE_BOTTOM` | `ZONE_BOTTOM` | `ARGB Header 1` |
| `RGB_ZONE_TOP` | `ZONE_TOP` | `ARGB Header 2` (empty = no top zone) |
| `RGB_ZONE_BOTTOM_LEDS` | `ZONE_BOTTOM_LED_COUNT` | `30` |
| `RGB_ZONE_TOP_LEDS` | `ZONE_TOP_LED_COUNT` | `40` |
| `RGB_PUMP_HEAD_LEDS` | `PUMP_HEAD_LEDS` | `8` (0 = no pump head) |

The effect subprocess inherits the web server's environment, so setting these on the `rgb-lights` service (or the native shell) is enough.

### Docker Compose (`.env` file next to the compose file)

| Variable | Default | Purpose |
|----------|---------|---------|
| `TZ` | `UTC` | Timezone for both containers |
| `PORT` | `7777` | Web UI port (host side and container side) |
| `WEB_BIND` | `127.0.0.1` | Host address the web UI is published on. Set to a LAN address of the machine to reach it from a phone (see Section 13). |
| `HOSTNAME_LABEL` | `Free RGB` | Footer label |
| `OPENRGB_HOST` | *(WSL2 file only)* `host.docker.internal` | Where the Windows OpenRGB SDK server is reachable |

The compose files set `OPENRGB_HOST`, `OPENRGB_PORT`, `STATE_DIR=/data`, and `BIND_HOST=0.0.0.0` for the `rgb-lights` container themselves.

---

## 5. Docker Configuration

The AI generates these files exactly as shown (substituting only user-supplied values). They are **not** in the repository.

### `docker-compose.yml` (Linux Docker mode)

```yaml
services:
  openrgb:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: openrgb
    restart: unless-stopped
    privileged: true                    # Required for I2C/SMBus and USB hardware access
    environment:
      - TZ=${TZ:-UTC}
    volumes:
      - ./openrgb-config:/root/.config/OpenRGB    # Persistent OpenRGB config/profiles
    expose:
      - "6742"                          # Reachable by rgb-lights on rgb-net only — NOT published to the host
    # Only if an SDK client on this same host must connect, publish on loopback only:
    # ports:
    #   - "127.0.0.1:6742:6742"
    networks:
      - rgb-net

  rgb-lights:
    build:
      context: .
      dockerfile: Dockerfile.lights
    container_name: rgb-lights
    restart: unless-stopped
    environment:
      - TZ=${TZ:-UTC}
      - PORT=${PORT:-7777}
      - BIND_HOST=0.0.0.0               # Inside the container; host exposure is set by "ports" below
      - OPENRGB_HOST=openrgb
      - OPENRGB_PORT=6742
      - STATE_DIR=/data
      - HOSTNAME_LABEL=${HOSTNAME_LABEL:-Free RGB}
    volumes:
      - rgb-state:/data                 # Persistent effect state
    ports:
      - "${WEB_BIND:-127.0.0.1}:${PORT:-7777}:${PORT:-7777}"
    depends_on:
      - openrgb
    networks:
      - rgb-net

volumes:
  rgb-state:

networks:
  rgb-net:
```

**Important notes:**
- The `openrgb` container **must** be privileged for I2C/SMBus hardware access. The host must have the kernel modules from Section 1, Step 3 loaded.
- Port 6742 is **not published**: both containers share `rgb-net`, so `rgb-lights` reaches `openrgb:6742` directly. The OpenRGB SDK has no authentication — publishing it would let anything that can reach the port control the lights.
- The web UI is published on `127.0.0.1` by default. To use it from a phone on your LAN, set `WEB_BIND` in `.env` to the machine's LAN address (or `0.0.0.0` behind a host firewall) — read Section 13 first.
- The `openrgb-config` bind mount persists OpenRGB's device profiles and zone sizes between restarts. The `rgb-state` named volume persists the last-active effect.
- Timezone comes from `TZ` only. Do not bind-mount `/etc/localtime` (it is host-specific and does not exist on every platform).

### `docker-compose.wsl2.yml` (Windows + WSL2 mode)

A **standalone** file (used on its own with `-f`, not layered on top of `docker-compose.yml`). It contains only the web app, with **no `depends_on`** — the OpenRGB service doesn't exist in this mode, and a `depends_on: openrgb` would make Compose refuse to start.

```yaml
services:
  rgb-lights:
    build:
      context: .
      dockerfile: Dockerfile.lights
    container_name: rgb-lights
    restart: unless-stopped
    environment:
      - TZ=${TZ:-UTC}
      - PORT=${PORT:-7777}
      - BIND_HOST=0.0.0.0
      - OPENRGB_HOST=${OPENRGB_HOST:-host.docker.internal}   # OpenRGB running on Windows (see Section 14)
      - OPENRGB_PORT=6742
      - STATE_DIR=/data
      - HOSTNAME_LABEL=${HOSTNAME_LABEL:-Free RGB}
    extra_hosts:
      - "host.docker.internal:host-gateway"   # Makes the name resolve on Docker Engine too; harmless on Docker Desktop
    volumes:
      - rgb-state:/data
    ports:
      - "${WEB_BIND:-127.0.0.1}:${PORT:-7777}:${PORT:-7777}"

volumes:
  rgb-state:
```

Run it with:

```bash
docker compose -f docker-compose.wsl2.yml up -d --build
```

### `Dockerfile` (OpenRGB SDK Server)

```dockerfile
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates wget tzdata \
    libusb-1.0-0 libhidapi-hidraw0 libhidapi-libusb0 \
    i2c-tools libmbedtls14 libstdc++6 \
    libqt5core5a libqt5gui5 libqt5widgets5

ARG OPENRGB_VERSION=1.0rc2
ARG OPENRGB_HASH=0fca93e
RUN wget -q -O /tmp/openrgb.deb \
    "https://codeberg.org/OpenRGB/OpenRGB/releases/download/release_candidate_${OPENRGB_VERSION}/openrgb_${OPENRGB_VERSION}_amd64_bookworm_${OPENRGB_HASH}.deb" \
    && (dpkg -i /tmp/openrgb.deb || apt-get install -f -y) \
    && rm /tmp/openrgb.deb \
    && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /root/.config/OpenRGB
EXPOSE 6742
ENTRYPOINT ["openrgb"]
# 0.0.0.0 is inside the container only; the port is not published (see docker-compose.yml)
CMD ["--server", "--server-host", "0.0.0.0", "--server-port", "6742"]
```

**Notes:**
- Check [OpenRGB releases](https://codeberg.org/OpenRGB/OpenRGB/releases) for the current `.deb` URL and update `OPENRGB_VERSION` and `OPENRGB_HASH` accordingly. If the URL 404s, the build fails at this step — that is the first thing to check.
- Qt5 libraries are required even for headless/server mode.

### `Dockerfile.lights` (Web App)

```dockerfile
FROM python:3.12-slim
ENV PYTHONUNBUFFERED=1
WORKDIR /app
COPY rgb_server.py .
COPY effects/ ./effects/
COPY static/ ./static/
RUN mkdir -p /data
EXPOSE 7777
CMD ["python3", "rgb_server.py"]
```

**Zero pip dependencies.** The entire backend is pure Python stdlib. If the base image lacks `/usr/share/zoneinfo`, add `tzdata` via `apt-get` so `TZ` takes effect.

---

## 6. Backend: Web Server

### File: `rgb_server.py`

**Role:** HTTP server that serves the web UI, exposes a REST API, and manages `rgb_effects.py` as a subprocess.

**Dependencies:** Python stdlib only — `http.server`, `subprocess`, `json`, `signal`, `threading`, `pathlib`, `time`, `os`, `sys`, `re`, `math`, `mimetypes`.

Use `http.server.ThreadingHTTPServer` so a slow `/api/info` call (up to 15 s) never blocks status polling. All effect state changes go through the `EffectManager` lock.

### Configuration Constants

```python
PORT            = int(os.environ.get("PORT", "7777"))
BIND_HOST       = os.environ.get("BIND_HOST", "127.0.0.1")
OPENRGB_HOST    = os.environ.get("OPENRGB_HOST", "127.0.0.1")
OPENRGB_PORT    = os.environ.get("OPENRGB_PORT", "6742")
HOSTNAME_LABEL  = os.environ.get("HOSTNAME_LABEL", "").strip() or "Free RGB"
CORS_ORIGINS    = {o.strip() for o in os.environ.get("CORS_ORIGINS", "").split(",") if o.strip()}
STATE_DIR       = resolve_state_dir()           # See Section 4, "STATE_DIR resolution"
STATE_FILE      = STATE_DIR / "state.json" if STATE_DIR else None
BASE_DIR        = Path(__file__).resolve().parent
STATIC_DIR      = BASE_DIR / "static"
EFFECTS_SCRIPT  = BASE_DIR / "effects" / "rgb_effects.py"

MAX_BODY_BYTES  = 4096                          # Request body size limit (Section 13)

SPEED_MIN, SPEED_MAX, SPEED_STEP = 0.25, 3.0, 0.25   # The ONE speed range — API, CLI, and UI all use it
ZONES           = ("both", "bottom", "top")
HEX_COLOR_RE    = re.compile(r"^#[0-9a-fA-F]{6}$")

DEFAULTS = {
    "zone": "both",
    "speed": 1.0,
    "brightness": 1.0,
    "fps": 30,
}
```

### Effect Metadata Registry

The server maintains a static list of all effects with their display metadata. This list drives the `/api/effects` endpoint and is the source of truth for effect validation.

```python
EFFECTS = [
    {"name": "effect_name", "description": "Human-readable description", "category": "Category Name"},
    # ... one entry per visible effect, in catalog order (Section 9)
]

EFFECT_NAMES = {e["name"] for e in EFFECTS}
# Hidden effects (accepted by API but not shown in UI):
EFFECT_NAMES.add("solid_color")
```

**8 categories, 47 visible effects + 1 hidden.** See [Section 9: Effect Catalog](#9-effect-catalog) for the complete list. The names here, in `ALL_EFFECTS` (Python), and in `renderers` (JavaScript) must be identical. The UI never hard-codes the count; it renders whatever `/api/effects` returns.

### EffectManager Class

Singleton process manager with thread-safe state.

**State shape:**
```python
{
    "running": bool,        # Is an effect subprocess alive?
    "effect": str | None,   # Current effect name
    "zone": str,            # "both", "bottom", or "top"
    "speed": float,         # Animation speed multiplier, SPEED_MIN–SPEED_MAX
    "brightness": float,    # Global brightness 0.0–1.0
    "static": bool,         # Static rendering mode (flags only)
    "color": str | None,    # Lowercase hex color for solid_color (e.g., "#ff0000")
}
```

Plus internal tracking:
- `_started_at` — `time.monotonic()` when the current effect was started (for uptime display).
- `_server_boot` — `time.time()` when the server process started (wall clock, for 1337 mode stats).
- `_info_cache` — last successful `/api/info` result (see below).

**Methods:**

| Method | Behavior |
|--------|----------|
| `load_state()` | On startup, reads `STATE_FILE` (if persistence is available). Re-validates every field with the same rules as `POST /api/effect`; discards invalid values. If a valid effect is persisted and was running, auto-starts it. |
| `_save_state()` | Writes current state to `STATE_FILE` as JSON (write to a temp file, then `os.replace`). Called after every state change. Logs and continues on failure. |
| `start_effect(effect, zone, speed, brightness, static, color)` | Acquires lock. Kills existing subprocess. Builds the argument list. Spawns new subprocess. Updates state. Saves state. |
| `stop()` | Acquires lock. Kills subprocess. Sets `running=False, effect=None`. Saves state. Runs a **separate synchronous subprocess** with `--off` (timeout 10 s) to explicitly zero all LEDs. |
| `_stop_subprocess()` | Graceful kill: `terminate()` → wait 5 s → `kill()` → wait 3 s. |
| `get_status()` | Returns state dict plus computed `uptime`, `server_boot`, and `label`. Checks if process died unexpectedly. |
| `get_info(refresh=False)` | Returns cached device/zone info, or runs `rgb_effects.py --info --json` (timeout 15 s) and caches a successful result. |
| `shutdown()` | Called on SIGINT/SIGTERM. Kills subprocess. |

**Subprocess command construction** (argument list only — never `shell=True`; every value has already been validated):
```python
cmd = [
    sys.executable, str(EFFECTS_SCRIPT),
    "--effect", effect,
    "--zone", zone,
    "--speed", str(speed),
    "--brightness", str(brightness),
    "--host", OPENRGB_HOST,
    "--port", OPENRGB_PORT,
]
if static:
    cmd.append("--static")
if effect == "solid_color" and color:
    cmd.extend(["--color", color])
```

### HTTP API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Serve `static/index.html` |
| `GET` | `/static/<file>` | Serve files from `STATIC_DIR` only (MIME via `mimetypes`; path-traversal guarded, Section 13) |
| `GET` | `/api/status` | Current effect state + uptime + server boot time + display label |
| `GET` | `/api/effects` | Full effect catalog (array of `{name, description, category}`) |
| `GET` | `/api/info` | Detected device and zone sizes (cached; `?refresh=1` re-queries OpenRGB) |
| `POST` | `/api/effect` | Switch to a new effect |
| `POST` | `/api/off` | Stop current effect, turn all LEDs off |
| `OPTIONS` | `/api/*` | CORS preflight — `204` for allowed origins only, otherwise `403` |

Anything else returns `404` with `{"error": "not found"}`. Unsupported methods return `405`.

**CORS:** Same-origin by default — **no** `Access-Control-Allow-Origin: *`. If the request's `Origin` header exactly matches an entry in `CORS_ORIGINS`, echo that origin back in `Access-Control-Allow-Origin` with `Vary: Origin`, `Access-Control-Allow-Methods: GET, POST, OPTIONS`, and `Access-Control-Allow-Headers: Content-Type`. Otherwise send no CORS headers.

**Security headers on every response:** `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, and on HTML: `Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'`.

**Request logging:** Suppress per-request log lines (too noisy for a polling UI). Only log errors (status >= 400).

#### `GET /api/status` Response

```json
{
  "running": true,
  "effect": "fire",
  "zone": "both",
  "speed": 1.5,
  "brightness": 0.8,
  "static": false,
  "color": null,
  "uptime": "2h 15m",
  "server_boot": 1777015554.578,
  "label": "Free RGB"
}
```

- `uptime`: Human-readable string. `null` when not running. Format: `"Xh Ym"` if hours > 0, else `"Ym"`.
- `server_boot`: Unix timestamp (float) of when the server process started. Used by the frontend for real uptime calculations.
- `label`: The `HOSTNAME_LABEL` value, shown in the footer.

#### `POST /api/effect` Request

```json
{
  "effect": "fire",
  "zone": "both",
  "speed": 1.5,
  "brightness": 0.8,
  "static": false,
  "color": null
}
```

**Validation** (reject with `400 {"error": "<reason>"}` unless stated otherwise):
- Body: must have `Content-Type: application/json` (else `415`), a `Content-Length` ≤ `MAX_BODY_BYTES` (else `413`; missing length → `411`), and parse to a JSON object (else `400`).
- `effect`: Required string. Must be in `EFFECT_NAMES`.
- `zone`: Optional. Must be one of `"both"`, `"bottom"`, `"top"`. Defaults to `"both"`. If the requested zone is not configured (e.g. `"top"` on a single-zone setup), return `400`.
- `speed`: Optional. Must be a finite number (not a boolean, not NaN/Infinity). Clamped to `[SPEED_MIN, SPEED_MAX]` = `[0.25, 3.0]`. Defaults to `1.0`.
- `brightness`: Optional. Finite number, clamped to `[0.0, 1.0]`. Defaults to `1.0`.
- `static`: Optional. Must be a JSON boolean. Defaults to `false`.
- `color`: Optional. `null` or a string matching `^#[0-9a-fA-F]{6}$` exactly (no shorthand, no names, no whitespace); normalized to lowercase. Only used by `solid_color`; `solid_color` without a color defaults to `#ffffff`.
- Unknown keys are ignored.

#### `POST /api/effect` Response

```json
{
  "ok": true,
  "effect": "fire",
  "zone": "both",
  "speed": 1.5,
  "brightness": 0.8,
  "static": false,
  "color": null
}
```

The response echoes the **clamped/normalized** values actually applied.

#### `POST /api/off`

Same body rules as above (an empty JSON object `{}` is fine). Response: `{"ok": true}`.

#### `GET /api/info` Response

```json
{
  "device": "Example Board X1",
  "zones": [
    {"key": "bottom", "name": "ARGB Header 1", "led_count": 30},
    {"key": "top",    "name": "ARGB Header 2", "led_count": 40}
  ],
  "total_leds": 70,
  "pump_head_leds": 0,
  "output": "Device 0: Example Board X1\n  Zone 0: ARGB Header 1 (30 LEDs)\n  Zone 1: ARGB Header 2 (40 LEDs)\n"
}
```

- `zones` lists only the **configured** zones (`bottom`, and `top` if set), with the sizes OpenRGB reports after resizing.
- `total_leds` is the sum of the configured zones' `led_count`.
- `output` is the human-readable text from `rgb_effects.py --info`, for display/debugging.

On error (not cached):
```json
{"error": "OpenRGB connection timed out"}
```

### Entry Point

```python
def main():
    signal.signal(signal.SIGINT, on_signal)
    signal.signal(signal.SIGTERM, on_signal)
    log(f"State dir: {STATE_DIR or 'disabled'}; OpenRGB at {OPENRGB_HOST}:{OPENRGB_PORT}")
    manager.load_state()        # Auto-restore last effect
    server = ThreadingHTTPServer((BIND_HOST, PORT), RGBHandler)
    log(f"Listening on http://{BIND_HOST}:{PORT}")
    server.serve_forever()
```

---

## 7. Backend: Effects Engine

### File: `effects/rgb_effects.py`

**Role:** Standalone CLI tool that connects to the OpenRGB SDK server, renders effects in a loop, and pushes LED color updates. Spawned as a subprocess by `rgb_server.py`; also usable by hand for debugging.

**Dependencies:** Python stdlib only — `socket`, `struct`, `math`, `time`, `signal`, `argparse`, `sys`, `os`, `json`, `re`, `abc`, `dataclasses`, `random`, `typing`. **No pip packages.**

### OpenRGB SDK Protocol Implementation

The effects engine implements the OpenRGB SDK binary protocol from scratch using raw TCP sockets.

**Protocol basics:**
- Magic: `b"ORGB"` (4 bytes)
- Header: magic (4) + device_index (4) + packet_id (4) + data_length (4) = 16 bytes
- All integers are little-endian unsigned 32-bit unless noted.
- Strings are null-terminated and prefixed with a 2-byte length (including the null byte).
- Socket timeout: 5 seconds for connect and reads.

**Required packet types:**

| Packet ID | Name | Direction | Purpose |
|-----------|------|-----------|---------|
| 0 | `REQUEST_CONTROLLER_COUNT` | Client → Server | Get number of detected devices |
| 1 | `REQUEST_CONTROLLER_DATA` | Client → Server | Get full device info (zones, LEDs) |
| 50 | `SET_CLIENT_NAME` | Client → Server | Identify the client (`"Free RGB"`) |
| 1000 | `RESIZEZONE` | Client → Server | Set LED count for a zone |
| 1050 | `UPDATELEDS` | Client → Server | Send color data to device |

**Connection class must implement:**

```python
class OpenRGBConnection:
    def connect()                                          # TCP connect + set client name
    def close()                                            # Close socket
    def request_controller_count() -> int                  # Get device count
    def request_controller_data(device_idx) -> DeviceInfo  # Parse device/zone info
    def resize_zone(device_idx, zone_idx, new_size)        # Set zone LED count
    def update_device_leds(device_idx, colors)             # Send full color array
    def set_all_off(device_idx, total_leds)                # Zero all LEDs
```

**Device data parsing strategy:** The binary format for controller data is complex and varies between OpenRGB versions. Rather than implementing a full parser with exact byte offsets, **search for the configured zone name strings in the raw binary data** and extract zone information (index, LED start, LED count) from their surrounding context. This is more robust across different hardware configurations. If a configured zone name is not found, exit with a clear error listing the zone names that were found.

**Data classes:**

```python
@dataclass
class ZoneInfo:
    name: str
    zone_idx: int
    led_start: int
    led_count: int

@dataclass
class DeviceInfo:
    device_idx: int
    name: str
    total_leds: int
    zones: List[ZoneInfo]

    def zone_by_name(self, name: str) -> Optional[ZoneInfo]:
        """Find zone by name (case-insensitive substring match)."""
```

### Hardware Configuration (USER MUST CUSTOMIZE)

These constants are set from the user's hardware details (Section 1). Each can be overridden by the environment variable in Section 4.

```python
# Device discovery — case-insensitive substring match against OpenRGB device names
DEVICE_NAME = os.environ.get("RGB_DEVICE_NAME", "YOUR_DEVICE_NAME_HERE")       # e.g. "Example Board X1"

# Zone names — must match what OpenRGB reports for your hardware
ZONE_BOTTOM = os.environ.get("RGB_ZONE_BOTTOM", "YOUR_BOTTOM_ZONE_NAME")       # e.g. "ARGB Header 1"
ZONE_TOP    = os.environ.get("RGB_ZONE_TOP",    "YOUR_TOP_ZONE_NAME")          # e.g. "ARGB Header 2"; "" = single-zone setup

# Configured LED counts per addressable zone — must match the physical hardware.
# These are the sizes we ASK OpenRGB to use. Rendering always uses the sizes OpenRGB
# reports back after resizing (ZoneInfo.led_count), never these constants directly.
ZONE_BOTTOM_LED_COUNT = int(os.environ.get("RGB_ZONE_BOTTOM_LEDS", "30"))       # placeholder
ZONE_TOP_LED_COUNT    = int(os.environ.get("RGB_ZONE_TOP_LEDS",    "40"))       # placeholder

# Optional AIO pump head: the LAST N LEDs of the top zone form a center ring. 0 = none.
PUMP_HEAD_LEDS = int(os.environ.get("RGB_PUMP_HEAD_LEDS", "0"))
```

> **AI Implementation Note:** Substitute the user's values as the defaults. If the user has only one zone, set `ZONE_TOP = ""`; then `--zone top` is rejected, `--zone both` means the bottom zone only, and the UI hides the Top/Bottom buttons (Section 8). For zones whose size OpenRGB reports correctly on its own (non-resizable zones), skip resizing and use the reported size.

### Color Helpers

```python
def clamp(v: float) -> int:
    """Clamp to [0, 255] and round to int."""

def lerp_color(c1, c2, t):
    """Linear interpolation between two RGB tuples."""

def hsv_to_rgb(h, s, v):
    """HSV (h: 0-360, s: 0-1, v: 0-1) to RGB tuple."""

def dim(color, factor):
    """Multiply all channels by a factor."""

def parse_hex(s: str) -> Tuple[int, int, int]:
    """Parse '#rrggbb' (strict, same regex as the server). Raise ValueError otherwise."""
```

### Base Effect Class

```python
class Effect(ABC):
    name: str = "base"
    description: str = "Base effect"

    @abstractmethod
    def render(self, t: float, num_leds: int) -> List[Tuple[int, int, int]]:
        """
        Render one frame of the effect.

        Args:
            t: Elapsed time in seconds, pre-multiplied by speed.
            num_leds: Number of LEDs in the target zone (as reported by OpenRGB).

        Returns:
            List of exactly num_leds (R, G, B) tuples.
        """

    def render_static(self, num_leds: int) -> Optional[List[Tuple[int, int, int]]]:
        """
        Render a static (non-animated) frame. Used for flag effects.
        Returns None if this effect doesn't support static mode (falls back to animated).
        """
        return None
```

Every effect must work for **any** `num_leds ≥ 1` — never assume a specific count.

### Effect Implementation Patterns

**Simple ambient effect (e.g., Fire):**
```python
class Fire(Effect):
    name = "fire"
    description = "Red/orange/yellow flames flickering"

    def render(self, t, num_leds):
        colors = []
        for i in range(num_leds):
            # Use noise functions, sine waves, random flicker
            # Map position (i/num_leds) and time (t) to color
            colors.append((r, g, b))
        return colors
```

**Tricolor flag with static support (reusable base class):**
```python
class _TricolorFlag(Effect):
    """Base for three-stripe flags. Override C1, C2, C3."""
    C1 = (0, 0, 0)
    C2 = (0, 0, 0)
    C3 = (0, 0, 0)

    def render(self, t, num_leds):
        # Animated: flowing gradient with wave motion
        ...

    def render_static(self, num_leds):
        # Static: three equal segments
        third = num_leds // 3
        return [self.C1]*third + [self.C2]*third + [self.C3]*(num_leds - 2*third)

class IrishFlag(_TricolorFlag):
    name = "irish_flag"
    description = "Green, white & orange"
    C1, C2, C3 = (0, 155, 72), (255, 255, 255), (255, 130, 0)
```

**Pump-head-aware effects (for AIO coolers):**
```python
# If PUMP_HEAD_LEDS > 0 and the effect is rendering the top zone, treat the LAST
# PUMP_HEAD_LEDS LEDs of that zone as the pump head (the physical center circle).
# Pump-head-aware effects (japan, bangladesh, arc_reactor) draw their "center"
# feature there. If PUMP_HEAD_LEDS == 0, or for the bottom zone, they draw the
# center feature in the middle of the strip instead (e.g. the middle ~20% of LEDs).
# The JS preview uses the same rule with pump_head_leds from /api/info.
```

**SolidColor (hidden, for 1337 mode):**
```python
class SolidColor(Effect):
    name = "solid_color"
    description = "Solid custom color with subtle breathing"

    def __init__(self, color=(255, 255, 255)):
        self._color = color

    def render(self, t, num_leds):
        br = 0.97 + 0.03 * math.sin(t * 0.8)    # Subtle breathing
        return [dim(self._color, br)] * num_leds

    def render_static(self, num_leds):
        return [self._color] * num_leds
```

### Effect Registry

```python
ALL_EFFECTS = {
    "triforce": Triforce(),
    "fire": Fire(),
    # ... all 47 visible effects from Section 9 ...
    "solid_color": SolidColor(),    # Default white, replaced at runtime when --color is given
}
```

### Zone Management

**`find_device(conn)`** — Scan all devices returned by OpenRGB. Match device name using case-insensitive substring match against `DEVICE_NAME`. Return the first match.

**`ensure_zones_sized(conn, device, wanted)`** — `wanted` maps zone name → configured LED count (from the constants or `--bottom-leds` / `--top-leds`). OpenRGB needs to be told how many LEDs are on each addressable header. Compare current zone sizes to the wanted counts; if different, call `resize_zone()`. After any resize, **re-fetch controller data** (the offsets change after a resize) and return the fresh `DeviceInfo`. All later code uses the returned `ZoneInfo.led_count` values.

### Run Effect Function

```python
def run_effect(effect, conn, device, zone="both", brightness=1.0, fps=30, speed=1.0, static=False):
```

**Signal handling:** Register `SIGTERM`/`SIGINT` handlers that set `running = False` to break the render loop. On exit, always turn LEDs off and close the connection.

**Static mode:**
1. Call `effect.render_static(zone.led_count)`. If it returns `None`, fall back to animated.
2. Build full-device color array (`device.total_leds` entries). Non-target zones stay black `(0,0,0)`.
3. For each target zone: write static colors at `zone.led_start`. Apply brightness.
4. Send one `update_device_leds()` call.
5. Sleep in a loop (`time.sleep(0.5)`) until killed.

**Animated mode:**
1. Calculate frame timing: `frame_time = 1.0 / fps`.
2. Loop while `running`:
   - `t = (time.monotonic() - start_time) * speed`
   - Build full-device color array. Non-target zones stay black.
   - For each target zone: `colors = effect.render(t, zone.led_count)`, apply brightness, write at `zone.led_start`.
   - Send `update_device_leds()`.
   - Sleep for remaining frame time.
3. On exit: LEDs off, connection closed.

**Zone targeting (`--zone` flag):**
- `"both"`: Render the effect on every configured zone (each zone renders independently with its own `led_count`).
- `"bottom"`: Only render on the bottom zone. Other zones stay black.
- `"top"`: Only render on the top zone. Other zones stay black. Error if no top zone is configured.

### CLI Arguments

```
rgb_effects.py [options]

Options:
  --list                        List all available effects
  --info                        Show detected device and zone layout (human-readable)
  --json                        With --info: print the /api/info JSON shape instead
  --probe                       Interactively discover LED counts (light LEDs one by one)
  --effect NAME, -e NAME        Effect name to run
  --off                         Turn all LEDs off
  --zone {both,bottom,top}      Target zone(s) (default: both)
  --host HOST                   OpenRGB server host (default: $OPENRGB_HOST or 127.0.0.1)
  --port PORT                   OpenRGB server port (default: $OPENRGB_PORT or 6742)
  --speed FLOAT                 Animation speed multiplier, clamped to 0.25–3.0 (default: 1.0)
  --brightness FLOAT, -b FLOAT  Global brightness, clamped to 0.0–1.0 (default: 1.0)
  --static                      Use static (non-animated) rendering
  --color HEX                   Hex color #rrggbb for solid_color (e.g., "#ff0000")
  --fps INT                     Target FPS, clamped to 1–60 (default: 30)
  --bottom-leds INT             Override the configured bottom zone LED count
  --top-leds INT                Override the configured top zone LED count
```

**`--color` handling in `main()`:** When `--effect solid_color` is used with `--color`, parse it with `parse_hex()` (exit 2 on invalid input) and create a **new** `SolidColor(color=(r,g,b))` instance, replacing the default from the registry.

---

## 8. Frontend: Single-Page Application

### File: `static/index.html`

**One file. Everything inline.** No external CSS, no external JS, no web fonts, no CDN, no build step, no framework.

Wrapped in an IIFE: `(function() { 'use strict'; ... })();`

**Never inject server-provided strings with `innerHTML`.** Effect names, descriptions, the footer label, and device/zone names are set with `textContent`.

### Constants

```javascript
const API              = window.location.origin;
const POLL_MS          = 3000;     // Status poll interval (milliseconds)
const SPEED_MIN        = 0.25;     // Must match the server's SPEED_MIN/SPEED_MAX/SPEED_STEP
const SPEED_MAX        = 3.0;
const SPEED_STEP       = 0.25;
const FALLBACK_LEDS    = 24;       // Preview ring size ONLY if /api/info is unavailable
const MIN_PREVIEW_LEDS = 8;        // Never draw fewer segments than this
const CARD_LEDS        = 16;       // Mini-preview size on effect cards (cosmetic, fixed)
const WATTS_PER_LED    = 0.06;     // ~60 mW per LED at full white (estimate)
const PRICE_PER_KWH    = 0.12;     // Used by the 1337 "Running Cost" stat (estimate)
const TRIPLE_CLICK_MS  = 500;      // Japan easter-egg window (Section 11)
```

### State Variables

```javascript
let effects        = [];        // Array from /api/effects
let zoneInfo       = null;      // From /api/info: { zones: [{key, name, led_count}], total_leds, pump_head_leds }
let currentEffect  = null;      // Server-confirmed active effect
let currentZone    = 'both';
let currentSpeed   = 1.0;
let currentBright  = 1.0;
let currentStatic  = false;
let selectedEffect = null;      // What the user clicked (may differ during transition)
let cardCanvases   = {};        // Map: effectName → {canvas, ctx, animId}
let ringCtx        = null;      // Main ring canvas context
let ringAnimId     = null;      // Main ring animation frame ID
```

### LED Counts in the UI (derived, never hard-coded)

```javascript
// Number of LEDs actually lit for the current zone selection.
function litLedCount(zone = currentZone) {
    if (!zoneInfo) return FALLBACK_LEDS;
    const zones = zoneInfo.zones.filter(z => zone === 'both' || z.key === zone);
    return zones.reduce((sum, z) => sum + z.led_count, 0) || FALLBACK_LEDS;
}
function previewLedCount() { return Math.max(MIN_PREVIEW_LEDS, litLedCount()); }
```

- The main preview ring draws `previewLedCount()` segments and updates when the zone changes. With `both`, it renders each zone with its own `led_count` and concatenates them — exactly as the hardware does.
- The 1337 Power Draw stat uses `litLedCount()` (Section 11).
- `/api/info` is fetched once on init (and again on a manual refresh); if it fails, the UI uses `FALLBACK_LEDS` and keeps working.
- If `zoneInfo` has only one zone, hide the Bottom/Top zone buttons and always send `zone: "both"`.

### HTML Structure

```html
<body>
  <canvas id="matrix-bg"></canvas>          <!-- 1337 mode background (hidden by default) -->
  <div class="app">
    <header>
      <h1>RGB Lights</h1>
      <span class="status-pill">          <!-- Green when running, dim when off -->
        <span class="dot"></span>
        <span id="status-text">OFF</span>
      </span>
    </header>

    <!-- LED Ring Preview: 440×440 canvas -->
    <div class="preview-section">
      <div class="preview-ring">
        <canvas id="ring-canvas" width="440" height="440"></canvas>
        <div class="preview-label">
          <div class="effect-name" id="preview-name">—</div>
          <div class="effect-detail" id="preview-detail"></div>
        </div>
      </div>
    </div>

    <!-- Controls -->
    <div class="controls">
      <!-- Zone: Both / Bottom / Top (Bottom/Top hidden on single-zone setups) -->
      <!-- Speed: slider SPEED_MIN–SPEED_MAX (0.25–3.0), step 0.25 -->
      <!-- Brightness: slider 0.0–1.0, step 0.05 -->
      <!-- Mode Toggle: Animated / Static (visible only for flag effects) -->
      <!-- OFF button -->
    </div>

    <!-- Effect Grid: dynamically populated by JS -->
    <div id="effects-container"></div>

    <!-- 1337 Mode Panel (hidden until activated) -->
    <div class="leet-panel" id="leet-panel">
      <!-- Color Lab + System Diagnostics -->
    </div>

    <footer>
      <span id="host-label">Free RGB</span> · OpenRGB · <span id="uptime-text"></span>
    </footer>
  </div>
</body>
```

The footer label is **never** hard-coded to a machine name: `#host-label` starts as `Free RGB` and is replaced with `status.label` (from `HOSTNAME_LABEL`) on the first successful status poll. `document.title` is `RGB Lights · <label>`.

### Canvas Drawing System

Two drawing functions, both using arc segments to form a ring:

**`drawRing(ctx, w, h, colors, innerRatio=0.65)`** — Main preview ring.
- Draw `colors.length` arc segments arranged in a circle.
- Gap between segments: `0.015` radians (skip the gap if there are more than 120 segments so they don't vanish).
- Outer radius: `min(w, h) / 2 - 4`.
- Inner radius: `outer * innerRatio`.
- Fill each segment with its color using `arc()` and `fill()`.
- Scale the canvas backing store by `devicePixelRatio` for sharp rendering.

**`drawMiniRing(ctx, w, h, colors)`** — Card preview mini-ring (72×72 px).
- Same logic but `innerRatio=0.55`, no gap.

### Effect Renderer Architecture

Every Python effect class has a **corresponding JavaScript renderer function** that produces identical visual output. This means the browser can preview effects in real time without hitting the server.

```javascript
const renderers = {
    fire(t, n) {
        const out = [];
        for (let i = 0; i < n; i++) {
            // Same math as Python Fire.render()
            out.push([r, g, b]);
        }
        return out;
    },
    // ... one per visible effect
};
```

**Renderer signature:** `(t: number, n: number) => Array<[number, number, number]>`
- `t`: Elapsed time in seconds (pre-multiplied by speed in the animation loop).
- `n`: Number of LEDs to render.
- Returns: Array of exactly `n` `[R, G, B]` arrays.

Effects that use randomness (fire, matrix, disco, …) should use a small seeded PRNG in both languages so the preview *resembles* the hardware; exact frame-for-frame equality is not required for random effects.

**Static renderers** for flag effects:
```javascript
const staticRenderers = {
    american_flag(n) { /* ... */ },
    irish_flag(n) { /* ... */ },
    // ... one per flag effect
};
```
Signature: `(n: number) => Array<[number, number, number]>` — no time parameter.

**Shared helpers** (same as Python):
```javascript
function clamp(v) { return Math.max(0, Math.min(255, Math.round(v))); }
function lerp(a, b, t) { /* linear interpolate two [r,g,b] arrays */ }
function dim(c, f) { /* multiply all channels by factor */ }
function hsvToRgb(h, s, v) { /* HSV to RGB, h: 0-360 */ }
```

### Animation System

```javascript
const t0 = performance.now() / 1000;    // Base time for all animations

function animateRing() {
    // n = previewLedCount()
    // 1. If solid_color: parse hex, render with subtle breathing, draw, recurse.
    // 2. If static mode + static renderer exists: render static, draw, recurse.
    // 3. Otherwise: t = (now - t0) * currentSpeed, render animated, apply brightness, draw, recurse.
    ringAnimId = requestAnimationFrame(animateRing);
}

function animateCard(name) {
    // Render at CARD_LEDS (16), speed=1 (always), draw mini ring.
    info.animId = requestAnimationFrame(() => animateCard(name));
}
```

**All card canvases animate simultaneously** — each gets its own `requestAnimationFrame` loop started when the UI builds. Pause card loops for cards scrolled off-screen (`IntersectionObserver`) and all loops when `document.hidden` is true, to save battery on phones.

### Effect Grid Construction

On init:
1. Fetch `GET /api/effects` and `GET /api/info` (in parallel).
2. Group effects by `category`, preserving the order returned by the server.
3. For each category: create a header + CSS grid.
4. For each effect: create a card with:
   - A 72×72 `<canvas>` for the mini-preview.
   - The effect name (underscores replaced with spaces).
   - The effect description.
   - A click handler that calls `setEffect(name)` (the `japan` card uses the special handler in Section 11 instead).
   - A `data-cat` attribute for category-specific accent colors.
5. Start `animateCard()` for each card.

### Controls

**Zone buttons:** Three buttons (`Both`, `Bottom`, `Top`). Click → set `currentZone`, highlight active, resize preview ring, re-send current effect to API. Bottom/Top are hidden on single-zone setups.

**Speed slider:** `min=0.25, max=3, step=0.25` (`SPEED_MIN`/`SPEED_MAX`/`SPEED_STEP` — the same range the server clamps to). `input` event updates the display label (`1.5×`). `change` event re-sends the effect to the API.

**Brightness slider:** `min=0, max=1, step=0.05`. Same event pattern as speed. Display as a percentage.

**Mode toggle (Animated/Static):** Two buttons. **Only visible when a flag effect is selected.** Managed by the `FLAG_EFFECTS` Set and `updateModeToggle()` function. Non-flag effects force `currentStatic = false`.

```javascript
const FLAG_EFFECTS = new Set([
    'american_flag', 'american_flag_smooth', 'irish_flag', 'italy',
    'france', 'germany', 'ukraine', 'japan', 'bangladesh', 'brazil',
    'jamaica', 'india', 'south_korea', 'canada'
]);   // Exactly the 14 effects in the Flags category (Section 9)
```

**OFF button:** Calls `POST /api/off`, clears `selectedEffect`.

**`setEffect(name, extra = {})`:** POSTs `{ effect, zone, speed, brightness, static, color }` as JSON with `Content-Type: application/json`, then applies the server's echoed (clamped) values to the local state and controls.

### Status Polling

```javascript
async function pollStatus() {
    const s = await fetchJSON('/api/status');
    // Update status pill (green ON / dim OFF)
    // For solid_color: show "SOLID #FF0000" in pill
    // Capture server_boot timestamp
    // Update #host-label from s.label (textContent)
    // Sync local state from server (zone, speed, brightness, static, effect)
    //   BUT skip syncing while the user is interacting (a slider is being dragged,
    //   or a setEffect request is in flight)
    // Update uptime text in footer
    // Update preview detail label
}

// Called on init, then every 3 seconds
setInterval(pollStatus, POLL_MS);
```

### Responsive Design

Single breakpoint at `600px`:

```css
@media (max-width: 600px) {
    .app { padding: 16px 12px 80px; }
    header h1 { font-size: 18px; }
    .preview-ring { width: 160px; height: 160px; }
    .controls { flex-direction: column; align-items: stretch; }
    .effect-grid { grid-template-columns: 1fr 1fr; gap: 8px; }
}
```

Desktop: effect grid uses `auto-fill, minmax(200px, 1fr)`.
Mobile: 2-column grid, stacked controls, smaller preview ring. Touch targets are at least 44×44 px.

---

## 9. Effect Catalog

### 8 Categories, 47 Visible Effects (+ 1 Hidden)

| Category | Count |
|----------|-------|
| Zelda | 6 |
| Elements | 4 |
| Digital | 2 |
| Mood | 2 |
| Flags | 14 |
| Pop Culture | 10 |
| Atmospheric | 4 |
| Party | 5 |
| **Total visible** | **47** |
| Hidden (`solid_color`) | 1 |

If you add or remove an effect, update this table, the headings below, and the counts in Section 6 and README.md.

#### Zelda (6 effects)

| Name | Description | Notes |
|------|-------------|-------|
| `triforce` | Golden triangles pulse and glow | Three triangular LED groups with breathing gold |
| `navi` | Blue-white sparkle darting around | Fast-moving bright point with blue trail |
| `lost_woods` | Enchanted forest greens with fireflies | Deep greens with random warm sparkles |
| `fairy_fountain` | Shimmering pink/purple/cyan magic | Multi-hue shimmer cycling through fairy colors |
| `majoras_mask` | Menacing purple/orange heartbeat pulse | Dark base with threatening pulsing crescendo |
| `master_sword` | Radiant blue blade with holy glow | Cool blue energy radiating from center |

#### Elements (4 effects)

| Name | Description | Notes |
|------|-------------|-------|
| `fire` | Red/orange/yellow flames flickering | Noise-driven fire with random flicker intensity |
| `ocean` | Dark blues and teals with white crests | Sine-wave undulation, occasional bright crests |
| `lava` | Deep reds and oranges oozing slowly | Very slow-moving flow, high saturation |
| `aurora` | Greens, teals, and purples dancing | Wide sine waves blending northern lights palette |

#### Digital (2 effects)

| Name | Description | Notes |
|------|-------------|-------|
| `matrix` | Chaotic green code drips with glitches | **Independent, random** drops: 8 drops, each with its own speed (1.5–4×, golden-ratio phase offsets), bright white-green head, quadratic-falloff green tail of 3–6 LEDs, plus random glitch flickers. Irregular and busy. |
| `rainbow` | Smooth rainbow cycle rotating around | Classic HSV hue sweep |

#### Mood (2 effects)

| Name | Description | Notes |
|------|-------------|-------|
| `breathe` | Gentle single-color breathing pulse | Warm white sine-wave brightness |
| `chill` | Calm lo-fi vibes, soft purple and pink | Slow morph between cool pastels |

#### Flags (14 effects)

All flag effects support both **animated** (flowing wave motion) and **static** (non-animated layout) modes.

| Name | Description | Static Pattern |
|------|-------------|----------------|
| `american_flag` | Red, white & blue stripes | Alternating red/white stripes with blue segment |
| `american_flag_smooth` | Smooth gradient American flag | Blended red/white/blue gradient |
| `irish_flag` | Green, white & orange | Three equal segments |
| `italy` | Green, white & red | Three equal segments |
| `france` | Blue, white, red | Three equal segments |
| `germany` | Black, red, gold | Three equal segments |
| `ukraine` | Blue sky over golden wheat | Two equal halves |
| `japan` | White field with red center | White with red center segment (pump-head-aware). Card also hosts the 1337 easter egg (Section 11). |
| `bangladesh` | Green field with red circle | Green with red center (pump-head-aware) |
| `brazil` | Green with golden diamond | Green field, yellow center pulse |
| `jamaica` | Green, black & gold X | Diagonal pattern |
| `india` | Saffron, white & green with blue | Three segments with blue center |
| `south_korea` | Red & blue yin-yang on white | White with red/blue center swirl |
| `canada` | Red-white-red with maple leaf | Red ends, white center with red center accent |

**Tricolor flag base class:** `irish_flag`, `italy`, `france`, and `germany` share the `_TricolorFlag` base class. Subclasses only override three color constants (`C1`, `C2`, `C3`).

#### Pop Culture (10 effects)

| Name | Description | Notes |
|------|-------------|-------|
| `matrix_rain` | Movie-style synchronized code rain | **Orderly, coordinated** columns: the zone is split into evenly spaced lanes (one head every ~6 LEDs) that all fall at the same speed in lockstep, with a soft fixed-length tail and no glitches. A steady, hypnotic "curtain" — visibly different from the chaotic `matrix`. |
| `cyberpunk` | Hot pink & electric cyan with glitch flashes | |
| `synthwave` | 80s sunset with purple, pink & orange | |
| `portal` | Orange & blue spinning portals on opposite sides | |
| `tron` | Light cycle traces blazing across the grid | |
| `lightsaber` | Blade ignites and hums, cycling colors | |
| `arc_reactor` | Pulsing blue-white energy from center | Pump-head-aware |
| `stranger_things` | Erratic Christmas lights in the Upside Down | |
| `pac_man` | Yellow chomp chasing colorful ghosts | |
| `limelight` | Mirrored L shapes in orange & lime | |

#### Atmospheric (4 effects)

| Name | Description |
|------|-------------|
| `thunderstorm` | Dark clouds with bright lightning strikes |
| `sakura` | Cherry blossom petals drifting on a gentle breeze |
| `sunset` | Warm sky gradient from gold to deep purple |
| `ice` | Cold crystalline blues and whites with frozen shimmer |

#### Party (5 effects)

| Name | Description |
|------|-------------|
| `disco` | Rapid colorful party lights with bright flashes (≤ 4 Hz) |
| `police` | Alternating red & blue with rapid flash (≤ 4 Hz) |
| `heartbeat` | Red cardiac pulse, lub-dub rhythm |
| `halloween` | Spooky orange & purple with eerie flicker |
| `christmas` | Red & green with twinkling white star lights |

#### Hidden (1 effect)

| Name | Description | Access |
|------|-------------|--------|
| `solid_color` | Solid custom color with subtle breathing | 1337 mode only (not in `/api/effects`, not in the grid) |

### JavaScript Renderers

**Every visible effect in the catalog must have a corresponding JavaScript renderer** that produces visually matching output to the Python version (plus a `solid_color` path in `animateRing`). The renderer is used for:
1. The main LED ring preview (440×440 canvas).
2. The mini-preview on each effect card (72×72 canvas).

Both Python `render()` and JS `renderer()` must use the same math, same color values, same variable names where possible. The only difference is language syntax.

Flag effects also need **static renderers** in the `staticRenderers` object.

**Flash-rate rule:** No effect may strobe faster than ~4 Hz at the maximum speed (3.0×) — cap flash frequency in the effect code, not just at speed 1.0 (photosensitivity safety).

---

## 10. Theme System

### CSS Custom Properties

The entire color scheme is driven by CSS custom properties on `:root`. This makes theming trivial — override the variables to change the entire look.

```css
:root {
    --bg:          #0a0e1a;                    /* Page background */
    --bg-card:     #111827;                    /* Card background */
    --bg-hover:    #1a2235;                    /* Hover state */
    --border:      rgba(255,255,255,0.06);     /* Default borders */
    --border-hi:   rgba(255,255,255,0.12);     /* Highlighted borders */
    --text:        #b8c6dd;                    /* Default text */
    --text-dim:    #4c5e78;                    /* Dimmed text */
    --text-bright: #e8eef8;                    /* Bright text */
    --accent:      #6366f1;                    /* Primary accent (indigo) */
    --accent-glow: rgba(99,102,241,0.35);      /* Accent glow/shadow */
    --green:       #4ade80;                    /* Status: active/on */
    --red:         #f87171;                    /* Status: off/error */
    --yellow:      #fbbf24;                    /* Status: warning */
    --radius:      12px;                       /* Default border radius */
    --radius-sm:   8px;                        /* Small border radius */
}
```

### Category Accent Colors

Each effect category has a unique glow color applied to card preview circles via `data-cat` attribute selectors:

| Category | Glow Color | CSS Selector |
|----------|-----------|--------------|
| Zelda | Gold `rgba(255,200,0,0.3)` | `[data-cat="Zelda"]` |
| Elements | Orange `rgba(255,120,0,0.3)` | `[data-cat="Elements"]` |
| Digital | Green `rgba(0,255,100,0.3)` | `[data-cat="Digital"]` |
| Mood | Purple `rgba(150,50,200,0.3)` | `[data-cat="Mood"]` |
| Flags | Light blue `rgba(100,200,255,0.3)` | `[data-cat="Flags"]` |
| Pop Culture | Hot pink `rgba(255,0,100,0.3)` | `[data-cat="Pop Culture"]` |
| Atmospheric | Blue `rgba(100,150,255,0.3)` | `[data-cat="Atmospheric"]` |
| Party | Magenta `rgba(255,50,255,0.3)` | `[data-cat="Party"]` |

Unknown categories fall back to `--accent-glow`.

### Theme Extensibility

To add a new theme:
1. Define a new set of CSS variable values.
2. Apply them via a class on `<body>` (same pattern as 1337 mode).
3. The entire UI will adapt — all components reference the variables.

Example: light theme could override `--bg: #f5f5f5; --text: #1a1a1a; --bg-card: #ffffff;`.

---

## 11. 1337 Mode (Easter Egg)

### Activation: Triple-Click the Japan Card

The `japan` card uses a dedicated click handler (instead of the plain `setEffect` handler) that **defers** the normal action so the first two clicks of a triple-click never flicker or switch effects:

1. Each click on the `japan` card increments a counter. The first click starts a `TRIPLE_CLICK_MS` (500 ms) timer.
2. Clicks 1 and 2 **do nothing visible** except add a subtle "pressed" highlight to the card. No API call, no preview change.
3. If a **third** click arrives before the timer fires: cancel the timer, reset the counter, and **toggle** 1337 mode. The current effect is **not** changed.
4. If the timer fires with 1 or 2 clicks counted: reset the counter and call `setEffect('japan')` exactly **once**.
5. Clicks beyond the third within the same window are ignored.

So a normal single click selects Japan after a ≤ 500 ms delay, a double-click also selects Japan (once), and a triple-click enters 1337 mode (or exits it if already active) without touching the lights. Keyboard users can activate the card with Enter/Space (counts as one click). No document-level capture listener or `stopPropagation()` is needed.

### Visual Changes

1. **Matrix rain background:** Full-viewport `<canvas id="matrix-bg">` with falling green katakana/code characters (`'01アイウエオカキクケコサシスセソ<>{}[]!@#$%^&*=+'`). Self-erasing via `rgba(0,0,0,0.05)` fill. Responsive to window resize. (This is a page background only — unrelated to the `matrix` / `matrix_rain` LED effects.) Respect `prefers-reduced-motion`: draw it static or at a very low frame rate.

2. **Theme override:** `body.leet-active` class overrides all CSS variables to green-on-black:
```css
body.leet-active {
    --bg: #0a0a0a; --bg-card: #0d1a0d; --bg-hover: #142014;
    --border: #1a3a1a; --text: #33ff33; --text-dim: #1a8a1a;
    --accent: #33ff33; --green: #33ff33; --red: #ff3333;
}
```

3. **Font change:** Header, status pill, footer, category headers switch to `'Courier New', monospace`.

4. **Header text:** Changes to "RGB L1GH75".

5. **Leet panel appears** (below the effect grid, `display: block`).

### Color Lab

| Element | Description |
|---------|-------------|
| Color preview | 60px tall swatch showing current color |
| Preset swatches | 10 circular buttons: Matrix (#33ff33), Red Alert (#ff0000), Cyan (#00ffff), Magenta (#ff00ff), Warning (#ffff00), Ember (#ff6600), Ice (#0066ff), White (#ffffff), Hot Pink (#ff0066), Ultraviolet (#9933ff) |
| Color picker | Native `<input type="color">` |
| Hex input | Text field, `#RRGGBB` format, displayed uppercase, synced with picker. Invalid input (anything not matching `^#[0-9a-fA-F]{6}$`) is highlighted red and never sent. |
| Apply button | "UPLOAD TO MAINFRAME" — sends `setEffect('solid_color', { color: hex.toLowerCase() })` |

All three inputs (presets, picker, hex) are kept in sync bidirectionally.

### System Diagnostics Panel

8 stats, updated every 1 second:

| Stat | Source | Calculation |
|------|--------|-------------|
| Server Uptime | **Real** | `Date.now()/1000 - serverBootTime` (from `/api/status` `server_boot` field) |
| Effect Runtime | **Real** | From existing uptime text in footer |
| Power Draw | **Estimate** | `litLedCount() × WATTS_PER_LED × currentBrightness` — e.g. 70 LEDs × 0.06 W × 1.0 = 4.2 W. `0` when off. LED count comes from `/api/info` (Section 8), never a hard-coded number. |
| Running Cost | **Estimate** | `Power Draw (kW) × server uptime (h) × PRICE_PER_KWH` |
| Photons Emitted | **Fake** | Random increment ~1-10 trillion/sec, displayed in scientific notation (`3.42e15`) |
| Neural Link | **Fake** | Oscillating `92 + 8 * sin(t*0.7) * sin(t*0.3)`, shown as `XX.X% SYNC` |
| Nodes Pwned | **Fake** | ~30% chance per second to increment by 1 |
| Hack Level | **Fake** | Time-based progression since entering 1337 mode: Script Kiddie (0m) → Novice (1m) → Apprentice (3m) → Journeyman (5m) → Elite (10m) → Legendary (20m) → 1337 G0D (60m) |

Photons and Neural Link rows get a CSS `pulse` animation (opacity oscillates).

### Leet Card Styling

Each card has a scanning green gradient line across the top:
```css
.leet-card::before {
    content: '';
    position: absolute; top: 0; left: 0; right: 0; height: 2px;
    background: linear-gradient(90deg, transparent, #33ff33, transparent);
    animation: leetScan 3s linear infinite;
}
```

### Exit

Three exit methods:
1. Click the red "[ ESC ] EXIT 1337 M0D3" button.
2. Press the Escape key.
3. Triple-click the `japan` card again.

On exit: remove `leet-active` class, stop matrix rain animation, clear stats interval, restore header text to "RGB Lights". The active LED effect is left unchanged.

### 1337 Mode State Variables

```javascript
let leetActive     = false;       // Is 1337 mode currently on?
let leetMatrixId   = null;        // requestAnimationFrame ID for matrix rain
let leetStatsId    = null;        // setInterval ID for stats updates
let leetEnteredAt  = null;        // Unix timestamp when mode was entered
let leetPhotons    = 0;           // Running photon counter
let leetNodesPwned = 0;           // Running nodes counter
let serverBootTime = null;        // From /api/status server_boot field
let currentColor   = '#33ff33';   // Current color in color lab
let japanClicks    = 0;           // Triple-click counter
let japanTimer     = null;        // Triple-click window timer
```

---

## 12. Error Handling

### OpenRGB Connection Failures

**In `rgb_effects.py`:**
- If the TCP connection to OpenRGB fails, print a clear error message to stderr and exit with code 1:
  ```
  ERROR: Could not connect to OpenRGB at <host>:<port> — Connection refused
  ```
- The web server detects the subprocess exit (via `poll()`) and sets `running: false`. It captures the last lines of the subprocess's stderr and logs them.
- The browser's next `pollStatus()` will show the OFF state.

**In `rgb_server.py` (`GET /api/info`):**
- If the `--info` subprocess times out (15 seconds), return HTTP 504:
  ```json
  {"error": "OpenRGB connection timed out"}
  ```
- If any other error occurs, return HTTP 502 with `{"error": "<short message>"}`. Never return stack traces to the client.

### Device or Zone Not Found

If `find_device()` returns `None`:
```
ERROR: Could not find RGB controller (looking for 'YOUR_DEVICE_NAME').
Available devices:
  Device 0: Some Other Device
```
Print available devices to help the user fix `DEVICE_NAME` / `RGB_DEVICE_NAME`, then exit with code 1. Do the same for a configured zone name that isn't found (list the zones that exist).

### Subprocess Management

- If the effect subprocess crashes, `get_status()` detects it on the next poll (`poll() is not None`) and updates state to `running: false`.
- If `terminate()` doesn't work within 5 seconds, escalate to `kill()`.
- On server shutdown (SIGINT/SIGTERM), always attempt to kill the subprocess before exiting.
- When stopping effects (`POST /api/off`), always run a separate `--off` subprocess to zero all LEDs, even if the effect subprocess already died.

### State File Problems

- Missing, unreadable, or corrupt `state.json` → log a warning and start with defaults.
- Unwritable state directory → log once and run without persistence.

### Frontend Error Handling

- `fetchJSON()` wraps all API calls in try/catch with an 8-second `AbortController` timeout. Returns `null` on any error (network, parse, timeout, non-2xx).
- `pollStatus()` silently skips if the fetch returns `null` — prevents UI flickering during transient network issues.
- Status pill shows "OFF" when the server reports `running: false`, regardless of reason.

### User-Friendly Error Scenarios

| Scenario | User Experience |
|----------|----------------|
| OpenRGB not running / unreachable | Status shows OFF. Effect clicks do nothing visible. `/api/info` returns a timeout error; preview uses the fallback ring size. |
| Wrong device name configured | Effect starts but subprocess immediately exits. Status shows OFF. Server log shows the available devices. |
| Wrong LED count configured | LEDs light up incorrectly (wrong colors in wrong positions, or part of the chain dark). User fixes the count and restarts. |
| Browser loses connection to server | Status poll fails silently. UI freezes at last known state. Resumes automatically when connection restores. |
| Effect subprocess crashes | Status shows OFF within 3 seconds (next poll). User can click another effect. |

---

## 13. Security

Free RGB has **no login** and the OpenRGB SDK protocol has **no authentication**. Anyone who can reach port 7777 can change your lights, and anyone who can reach port 6742 can control every RGB device OpenRGB manages. Treat both as trusted-network-only services. Generated code and compose files **must** follow these rules.

### Network Exposure

| Port | Service | Rule |
|------|---------|------|
| 7777 (`PORT`) | Web UI + API | Bind to `127.0.0.1` by default. Expose to the LAN only deliberately (`WEB_BIND`), and only on a trusted network. |
| 6742 | OpenRGB SDK | Never publish when both containers share a Docker network (`expose`, not `ports`). Elsewhere, listen on loopback, or on the narrowest interface that works (e.g. only reachable from the WSL2 subnet). |

- **Never expose either port to the internet.** No router port-forwards, no UPnP, no public cloud tunnels without authentication in front.
- To use the UI from a phone on your home network, either:
  - set `WEB_BIND` to the machine's LAN address **and** restrict the port with a host firewall to your LAN subnet, e.g. `sudo ufw allow from <your-lan-subnet> to any port 7777 proto tcp`; or
  - keep `WEB_BIND=127.0.0.1` and put a **reverse proxy with authentication** in front (Caddy, nginx, Traefik), e.g. with Caddy:
    ```
    rgb.home.arpa {
        tls internal
        basic_auth {
            <username> <bcrypt-hash-from-`caddy hash-password`>
        }
        reverse_proxy 127.0.0.1:7777
    }
    ```
- Remote access from outside the home should go through a VPN (e.g. WireGuard or a mesh VPN), not an open port.

### HTTP Hardening (required in `rgb_server.py`)

1. **CORS:** same-origin only by default; optional exact-match allowlist via `CORS_ORIGINS` (Section 6). Never send `Access-Control-Allow-Origin: *`.
2. **JSON-only POSTs:** `POST` endpoints require `Content-Type: application/json` (else `415`). Together with the CORS rule this stops other websites from silently driving your lights via form posts.
3. **Body size limit:** reject bodies over `MAX_BODY_BYTES` (4 KiB) with `413` before reading them; require `Content-Length` (`411`).
4. **Strict input validation:** effect name from the allowlist, zone from the allowlist, finite numbers clamped to their ranges, booleans must be booleans, `color` must match `^#[0-9a-fA-F]{6}$` exactly (Section 6). The same validation is applied to values loaded from `state.json`.
5. **Static files — path-traversal guard:** URL-decode the path, reject anything containing a NUL byte, then resolve it with `(STATIC_DIR / rel).resolve()` and serve it **only if** the result `is_relative_to(STATIC_DIR.resolve())` and is a regular file. Otherwise return `404`. No directory listings. Never serve files outside `static/` (e.g. `/static/../rgb_server.py` or `/static/%2e%2e/...` must 404).
6. **No shell:** subprocesses are spawned with an argument list; user input never reaches a shell.
7. **Security headers** on every response (Section 6), and no stack traces or internal paths in error responses.
8. **Frontend:** server-provided strings are inserted with `textContent`, never `innerHTML`.

### Containers and Privileges

- Only the `openrgb` container is privileged; `rgb-lights` runs unprivileged and needs no devices. Optionally run `rgb-lights` as a non-root user (`USER 1000` in `Dockerfile.lights`, with `/data` owned by that user).
- In native mode, run the web server as a normal user. Only OpenRGB needs elevated access (or its udev rules).
- Don't commit `.env` files or real hostnames/IPs if you publish your generated code.

---

## 14. Platform-Specific Setup

The repository contains only the spec. **In every mode, first generate the code**: hand `SPEC.md` plus your hardware details (Section 1) to your AI assistant and let it create the files in Section 3 inside your clone. Then follow the mode that fits you:

| Mode | OpenRGB runs | Web app runs | Start with |
|------|--------------|--------------|------------|
| A. Linux + Docker (primary) | `openrgb` container | `rgb-lights` container | `docker compose up -d --build` |
| B. Linux native (no Docker) | OpenRGB binary/AppImage on the host | `python3 rgb_server.py` | Section "Native Run" |
| C. Windows + WSL2 | OpenRGB on Windows | `rgb-lights` container in WSL2 | `docker compose -f docker-compose.wsl2.yml up -d --build` |
| D. Windows native (no Docker) | OpenRGB on Windows | `py rgb_server.py` on Windows | Section "Native Run" |

### A. Linux + Docker (Primary)

OpenRGB runs inside Docker with `privileged: true` for I2C/USB access.

```bash
# 1. Load kernel modules (Section 1, Step 3) — i2c-i801 for Intel, i2c-piix4 for AMD
sudo modprobe i2c-dev
sudo modprobe i2c-i801

# 2. Clone, then generate the code from SPEC.md with your AI assistant
git clone https://github.com/mrshaun13/free-rgb.git
cd free-rgb
#    ...AI generates docker-compose.yml, Dockerfiles, rgb_server.py, effects/, static/...

# 3. Optional: configure
cat > .env <<'ENV'
TZ=UTC
PORT=7777
WEB_BIND=127.0.0.1
HOSTNAME_LABEL=Free RGB
ENV

# 4. Build and start
docker compose up -d --build

# 5. Open in browser
xdg-open http://localhost:7777
```

**Verify OpenRGB detects hardware:**
```bash
docker logs openrgb                       # look for your controller being detected
docker exec rgb-lights python3 effects/rgb_effects.py --info --host openrgb
```

Make sure the desktop OpenRGB app (or any other OpenRGB instance) is **not** also running on the host — two instances fighting over SMBus cause flicker and failed detection.

### B / D. Native Run (No Docker)

Because the backend is stdlib-only Python, it runs directly on any OS with Python 3.11+.

**1. Start the OpenRGB SDK server on the same machine, listening on loopback only:**

```bash
# Linux (as root, or as your user after installing OpenRGB's udev rules)
openrgb --server --server-host 127.0.0.1 --server-port 6742
```

On Windows, start OpenRGB, open the **SDK Server** tab, set the host to `127.0.0.1` and port `6742`, and click **Start Server** (enable "Start Server" in Settings to make it automatic).

**2. Start the web server:**

```bash
# Linux / macOS
cd free-rgb
OPENRGB_HOST=127.0.0.1 HOSTNAME_LABEL="Free RGB" python3 rgb_server.py
```

```powershell
# Windows (PowerShell)
cd free-rgb
$env:OPENRGB_HOST = "127.0.0.1"
py rgb_server.py
```

- With no `STATE_DIR` set and no writable `/data`, state is stored in `~/.local/state/free-rgb/` (Linux/macOS, honoring `XDG_STATE_HOME`) or `%LOCALAPPDATA%\free-rgb\` (Windows). The chosen path is logged at startup. Set `STATE_DIR=./data` to keep it next to the code instead.
- The server binds `127.0.0.1:7777` by default. Set `BIND_HOST`/`PORT` to change it — read Section 13 before binding to a LAN address.

**3. Optional — run at boot on Linux with a systemd user service** (`~/.config/systemd/user/free-rgb.service`):

```ini
[Unit]
Description=Free RGB web server
After=network-online.target

[Service]
WorkingDirectory=%h/free-rgb
Environment=OPENRGB_HOST=127.0.0.1
ExecStart=/usr/bin/python3 rgb_server.py
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now free-rgb
loginctl enable-linger "$USER"   # keep it running when you're logged out
```

OpenRGB itself also needs to start at boot (its own systemd service or autostart entry).

### C. Windows + WSL2

OpenRGB needs direct SMBus/USB hardware access, which WSL2 does not provide, so OpenRGB runs on **Windows** and only the web app runs in Docker inside WSL2.

**1. Clone inside the WSL2 filesystem** (e.g. `~/free-rgb`, not `/mnt/c/...`) for speed and correct line endings, then generate the code from `SPEC.md` with your AI assistant. `.gitattributes` keeps files LF.

**2. Run OpenRGB on Windows with the SDK server enabled:**
- Launch OpenRGB (as Administrator for SMBus devices) and verify your devices are detected.
- Open the **SDK Server** tab, set the port to `6742`, set the host according to the networking option below, and click **Start Server**. Enable "Start Server" (and optionally "Start at Login" / "Start Minimized") in **Settings** so it survives reboots. Exact labels can differ between OpenRGB versions.

**3. Make OpenRGB reachable from the container.** Pick the option that matches your setup:

| Your setup | `OPENRGB_HOST` | OpenRGB SDK host on Windows | Windows firewall |
|------------|----------------|-----------------------------|------------------|
| **Docker Desktop** (WSL2 backend) | `host.docker.internal` (the default in `docker-compose.wsl2.yml`) | `127.0.0.1` usually works; use `0.0.0.0` if the connection is refused | Allow inbound TCP 6742 only from the WSL/Docker subnet if you use `0.0.0.0` |
| **Docker Engine inside WSL2, default NAT networking** | The Windows host IP as seen from WSL: `ip route show default \| awk '{print $3}'` (put it in `.env` as `OPENRGB_HOST=...`; it can change after a reboot) | `0.0.0.0` — the SDK server must listen on all interfaces, because WSL traffic arrives on the WSL virtual adapter, not loopback | **Required:** allow inbound TCP 6742 from the WSL subnet only (below) |
| **Docker Engine inside WSL2, mirrored networking** (`networkingMode=mirrored` in `%UserProfile%\.wslconfig`, Windows 11 22H2+) | `127.0.0.1`, and add `network_mode: host` to the `rgb-lights` service (remove its `ports:` and `extra_hosts:`; set `BIND_HOST=127.0.0.1` or a LAN address) | `127.0.0.1` | Usually none needed for loopback |

Firewall rule for the NAT case (run in an **elevated** PowerShell). Find the WSL subnet from inside WSL with `ip -4 addr show eth0` (e.g. an address ending in `/20`) and use that network, not your whole LAN:

```powershell
New-NetFirewallRule -DisplayName "OpenRGB SDK (WSL only)" -Direction Inbound `
  -Protocol TCP -LocalPort 6742 -RemoteAddress <wsl-subnet-cidr> -Action Allow
```

Never allow 6742 from "Any" address — that would let any device on your network control your RGB hardware.

**4. Start the web app** (uses the standalone file, which has no `openrgb` service and no `depends_on`):

```bash
docker compose -f docker-compose.wsl2.yml up -d --build
docker exec rgb-lights python3 effects/rgb_effects.py --info   # should list your device and zones
```

**5. Open in browser:** `http://localhost:7777` (WSL2 forwards localhost to Windows).

**Important:** OpenRGB on Windows must be running with the SDK server started before effects will work. If it isn't, the UI loads but shows OFF and `/api/info` times out.

Alternatively, skip WSL2 entirely and use **mode D** (native Python on Windows) — it is the simplest Windows setup.

---

## 15. Extension Points

### Adding a New Effect

Code changes go in three files, plus the docs:

1. **Python (`effects/rgb_effects.py`):**
   - Create a new class extending `Effect`.
   - Implement `render(self, t, num_leds)`.
   - Optionally implement `render_static(num_leds)` if the effect has a meaningful static state.
   - Add an instance to the `ALL_EFFECTS` dict.

2. **Server (`rgb_server.py`):**
   - Add an entry to the `EFFECTS` list with `name`, `description`, and `category`.

3. **Frontend (`static/index.html`):**
   - Add a matching renderer function to the `renderers` object (same math as Python).
   - If it has a static mode, add a static renderer to `staticRenderers`.
   - If it's a flag, add its name to the `FLAG_EFFECTS` Set.
   - If it introduces a new category, add a category accent CSS rule (below).

4. **Docs:** add it to the catalog in Section 9 and update the counts (Section 9 table, Section 6, README.md).

The grid, cards, canvas, controls, validation, and API all adapt automatically.

### Adding a New Category

1. Add effects with the new category name in the `EFFECTS` list.
2. Add a CSS accent color rule:
   ```css
   .effect-card[data-cat="New Category"] .card-preview { box-shadow: 0 0 12px rgba(R,G,B,0.3); }
   ```
3. The grid auto-groups by category name. Add the category to the tables in Sections 9 and 10.

### Adding a New Theme

1. Define a CSS class with variable overrides:
   ```css
   body.my-theme {
       --bg: #fff; --text: #333; --bg-card: #f5f5f5;
       /* ... override all variables */
   }
   ```
2. Add a UI control (button, dropdown) to toggle the class on `<body>`.
3. The entire UI adapts immediately — every component uses the variables.

### Adding New Control Parameters

1. Add the parameter to the `DEFAULTS` dict in `rgb_server.py`.
2. Add it to `_state`, `_save_state()`, `load_state()`, `start_effect()`, and the `POST /api/effect` handler — **including validation** (Section 13).
3. Add the CLI argument in `rgb_effects.py`.
4. Pass it through to `run_effect()`.
5. Add a UI control in `index.html` (slider, toggle, etc.).
6. Include it in the `setEffect()` POST body and `pollStatus()` sync logic.
7. Document it in Sections 4, 6, and 7.

### Future Enhancements (Ideas)

- **Effect favorites / custom ordering**
- **Effect scheduling** (time-of-day presets)
- **Music sync** (microphone input → beat detection → effect parameters)
- **Multi-device support** (control multiple OpenRGB controllers)
- **WebSocket** for real-time status (replace polling)
- **PWA support** (installable on phone home screen)
- **Effect parameter customization** (per-effect speed, color palette overrides)
- **Optional built-in auth** (token or password) for users who can't run a reverse proxy
