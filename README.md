# Free RGB

A beautiful, mobile-friendly web interface for controlling your PC's ARGB lighting — powered by [OpenRGB](https://openrgb.org) and designed to be built entirely by AI from a spec.

**You don't write the code. Your AI does.** This repo contains the complete specification for an RGB lighting control system — **not** the generated code. Hand [`SPEC.md`](SPEC.md) to any AI coding assistant and it will produce a fully working app tailored to *your* hardware. `SPEC.md` is the source of truth; if this README ever disagrees with it, the spec wins.

## What You Get

- **47 handcrafted effects** across 8 categories — Zelda themes, atmospheric scenes, world flags, pop culture tributes, and more
- **Real-time LED ring preview** — every effect rendered live in your browser, sized to the LED counts OpenRGB reports for your zones
- **Animated mini-previews** on every effect card so you can see what you're picking before you pick it
- **Mobile-first responsive UI** — control your lights from your phone on the couch
- **Dark theme with category-colored accents** — looks great, no eye strain
- **Static/Animated mode toggle** for flag effects
- **Speed (0.25×–3×), brightness, and zone controls** — dial in the exact look you want
- **State persistence** — your last effect survives restarts
- **Run it in Docker or natively** — the backend is pure Python stdlib, zero pip dependencies
- **Secure by default** — binds to localhost, OpenRGB's port stays private, strict input validation
- **Hidden easter egg** — find it if you can

## Prerequisites

Before anything else, you must confirm OpenRGB can detect and control your RGB devices:

