# Free RGB

A beautiful, mobile-friendly web interface for controlling your PC's ARGB lighting — powered by [OpenRGB](https://openrgb.org) and designed to be built entirely by AI from a spec.

**You don't write the code. Your AI does.** This repo contains the complete specification for a professional RGB lighting control system. Hand `SPEC.md` to any AI coding assistant and it will produce a fully working app tailored to *your* hardware.

## What You Get

- **47 handcrafted effects** across 8 categories — Zelda themes, atmospheric scenes, world flags, pop culture tributes, and more
- **Real-time LED ring preview** — every effect rendered live in your browser, matching what's on your hardware
- **Animated mini-previews** on every effect card so you can see what you're picking before you pick it
- **Mobile-first responsive UI** — control your lights from your phone on the couch
- **Dark theme with category-colored accents** — looks great, no eye strain
- **Static/Animated mode toggle** for flag effects
- **Speed, brightness, and zone controls** — dial in the exact look you want
- **State persistence** — your last effect survives reboots
- **Docker-based** — consistent, reproducible, zero-dependency Python backend
- **Hidden easter egg** — find it if you can

## Prerequisites

Before anything else, you must confirm OpenRGB can detect and control your RGB devices:

1. **Install OpenRGB** on your machine — [openrgb.org/releases](https://openrgb.org/releases.html)
2. **Open it**, go to the **Devices** tab
3. **Verify your motherboard / controllers appear** and that you can change LED colors manually

If OpenRGB can't see your hardware, this project won't work. Fix that first — check the [OpenRGB supported devices list](https://openrgb.org/devices.html) and their [FAQ](https://openrgb.org/faq.html).

## Quick Start

```bash
git clone https://github.com/yourusername/free-rgb.git
cd free-rgb
```

Then hand `SPEC.md` to your AI assistant with a prompt like:

> Build this project according to SPEC.md. My motherboard is a [your board]. OpenRGB detects [your zones/devices]. I have [X] LEDs on [describe your setup].

The AI will generate all the code, Dockerfiles, and configuration — tailored to your specific hardware.

Once built:

```bash
docker compose up -d
```

Open `http://localhost:7777` and start controlling your lights.

## How It Works

```
Browser (phone/desktop)
    │
    │  HTTP API (port 7777)
    │
    ▼
┌─────────────┐         ┌──────────────┐
│  Web Server  │────────▶│  OpenRGB SDK  │
│  (Python)    │  spawn  │  Server       │
│              │◀────────│  (port 6742)  │
└─────────────┘  TCP     └──────┬───────┘
                                │
                           Hardware
                         (your LEDs)
```

The web server manages effect subprocesses that communicate with OpenRGB's SDK server over TCP. The browser talks to the web server via a clean REST API. No direct hardware access from the web app — ever.

## Project Structure

```
free-rgb/
├── docker-compose.yml          # Two-service stack (OpenRGB + web app)
├── Dockerfile                  # OpenRGB SDK server container
├── Dockerfile.lights           # Web app container (zero pip deps)
├── rgb_server.py               # Web server + process manager
├── effects/
│   └── rgb_effects.py          # Effect engine + OpenRGB SDK protocol
├── static/
│   └── index.html              # Single-file SPA (CSS + JS inline)
├── SPEC.md                     # Complete build specification
├── CONTRIBUTING.md              # How to add effects and contribute
├── LICENSE                      # MIT
└── README.md                   # You are here
```

## Supported Platforms

| Platform | OpenRGB | Web App |
|----------|---------|---------|
| Linux (native) | Native binary or AppImage | Docker |
| Windows + WSL2 | Windows native (GUI) | Docker in WSL2 |

See `SPEC.md` for detailed setup instructions for both platforms.

## License

[MIT](LICENSE) — do whatever you want with it.

## Credits

Built with [OpenRGB](https://openrgb.org) — the open-source RGB lighting control project that makes all of this possible.

Designed as an AI-first specification — the code is generated, the spec is human-crafted.
