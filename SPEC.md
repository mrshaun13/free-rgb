# Free RGB — Complete Build Specification

This document is the authoritative specification for building the Free RGB lighting control system. It is designed to be handed to an AI coding assistant along with the user's hardware details. The AI should produce a complete, working, production-quality codebase by following this spec precisely.

> **To the AI reading this:** This spec is detailed and opinionated on purpose. Follow it closely. The architecture decisions, API shapes, file structure, and patterns described here have been validated in production. When the spec says "must", it means "must".

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Architecture Overview](#2-architecture-overview)
3. [Folder Structure](#3-folder-structure)
4. [Environment Variables](#4-environment-variables)
5. [Docker Configuration](#5-docker-configuration)
6. [Backend: Web Server (rgb_server.py)](#6-backend-web-server)
7. [Backend: Effects Engine (rgb_effects.py)](#7-backend-effects-engine)
8. [Frontend: Single-Page Application (index.html)](#8-frontend-single-page-application)
9. [Effect Catalog](#9-effect-catalog)
10. [Theme System](#10-theme-system)
11. [1337 Mode (Easter Egg)](#11-1337-mode-easter-egg)
12. [Error Handling](#12-error-handling)
13. [Platform-Specific Setup](#13-platform-specific-setup)
14. [Extension Points](#14-extension-points)

---

## 1. Prerequisites

### Step 0: Compatibility Check (MANDATORY)

Before writing any code, the user **must** confirm that OpenRGB can detect and control their RGB devices. This is not optional. If OpenRGB cannot see the hardware, nothing else in this project will work.

**Instructions for the user:**

1. Download and install [OpenRGB](https://openrgb.org/releases.html) for your platform.
2. Launch OpenRGB with administrative privileges (required for SMBus/I2C access).
3. Go to the **Devices** tab.
4. Confirm your motherboard controller (or other RGB controllers) appear in the device list.
5. Click on a device, manually change a color, and verify the physical LEDs respond.
6. Go to the **Information** tab — note down:
   - Device name(s) (e.g., "Gigabyte Z490 GAMING X AX", "ASUS Aura DRAM")
   - Zone name(s) and LED counts per zone (e.g., "D_LED1 Bottom: 24 LEDs", "D_LED2 Top: 64 LEDs")
   - Total LED count

**If this fails:** Check the [OpenRGB supported devices list](https://openrgb.org/devices.html), ensure required kernel modules are loaded (Linux), and consult the [OpenRGB FAQ](https://openrgb.org/faq.html).

### Step 1: Gather Hardware Information

The user must provide the AI with:

| Information | Example | Why It's Needed |
|-------------|---------|-----------------|
| Motherboard or controller name | "Gigabyte Z490 GAMING X AX" | Device discovery string matching |
| Zone names and LED counts | "D_LED1 Bottom: 24, D_LED2 Top: 64" | Zone sizing and effect rendering |
| Physical layout description | "6 fans on bottom hub, 3 AIO fans + pump on top" | Pump-head-aware effects, zone targeting |
| Operating system | "Ubuntu 24.04" or "Windows 11 + WSL2" | Platform-specific Docker and OpenRGB setup |

### Step 2: Software Requirements

| Requirement | Version | Notes |
|-------------|---------|-------|
| Docker + Docker Compose | 24.x+ / 2.x+ | Container runtime |
| OpenRGB | 0.9+ (1.0rc2 recommended) | Must be able to detect user's hardware |
| Python | 3.11+ | Runs inside Docker — no host install needed |
| Git | Any | Clone the repo |
| Web browser | Any modern | Chrome, Firefox, Safari, Edge |

### Step 3: Linux Kernel Modules (Linux only)

For motherboard RGB control via SMBus/I2C, these kernel modules must be loaded on the **host** (not inside Docker):

```bash
# Load immediately
sudo modprobe i2c-dev
sudo modprobe i2c-i801    # Intel — check OpenRGB docs for your chipset

# Persist across reboots
echo "i2c-dev" | sudo tee /etc/modules-load.d/i2c-dev.conf
echo "i2c-i801" | sudo tee /etc/modules-load.d/i2c-i801.conf
```

The specific I2C driver module depends on the chipset:
- **Intel**: `i2c-i801` (most desktop boards)
- **AMD**: `i2c-piix4`
- Check OpenRGB documentation for your specific platform.

---

## 2. Architecture Overview

### Two-Container Model

```
┌─────────────────────────────────────────────────────────────┐
│  Docker Network: rgb-net                                    │
│                                                             │
│  ┌──────────────┐    TCP:6742    ┌────────────────────┐    │
│  │   OpenRGB     │◀─────────────│    rgb-lights        │    │
│  │   SDK Server  │              │    (Web Server)       │    │
│  │   Container   │              │                       │    │
│  │               │              │  rgb_server.py        │    │
│  │  Port: 6742   │              │    ├── spawns ──▶     │    │
│  │  Privileged   │              │    │  rgb_effects.py  │    │
│  └──────────────┘              │    │  (subprocess)    │    │
│                                 │    │                  │    │
│                                 │  Port: 7777          │    │
│                                 └────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
          │                                  │
     /dev/i2c-*                        http://localhost:7777
     (hardware)                            (browser)
```

### Key Design Decisions

1. **OpenRGB runs in its own container** — privileged, with access to I2C devices. The web app never touches hardware directly.
2. **The web server communicates with OpenRGB via the SDK protocol** (raw TCP on port 6742) — not via CLI, not via REST, not via direct hardware access.
3. **Effects run as subprocesses** — the web server spawns `rgb_effects.py` as a child process with CLI arguments. This isolates crashes, simplifies state management, and makes it trivial to kill/restart effects.
4. **Zero external Python dependencies** — the entire backend uses only Python stdlib. The OpenRGB SDK protocol is implemented from scratch using raw sockets. No `pip install` required.
5. **Single-file frontend** — `index.html` contains all HTML, CSS, and JavaScript inline. No build tools, no npm, no bundler, no framework. Vanilla JS only.
6. **State persistence** — the last-active effect, zone, speed, brightness, and color are saved to a JSON file and auto-restored on container restart.

### Request Flow

1. User opens `http://localhost:7777` in browser.
2. Browser fetches `GET /api/effects` → receives the full effect catalog.
3. Browser renders effect grid with animated mini-previews (canvas-based).
4. User clicks an effect → browser sends `POST /api/effect { effect, zone, speed, brightness, static, color }`.
5. `rgb_server.py` receives the request, kills any existing effect subprocess, and spawns a new one:
   ```
   python3 rgb_effects.py --effect fire --zone both --speed 1.5 --brightness 0.8 --host openrgb --port 6742
   ```
6. The subprocess connects to OpenRGB SDK server via TCP, discovers the device, resizes zones, and enters a render loop at 30 FPS.
7. Each frame: `effect.render(t, num_leds)` → build color array → send `UPDATELEDS` packet to OpenRGB.
8. Browser polls `GET /api/status` every 3 seconds to stay synced.

---

## 3. Folder Structure

```
free-rgb/
├── docker-compose.yml              # Two-service stack definition
├── Dockerfile                      # OpenRGB SDK server container
├── Dockerfile.lights               # Web app container
├── rgb_server.py                   # Web server + subprocess manager
├── effects/
│   └── rgb_effects.py              # Effects engine + OpenRGB SDK protocol
├── static/
│   └── index.html                  # Complete SPA (HTML + CSS + JS, single file)
├── SPEC.md                         # This file
├── CONTRIBUTING.md                 # Contributor guide
├── README.md                       # Project overview
└── LICENSE                         # MIT
```

**Why this structure:**
- Flat top level — no unnecessary nesting. The two Python files and the HTML file are the entire application.
- `effects/` is a directory to allow future modularization (splitting effects into separate files).
- `static/` follows the convention of static file serving and keeps the HTML separate from Python.
- Docker files at the root — standard convention, easy to find.

---

## 4. Environment Variables

All configuration is via environment variables. **Nothing is hard-coded** except sensible defaults.

### Web Server (`rgb_server.py`)

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENRGB_HOST` | `localhost` | Hostname of the OpenRGB SDK server |
| `OPENRGB_PORT` | `6742` | Port of the OpenRGB SDK server |
| `STATE_DIR` | `/data` | Directory for persistent state file (`state.json`) |
| `PORT` | `7777` | Web server listen port (set in code, override via code change) |

### Docker Compose

| Variable | Set On | Value | Purpose |
|----------|--------|-------|---------|
| `TZ` | Both containers | User's timezone (e.g., `America/Chicago`) | Correct timestamps in logs |
| `OPENRGB_HOST` | `rgb-lights` | `openrgb` (container hostname) | Web server → OpenRGB connection |
| `OPENRGB_PORT` | `rgb-lights` | `6742` | SDK server port |
| `STATE_DIR` | `rgb-lights` | `/data` | State persistence mount point |

---

## 5. Docker Configuration

### `docker-compose.yml`

```yaml
services:
  openrgb:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: openrgb
    restart: unless-stopped
    privileged: true                    # Required for I2C/SMBus hardware access
    environment:
      - TZ=${TZ:-America/Chicago}
    volumes:
      - ./openrgb-config:/root/.config/OpenRGB    # Persistent OpenRGB config
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "6742:6742"
    networks:
      - rgb-net

  rgb-lights:
    build:
      context: .
      dockerfile: Dockerfile.lights
    container_name: rgb-lights
    restart: unless-stopped
    environment:
      - TZ=${TZ:-America/Chicago}
      - OPENRGB_HOST=openrgb
      - OPENRGB_PORT=6742
      - STATE_DIR=/data
    volumes:
      - rgb-state:/data                           # Persistent effect state
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "7777:7777"
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
- The `openrgb` container **must** be privileged for I2C/SMBus hardware access.
- The `openrgb-config` volume persists OpenRGB's device profiles between restarts.
- The `rgb-state` named volume persists the last-active effect state.
- Both containers share the `rgb-net` network so `rgb-lights` can reach `openrgb` by container hostname.

### `Dockerfile` (OpenRGB SDK Server)

```dockerfile
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates wget \
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
CMD ["--server", "--server-host", "0.0.0.0", "--server-port", "6742"]
```

**Notes:**
- Check [OpenRGB releases](https://codeberg.org/OpenRGB/OpenRGB/releases) for the latest `.deb` URL and update `OPENRGB_VERSION` and `OPENRGB_HASH` accordingly.
- Qt5 libraries are required even for headless/server mode.

### `Dockerfile.lights` (Web App)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY rgb_server.py .
COPY effects/ ./effects/
COPY static/ ./static/
EXPOSE 7777
CMD ["python3", "rgb_server.py"]
```

**Zero pip dependencies.** The entire backend is pure Python stdlib.

---

## 6. Backend: Web Server

### File: `rgb_server.py`

**Role:** HTTP server that serves the web UI, exposes a REST API, and manages `rgb_effects.py` as a subprocess.

**Dependencies:** Python stdlib only — `http.server`, `subprocess`, `json`, `signal`, `threading`, `pathlib`, `time`, `os`, `sys`.

### Configuration Constants

```python
PORT           = 7777
OPENRGB_HOST   = os.environ.get("OPENRGB_HOST", "localhost")
OPENRGB_PORT   = os.environ.get("OPENRGB_PORT", "6742")
STATE_FILE     = Path(os.environ.get("STATE_DIR", "/data")) / "state.json"
EFFECTS_SCRIPT = Path(__file__).parent / "effects" / "rgb_effects.py"

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
    # ... one entry per visible effect
]

EFFECT_NAMES = {e["name"] for e in EFFECTS}
# Hidden effects (accepted by API but not shown in UI):
EFFECT_NAMES.add("solid_color")
```

**8 categories, 47 visible effects + 1 hidden.** See [Section 9: Effect Catalog](#9-effect-catalog) for the complete list.

### EffectManager Class

Singleton process manager with thread-safe state.

**State shape:**
```python
{
    "running": bool,        # Is an effect subprocess alive?
    "effect": str | None,   # Current effect name
    "zone": str,            # "both", "bottom", or "top"
    "speed": float,         # Animation speed multiplier
    "brightness": float,    # Global brightness 0.0–1.0
    "static": bool,         # Static rendering mode (flags only)
    "color": str | None,    # Hex color for solid_color (e.g., "#ff0000")
}
```

Plus internal tracking:
- `_started_at` — `time.monotonic()` when the current effect was started (for uptime display).
- `_server_boot` — `time.time()` when the server process started (wall clock, for 1337 mode stats).

**Methods:**

| Method | Behavior |
|--------|----------|
| `load_state()` | On startup, reads `STATE_FILE`. If a valid effect is persisted, auto-starts it. |
| `_save_state()` | Writes current state to `STATE_FILE` as JSON. Called after every state change. |
| `start_effect(effect, zone, speed, brightness, static, color)` | Acquires lock. Kills existing subprocess. Builds CLI command. Spawns new subprocess. Updates state. Saves state. |
| `stop()` | Acquires lock. Kills subprocess. Sets `running=False, effect=None`. Saves state. Runs a **separate synchronous subprocess** with `--off` flag to explicitly zero all LEDs. |
| `_stop_subprocess()` | Graceful kill: `terminate()` → wait 5s → `kill()` → wait 3s. |
| `get_status()` | Returns state dict plus computed `uptime` string and `server_boot` timestamp. Checks if process died unexpectedly. |
| `shutdown()` | Called on SIGINT/SIGTERM. Kills subprocess. |

**Subprocess command construction:**
```python
cmd = [
    sys.executable, str(EFFECTS_SCRIPT),
    "--effect", effect,
    "--zone", zone,
    "--speed", str(speed),
    "--brightness", str(max(0.0, min(1.0, brightness))),
    "--host", OPENRGB_HOST,
    "--port", OPENRGB_PORT,
]
if static:
    cmd.append("--static")
if color:
    cmd.extend(["--color", color])
```

### HTTP API

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Serve `static/index.html` |
| `GET` | `/static/*` | Serve static assets (MIME auto-detected) |
| `GET` | `/api/status` | Current effect state + uptime + server boot time |
| `GET` | `/api/effects` | Full effect catalog (array of `{name, description, category}`) |
| `POST` | `/api/effect` | Switch to a new effect |
| `POST` | `/api/off` | Stop current effect, turn all LEDs off |
| `GET` | `/api/info` | Run `rgb_effects.py --info` and return device/zone info |
| `OPTIONS` | `*` | CORS preflight (204 No Content) |

**All responses include CORS headers:** `Access-Control-Allow-Origin: *`

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
  "server_boot": 1777015554.578
}
```

- `uptime`: Human-readable string. `null` when not running. Format: `"Xh Ym"` if hours > 0, else `"Ym"`.
- `server_boot`: Unix timestamp (float) of when the server process started. Used by the frontend for real uptime calculations.

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

**Validation:**
- `effect`: Required. Must be in `EFFECT_NAMES`. Return 400 if unknown.
- `zone`: Optional. Must be `"both"`, `"bottom"`, or `"top"`. Defaults to `"both"`.
- `speed`: Optional. Float, clamped to `[0.1, 5.0]`. Defaults to `1.0`.
- `brightness`: Optional. Float, clamped to `[0.0, 1.0]`. Defaults to `1.0`.
- `static`: Optional. Boolean. Defaults to `false`.
- `color`: Optional. Hex string like `"#ff0000"` or `null`. Only meaningful for `solid_color` effect.

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

#### `GET /api/info` Response

```json
{
  "output": "Device 0: Gigabyte Z490 GAMING X AX\n  Zone 0: D_LED1 Bottom (24 LEDs)\n  Zone 1: D_LED2 Top (64 LEDs)\n..."
}
```

Or on error:
```json
{"error": "OpenRGB connection timed out"}
```

### Entry Point

```python
def main():
    signal.signal(signal.SIGINT, on_signal)
    signal.signal(signal.SIGTERM, on_signal)
    manager.load_state()        # Auto-restore last effect
    server = HTTPServer(("0.0.0.0", PORT), RGBHandler)
    server.serve_forever()
```

---

## 7. Backend: Effects Engine

### File: `effects/rgb_effects.py`

**Role:** Standalone CLI tool that connects to OpenRGB SDK server, renders effects in a loop, and pushes LED color updates. Spawned as a subprocess by `rgb_server.py`.

**Dependencies:** Python stdlib only — `socket`, `struct`, `math`, `time`, `signal`, `argparse`, `sys`, `abc`, `dataclasses`, `random`. **No pip packages.**

### OpenRGB SDK Protocol Implementation

The effects engine implements the OpenRGB SDK binary protocol from scratch using raw TCP sockets.

**Protocol basics:**
- Magic: `b"ORGB"` (4 bytes)
- Header: magic (4) + device_index (4) + packet_id (4) + data_length (4) = 16 bytes
- All integers are little-endian unsigned 32-bit.
- Strings are null-terminated and prefixed with a 2-byte length (including the null byte).

**Required packet types:**

| Packet ID | Name | Direction | Purpose |
|-----------|------|-----------|---------|
| 0 | `REQUEST_CONTROLLER_COUNT` | Client → Server | Get number of detected devices |
| 1 | `REQUEST_CONTROLLER_DATA` | Client → Server | Get full device info (zones, LEDs) |
| 50 | `SET_CLIENT_NAME` | Client → Server | Identify the client |
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

**Device data parsing strategy:** The binary format for controller data is complex and varies between OpenRGB versions. Rather than implementing a full parser with exact byte offsets, **search for known zone name strings in the raw binary data** and extract zone information from their surrounding context. This is more robust across different hardware configurations.

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
        """Find zone by name (substring match)."""
```

### Hardware Constants (USER MUST CUSTOMIZE)

These constants must be set based on the user's hardware (from the prerequisite step):

```python
# Device discovery — partial string match against OpenRGB device names
MOBO_DEVICE_NAME = "YOUR_MOTHERBOARD_NAME_HERE"

# Zone names — must match what OpenRGB reports for your hardware
ZONE_BOTTOM = "YOUR_BOTTOM_ZONE_NAME"    # e.g., "D_LED1 Bottom"
ZONE_TOP    = "YOUR_TOP_ZONE_NAME"       # e.g., "D_LED2 Top"

# LED counts per zone — must match your physical hardware
ZONE_BOTTOM_LED_COUNT = 24    # Adjust to your setup
ZONE_TOP_LED_COUNT    = 64    # Adjust to your setup
```

> **AI Implementation Note:** When the user provides their hardware details (from the prerequisite step), substitute these values. If the user has only one zone, the "top" zone can be omitted and zone targeting simplified.

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
            num_leds: Number of LEDs in the target zone.

        Returns:
            List of (R, G, B) tuples, one per LED.
        """

    def render_static(self, num_leds: int) -> Optional[List[Tuple[int, int, int]]]:
        """
        Render a static (non-animated) frame. Used for flag effects.
        Returns None if this effect doesn't support static mode (falls back to animated).
        """
        return None
```

### Effect Implementation Patterns

**Simple ambient effect (e.g., Fire):**
```python
class Fire(Effect):
    name = "fire"
    description = "Roaring fire — red/orange/yellow flames flickering"

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
    description = "Irish tricolor — green, white & orange flowing"
    C1, C2, C3 = (0, 155, 72), (255, 255, 255), (255, 130, 0)
```

**Pump-head-aware effect (for AIO coolers):**
```python
# If the user has an AIO cooler with a visible pump head, effects can
# treat the last N LEDs of the top zone as the "pump head" (center circle).
# Detect this by checking if num_leds >= some threshold (e.g., 48).
# The pump head typically has 8-10 LEDs and sits physically in the center.
```

**SolidColor (hidden, for 1337 mode):**
```python
class SolidColor(Effect):
    name = "solid_color"
    description = "Solid custom color"

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
    # ... all 47 visible effects + solid_color
    "solid_color": SolidColor(),    # Default white, overridden at runtime by --color
}
```

### Zone Management

**`find_mobo_device(conn)`** — Scan all devices returned by OpenRGB. Match device name using substring match against `MOBO_DEVICE_NAME`. Return the first match.

**`ensure_zones_sized(conn, mobo, zone_top_led_count)`** — For addressable LED headers, OpenRGB needs to know how many LEDs are connected. Compare current zone sizes to expected counts. If different, call `resize_zone()`. After any resize, **re-fetch controller data** (the offsets change after a resize).

### Run Effect Function

```python
def run_effect(effect, conn, mobo, zone="both", brightness=1.0, fps=30, speed=1.0, static=False):
```

**Signal handling:** Register `SIGTERM`/`SIGINT` handlers that set `running = False` to break the render loop. On exit, always turn LEDs off and close the connection.

**Static mode:**
1. Call `effect.render_static(zone.led_count)`. If returns `None`, fall back to animated.
2. Build full-device color array (all LEDs). Non-target zones stay black `(0,0,0)`.
3. For each target zone: write static colors at correct LED offsets. Apply brightness.
4. Send one `update_device_leds()` call.
5. Sleep in a loop (`time.sleep(0.5)`) until killed.

**Animated mode:**
1. Calculate frame timing: `frame_time = 1.0 / fps`.
2. Loop while `running`:
   - `t = (time.monotonic() - start_time) * speed`
   - Build full-device color array (all LEDs). Non-target zones stay black.
   - For each target zone: `colors = effect.render(t, zone.led_count)`, apply brightness, write at correct offsets.
   - Send `update_device_leds()`.
   - Sleep for remaining frame time.
3. On exit: LEDs off, connection closed.

**Zone targeting (`--zone` flag):**
- `"both"`: Render effect on all zones.
- `"bottom"`: Only render on the bottom zone. Top zone stays black.
- `"top"`: Only render on the top zone. Bottom zone stays black.

### CLI Arguments

```
rgb_effects.py [options]

Options:
  --list                        List all available effects
  --info                        Show detected device and zone layout
  --probe                       Interactively discover LED counts
  --effect NAME, -e NAME        Effect name to run
  --off                         Turn all LEDs off
  --zone {both,bottom,top}      Target zone(s) (default: both)
  --host HOST                   OpenRGB server host (default: localhost)
  --port PORT                   OpenRGB server port (default: 6742)
  --speed FLOAT                 Animation speed multiplier (default: 1.0)
  --brightness FLOAT, -b FLOAT  Global brightness 0.0–1.0 (default: 1.0)
  --static                      Use static (non-animated) rendering
  --color HEX                   Hex color for solid_color (e.g., "#ff0000")
  --fps INT                     Target FPS (default: 30)
  --d2leds INT                  Override top zone LED count
```

**`--color` handling in `main()`:** When `--effect solid_color` is used with `--color`, parse the hex string and create a **new** `SolidColor(color=(r,g,b))` instance, replacing the default from the registry.

---

## 8. Frontend: Single-Page Application

### File: `static/index.html`

**One file. Everything inline.** No external CSS, no external JS, no build step, no framework.

Wrapped in an IIFE: `(function() { 'use strict'; ... })();`

### Constants

```javascript
const API      = window.location.origin;
const POLL_MS  = 3000;      // Status poll interval (milliseconds)
const NUM_LEDS = 24;        // Preview LED count for ring
```

### State Variables

```javascript
let effects       = [];        // Array from /api/effects
let currentEffect = null;      // Server-confirmed active effect
let currentZone   = 'both';
let currentSpeed  = 1.0;
let currentBright = 1.0;
let currentStatic = false;
let selectedEffect = null;     // What the user clicked (may differ during transition)
let cardCanvases  = {};        // Map: effectName → {canvas, ctx, animId}
let ringCtx       = null;      // Main ring canvas context
let ringAnimId    = null;      // Main ring animation frame ID
```

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
      <!-- Zone: Both / Bottom / Top -->
      <!-- Speed: slider 0.25–3.0, step 0.25 -->
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
      srobbins-server · OpenRGB · <span id="uptime-text"></span>
    </footer>
  </div>
</body>
```

### Canvas Drawing System

Two drawing functions, both using arc segments to form a ring:

**`drawRing(ctx, w, h, colors, innerRatio=0.65)`** — Main preview ring.
- Draw `colors.length` arc segments arranged in a circle.
- Gap between segments: `0.015` radians.
- Outer radius: `min(w, h) / 2 - 4`.
- Inner radius: `outer * innerRatio`.
- Fill each segment with its color using `arc()` and `fill()`.

**`drawMiniRing(ctx, w, h, colors)`** — Card preview mini-ring (72x72px).
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
- Returns: Array of `[R, G, B]` arrays.

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
    // 1. If solid_color: parse hex, render with subtle breathing, draw, recurse.
    // 2. If static mode + static renderer exists: render static, draw, recurse.
    // 3. Otherwise: t = (now - t0) * currentSpeed, render animated, apply brightness, draw, recurse.
    ringAnimId = requestAnimationFrame(animateRing);
}

function animateCard(name) {
    // Render at 16 LEDs, speed=1 (always), draw mini ring.
    info.animId = requestAnimationFrame(() => animateCard(name));
}
```

**All card canvases animate simultaneously** — each gets its own `requestAnimationFrame` loop started when the UI builds.

### Effect Grid Construction

On init:
1. Fetch `GET /api/effects`.
2. Group effects by `category`.
3. For each category: create a header + CSS grid.
4. For each effect: create a card with:
   - A 72×72 `<canvas>` for the mini-preview.
   - The effect name (underscores replaced with spaces).
   - The effect description.
   - A click handler that calls `setEffect(name)`.
   - A `data-cat` attribute for category-specific accent colors.
5. Start `animateCard()` for each card.

### Controls

**Zone buttons:** Three buttons (`Both`, `Bottom`, `Top`). Click → set `currentZone`, highlight active, re-send current effect to API.

**Speed slider:** `min=0.25, max=3, step=0.25`. `input` event updates display label. `change` event re-sends effect to API.

**Brightness slider:** `min=0, max=1, step=0.05`. Same event pattern as speed.

**Mode toggle (Animated/Static):** Two buttons. **Only visible when a flag effect is selected.** Managed by `FLAG_EFFECTS` Set and `updateModeToggle()` function. Non-flag effects force `currentStatic = false`.

```javascript
const FLAG_EFFECTS = new Set([
    'american_flag', 'american_flag_smooth', 'irish_flag', 'italy',
    'france', 'germany', 'ukraine', 'japan', 'bangladesh', 'brazil',
    'jamaica', 'india', 'south_korea', 'canada'
]);
```

**OFF button:** Calls `POST /api/off`, clears `selectedEffect`.

### Status Polling

```javascript
async function pollStatus() {
    const s = await fetchJSON('/api/status');
    // Update status pill (green ON / dim OFF)
    // For solid_color: show "SOLID #FF0000" in pill
    // Capture server_boot timestamp
    // Sync local state from server (zone, speed, brightness, static, effect)
    //   BUT only if user is not actively clicking (check :active pseudo-class)
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
Mobile: 2-column grid, stacked controls, smaller preview ring.

---

## 9. Effect Catalog

### 8 Categories, 47 Visible Effects

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
| `matrix` | Green code dripping down | 8 independent falling drops with bright heads, quadratic-falloff green tails (3-6 LEDs), random glitch flickers. Speeds 1.5-4x with golden-ratio offsets. |
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
| `japan` | White field with red center | White with red center segment (pump-head-aware) |
| `bangladesh` | Green field with red circle | Green with red center (pump-head-aware) |
| `brazil` | Green with golden diamond | Green field, yellow center pulse |
| `jamaica` | Green, black & gold X | Diagonal pattern |
| `india` | Saffron, white & green with blue | Three segments with blue center |
| `south_korea` | Red & blue yin-yang on white | White with red/blue center swirl |
| `canada` | Red-white-red with maple leaf | Red ends, white center with red center accent |

**Tricolor flag base class:** `ireland`, `italy`, `france`, `germany` share a `_TricolorFlag` base class. Subclasses only override three color constants (`C1`, `C2`, `C3`).

#### Pop Culture (10 effects)

| Name | Description |
|------|-------------|
| `matrix_rain` | Coordinated falling code columns with bright heads |
| `cyberpunk` | Hot pink & electric cyan with glitch flashes |
| `synthwave` | 80s sunset with purple, pink & orange |
| `portal` | Orange & blue spinning portals on opposite sides |
| `tron` | Light cycle traces blazing across the grid |
| `lightsaber` | Blade ignites and hums, cycling colors |
| `arc_reactor` | Pulsing blue-white energy from center (pump-head-aware) |
| `stranger_things` | Erratic Christmas lights in the Upside Down |
| `pac_man` | Yellow chomp chasing colorful ghosts |
| `limelight` | Mirrored L shapes in orange & lime |

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
| `disco` | Rapid colorful party lights with strobe flashes |
| `police` | Alternating red & blue with rapid flash |
| `heartbeat` | Red cardiac pulse, lub-dub rhythm |
| `halloween` | Spooky orange & purple with eerie flicker |
| `christmas` | Red & green with twinkling white star lights |

#### Hidden (1 effect)

| Name | Description | Access |
|------|-------------|--------|
| `solid_color` | Solid custom color with subtle breathing | 1337 mode only (not in effect grid) |

### JavaScript Renderers

**Every effect in the catalog must have a corresponding JavaScript renderer** that produces visually identical output to the Python version. The renderer is used for:
1. The main LED ring preview (440×440 canvas).
2. The mini-preview on each effect card (72×72 canvas).

Both Python `render()` and JS `renderer()` must use the same math, same color values, same variable names where possible. The only difference is language syntax.

Flag effects also need **static renderers** in the `staticRenderers` object.

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

### Theme Extensibility

To add a new theme:
1. Define a new set of CSS variable values.
2. Apply them via a class on `<body>` (same pattern as 1337 mode).
3. The entire UI will adapt — all components reference the variables.

Example: light theme could override `--bg: #f5f5f5; --text: #1a1a1a; --bg-card: #ffffff;`.

---

## 11. 1337 Mode (Easter Egg)

### Activation

**Triple-click the Japan flag effect card** within 500ms. The click listener runs in capture phase on `document`. On the third click, `stopPropagation()` prevents the normal `setEffect('japan')` behavior.

### Visual Changes

1. **Matrix rain background:** Full-viewport `<canvas>` with falling green katakana/code characters (`'01アイウエオカキクケコサシスセソ<>{}[]!@#$%^&*=+'`). Self-erasing via `rgba(0,0,0,0.05)` fill. Responsive to window resize.

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
| Hex input | Text field, `#RRGGBB` format, uppercase, synced with picker |
| Apply button | "UPLOAD TO MAINFRAME" — sends `setEffect('solid_color', { color: hex })` |

All three inputs (presets, picker, hex) are kept in sync bidirectionally.

### System Diagnostics Panel

8 stats, updated every 1 second:

| Stat | Source | Calculation |
|------|--------|-------------|
| Server Uptime | **Real** | `Date.now()/1000 - serverBootTime` (from `/api/status` `server_boot` field) |
| Effect Runtime | **Real** | From existing uptime text in footer |
| Power Draw | **Semi-real** | `1.44W * currentBrightness` (24 LEDs x ~60mW each) |
| Running Cost | **Semi-real** | `kWh * $0.12/kWh` (power draw x server uptime hours) |
| Photons Emitted | **Fake** | Random increment ~1-10 trillion/sec, displayed in scientific notation (`3.42e15`) |
| Neural Link | **Fake** | Oscillating `92 + 8 * sin(t*0.7) * sin(t*0.3)`, shown as `XX.X% SYNC` |
| Nodes Pwned | **Fake** | ~30% chance per second to increment by 1 |
| Hack Level | **Fake** | Time-based progression: Script Kiddie (0m) → Novice (1m) → Apprentice (3m) → Journeyman (5m) → Elite (10m) → Legendary (20m) → 1337 G0D (60m) |

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

Two exit methods:
1. Click the red "[ ESC ] EXIT 1337 M0D3" button.
2. Press the Escape key.

On exit: remove `leet-active` class, stop matrix rain animation, clear stats interval, restore header text to "RGB Lights".

### 1337 Mode State Variables

```javascript
let leetActive    = false;       // Is 1337 mode currently on?
let leetMatrixId  = null;        // requestAnimationFrame ID for matrix rain
let leetStatsId   = null;        // setInterval ID for stats updates
let leetEnteredAt = null;        // Unix timestamp when mode was entered
let leetPhotons   = 0;           // Running photon counter
let leetNodesPwned = 0;          // Running nodes counter
let serverBootTime = null;       // From /api/status server_boot field
let currentColor  = '#33ff33';   // Current color in color lab
```

---

## 12. Error Handling

### OpenRGB Connection Failures

**In `rgb_effects.py`:**
- If TCP connection to OpenRGB fails, print a clear error message and exit with code 1:
  ```
  ERROR: Could not connect to OpenRGB at openrgb:6742 — Connection refused
  ```
- The web server detects the subprocess exit (via `poll()`) and sets `running: false`.
- The browser's next `pollStatus()` will show the OFF state.

**In `rgb_server.py` (`GET /api/info`):**
- If the `--info` subprocess times out (15 seconds), return HTTP 504:
  ```json
  {"error": "OpenRGB connection timed out"}
  ```
- If any other error occurs, return HTTP 500.

### Device Not Found

If `find_mobo_device()` returns `None`:
```
ERROR: Could not find motherboard controller (looking for 'YOUR_DEVICE_NAME').
Available devices:
  Device 0: Some Other Device
```
Print available devices to help the user fix their `MOBO_DEVICE_NAME` constant, then exit with code 1.

### Subprocess Management

- If the effect subprocess crashes, `get_status()` detects it on the next poll (`poll() is not None`) and updates state to `running: false`.
- If `terminate()` doesn't work within 5 seconds, escalate to `kill()`.
- On server shutdown (SIGINT/SIGTERM), always attempt to kill the subprocess before exiting.
- When stopping effects (`POST /api/off`), always run a separate `--off` subprocess to zero all LEDs, even if the effect subprocess already died.

### Frontend Error Handling

- `fetchJSON()` wraps all API calls in try/catch. Returns `null` on any error (network, parse, timeout).
- `pollStatus()` silently skips if the fetch returns `null` — prevents UI flickering during transient network issues.
- Status pill shows "OFF" when the server reports `running: false`, regardless of reason.

### User-Friendly Error Scenarios

| Scenario | User Experience |
|----------|----------------|
| OpenRGB container not started | Status shows OFF. Effect clicks do nothing visible. `/api/info` returns timeout error. |
| Wrong device name configured | Effect starts but subprocess immediately exits. Status shows OFF. Check container logs. |
| Wrong LED count configured | LEDs light up incorrectly (wrong colors in wrong positions). User adjusts constants. |
| Browser loses connection to server | Status poll fails silently. UI freezes at last known state. Resumes automatically when connection restores. |
| Effect subprocess crashes | Status shows OFF within 3 seconds (next poll). User can click another effect. |

---

## 13. Platform-Specific Setup

### Linux (Native)

This is the primary supported platform. OpenRGB runs inside Docker with `privileged: true` for I2C access.

```bash
# 1. Load kernel modules (see Prerequisites)
sudo modprobe i2c-dev
sudo modprobe i2c-i801

# 2. Clone and build
git clone https://github.com/yourusername/free-rgb.git
cd free-rgb
docker compose up -d

# 3. Open in browser
xdg-open http://localhost:7777
```

**Verify OpenRGB detects hardware:**
```bash
docker logs openrgb
# Look for: "Detected controller: YOUR_DEVICE_NAME"
```

### Windows + WSL2

OpenRGB requires direct hardware access, which is not available inside WSL2. The recommended setup is:

1. **Run OpenRGB natively on Windows** (download from [openrgb.org](https://openrgb.org)):
   - Launch OpenRGB.
   - Go to Settings → enable "Start Server" on port `6742`.
   - Verify devices are detected.

2. **Run the web app in Docker on WSL2:**
   - Modify `docker-compose.yml`: remove the `openrgb` service entirely.
   - Set `OPENRGB_HOST` to your Windows host IP (usually the WSL gateway):
     ```bash
     # Find your Windows host IP from WSL2:
     ip route show default | awk '{print $3}'
     ```
   - Update the environment:
     ```yaml
     environment:
       - OPENRGB_HOST=172.x.x.x    # Your Windows host IP
       - OPENRGB_PORT=6742
     ```
   - Run only the web app:
     ```bash
     docker compose up -d rgb-lights
     ```

3. **Open in browser:** `http://localhost:7777`

**Important:** OpenRGB on Windows must be running with the SDK server enabled before starting the web app container.

---

## 14. Extension Points

### Adding a New Effect

1. **Python (`rgb_effects.py`):**
   - Create a new class extending `Effect`.
   - Implement `render(self, t, num_leds)`.
   - Optionally implement `render_static(num_leds)` if the effect has a meaningful static state.
   - Add an instance to `ALL_EFFECTS` dict.

2. **Server (`rgb_server.py`):**
   - Add an entry to the `EFFECTS` list with `name`, `description`, and `category`.

3. **Frontend (`index.html`):**
   - Add a matching renderer function to the `renderers` object (same math as Python).
   - If it has a static mode, add a static renderer to `staticRenderers`.
   - If it's a flag, add its name to the `FLAG_EFFECTS` Set.

4. **No other changes needed.** The grid, cards, canvas, controls, and API all adapt automatically.

### Adding a New Category

1. Add effects with the new category name in `EFFECTS` list.
2. Add a CSS accent color rule:
   ```css
   .effect-card[data-cat="New Category"] .card-preview { box-shadow: 0 0 12px rgba(R,G,B,0.3); }
   ```
3. The grid auto-groups by category name.

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

1. Add the parameter to `DEFAULTS` dict in `rgb_server.py`.
2. Add it to `_state`, `_save_state()`, `load_state()`, `start_effect()`, `_handle_set_effect()`.
3. Add the CLI argument in `rgb_effects.py`.
4. Pass it through to `run_effect()`.
5. Add a UI control in `index.html` (slider, toggle, etc.).
6. Include it in `setEffect()` POST body and `pollStatus()` sync logic.

### Future Enhancements (Ideas)

- **Effect favorites / custom ordering**
- **Effect scheduling** (time-of-day presets)
- **Music sync** (microphone input → beat detection → effect parameters)
- **Multi-device support** (control multiple OpenRGB controllers)
- **WebSocket** for real-time status (replace polling)
- **PWA support** (installable on phone home screen)
- **Effect parameter customization** (per-effect speed, color palette overrides)