1. **Install OpenRGB** on your machine — [openrgb.org/releases](https://openrgb.org/releases.html)
2. **Open it** (as administrator / root for motherboard SMBus devices), go to the **Devices** tab
3. **Verify your motherboard / controllers appear** and that you can change LED colors manually
4. **Note your device name, zone names, and LED counts** from the **Information** tab — your AI needs them

On Linux, load the SMBus kernel modules first: `i2c-dev` plus `i2c-i801` (Intel) or `i2c-piix4` (AMD). Details are in [SPEC.md §1](SPEC.md#1-prerequisites).

If OpenRGB can't see your hardware, this project won't work. Fix that first — check the [OpenRGB supported devices list](https://openrgb.org/devices.html) and their [FAQ](https://openrgb.org/faq.html).

## Quick Start

**1. Clone the repo**

```bash
git clone https://github.com/mrshaun13/free-rgb.git
cd free-rgb
```

**2. Generate the code from `SPEC.md`.** The repo ships without code, compose files, or Dockerfiles — your AI creates all of them. Hand it `SPEC.md` with a prompt like:

> Build this project according to SPEC.md, generating every file in its Section 3 into this folder. My controller is a [device name from OpenRGB]. My zones are [zone name: LED count, …]. My layout is [describe fans/strips/pump head]. I'm running [Linux + Docker / Linux native / Windows + WSL2 / Windows native].

**3. Run it** (only after step 2 — `docker compose` has nothing to start until the compose file is generated):

```bash
# Linux + Docker
docker compose up -d --build

# Windows + WSL2 (OpenRGB runs on Windows)
docker compose -f docker-compose.wsl2.yml up -d --build

# Native, no Docker (OpenRGB SDK server already running on this machine)
OPENRGB_HOST=127.0.0.1 python3 rgb_server.py
```

Open `http://localhost:7777` and start controlling your lights.

Want to use it from your phone? By default the UI only listens on localhost. Read [SPEC.md §13 Security](SPEC.md#13-security) for the safe ways to open it to your home network — and **never expose ports 7777 or 6742 to the internet**. In short:

- **Any mode:** best option is a reverse proxy with a login in front of `127.0.0.1:7777`.
- **Docker:** set `WEB_BIND` to the machine's LAN address, on a trusted network only. Note that **ufw/firewalld do not filter Docker-published ports** (Docker's iptables rules bypass them) — use the `DOCKER-USER` chain rules in the spec if you need a firewall.
- **Native:** bind to the LAN and restrict port 7777 to your LAN subnet with ufw (or Windows Firewall).
- **WSL2:** ports are only forwarded to Windows localhost; LAN access needs `netsh interface portproxy` + a Windows firewall rule, or mirrored networking + a Hyper-V firewall rule (see SPEC.md §14).

## How It Works

```
Browser (phone/desktop)
    │
    │  HTTP + JSON API (port 7777)
    ▼
┌──────────────────┐  spawns   ┌──────────────────┐   TCP 6742   ┌──────────────────┐
│  Web server      │──────────▶│  Effect process  │─────────────▶│  OpenRGB SDK     │
│  rgb_server.py   │  kills /  │  rgb_effects.py  │  SDK protocol│  server          │
│                  │  restarts │  (30 FPS loop)   │              │                  │
└──────────────────┘           └──────────────────┘              └────────┬─────────┘
                                                                          │
                                                                     Hardware
                                                                    (your LEDs)
```

The web server serves the UI and a small REST API, and starts/stops one effect subprocess at a time. That subprocess renders frames and streams them to OpenRGB's SDK server over TCP. Only OpenRGB touches hardware — the web app never does.

## Project Structure

After your AI generates the code, the folder looks like this. Files marked *generated* are not in this repo.

```
free-rgb/
├── docker-compose.yml          # generated — Linux Docker mode (OpenRGB + web app)
├── docker-compose.wsl2.yml     # generated — WSL2 mode (web app only)
├── Dockerfile                  # generated — OpenRGB SDK server image
├── Dockerfile.lights           # generated — web app image (zero pip deps)
├── rgb_server.py               # generated — web server + process manager
├── effects/
│   └── rgb_effects.py          # generated — effect engine + OpenRGB SDK protocol
├── static/
│   └── index.html              # generated — single-file SPA (CSS + JS inline)
├── .gitattributes              # forces LF line endings
├── SPEC.md                     # complete build specification (source of truth)
├── CONTRIBUTING.md             # how to add effects and contribute
├── LICENSE                     # MIT
└── README.md                   # you are here
```

## Supported Platforms

| Platform | OpenRGB runs | Web app runs | Notes |
|----------|--------------|--------------|-------|
| Linux + Docker (primary) | In a privileged Docker container | Docker | Host loads the i2c kernel modules |
| Linux native | Native binary or AppImage | `python3` on the host | No Docker needed |
| Windows + WSL2 | Windows native (GUI, SDK server enabled) | Docker in WSL2 | Needs networking + firewall setup for port 6742 |
| Windows native | Windows native (GUI, SDK server enabled) | `py` on Windows | Simplest Windows option |

See [SPEC.md §14](SPEC.md#14-platform-specific-setup) for step-by-step setup for each.

## Configuration

Everything is set with environment variables (or a `.env` file next to the compose file) — nothing about your machine is hard-coded. The most common ones:

| Variable | Default | What it does |
|----------|---------|--------------|
| `HOSTNAME_LABEL` | `Free RGB` | Name shown in the UI footer |
| `TZ` | `UTC` | Timezone for logs |
| `PORT` | `7777` | Web UI port |
| `WEB_BIND` | `127.0.0.1` | Host address the UI is published on (Docker) |
| `OPENRGB_HOST` | `127.0.0.1` (native) / `openrgb` (Docker) | Where the OpenRGB SDK server is |
| `STATE_DIR` | `/data`, else `~/.local/state/free-rgb` | Where the last effect is saved |

The full list is in [SPEC.md §4](SPEC.md#4-configuration-environment-variables).

## License

[MIT](LICENSE) — do whatever you want with it.

## Credits

Built with [OpenRGB](https://openrgb.org) — the open-source RGB lighting control project that makes all of this possible.

Designed as an AI-first specification — the code is generated, the spec is human-crafted.
