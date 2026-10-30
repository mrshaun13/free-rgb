# Contributing to Free RGB

Thanks for your interest in contributing! This project is designed to be AI-generated from a spec, but human contributions are welcome for spec improvements, new effects, and bug reports.

**What lives in this repo:** only documentation — [`SPEC.md`](SPEC.md) (the source of truth), `README.md`, this file, `LICENSE`, and `.gitattributes`. The app code, compose files, and Dockerfiles are generated from the spec and are not checked in. Most contributions are therefore changes to `SPEC.md`. If this guide ever disagrees with `SPEC.md`, the spec wins.

## Quick Start

```bash
git clone https://github.com/mrshaun13/free-rgb.git
cd free-rgb
# 1. Generate the code: hand SPEC.md + your hardware details to your AI assistant
#    (see README.md "Quick Start"). This creates docker-compose.yml, the Dockerfiles,
#    rgb_server.py, effects/rgb_effects.py, and static/index.html.
# 2. Then run it:
docker compose up -d --build                              # Linux + Docker
# docker compose -f docker-compose.wsl2.yml up -d --build # Windows + WSL2
# OPENRGB_HOST=127.0.0.1 python3 rgb_server.py            # native, no Docker
# 3. Open http://localhost:7777
```

`docker compose up` will fail with "no configuration file provided" until step 1 has generated `docker-compose.yml`.

Don't commit your generated code, `.env`, `openrgb-config/`, or `data/` in PRs to this repo, and never include your own hostnames, IPs, or hardware serials in examples — use generic placeholders like `Example Board X1`.

## How to Contribute

### Adding a New Effect

This is the most common contribution. An effect touches **three code files** in the generated app, and its definition must be added to **`SPEC.md`** so every future generation includes it. A PR to this repo is the `SPEC.md` change; test it in your generated app first.

**1. Python effect class** (`effects/rgb_effects.py`)

```python
class MyEffect(Effect):
    name        = "my_effect"
    description = "Short description of what it looks like"

    def render(self, t: float, num_leds: int) -> List[Tuple[int, int, int]]:
        colors = []
        for i in range(num_leds):
            # Your animation math here
            # t = elapsed seconds * speed multiplier
            # i = LED index (0 to num_leds-1)
            colors.append((r, g, b))
        return colors
```

Then add it to `ALL_EFFECTS`:
```python
"my_effect": MyEffect(),
```

**2. Server metadata** (`rgb_server.py`)

Add one line to the `EFFECTS` list:
```python
{"name": "my_effect", "description": "Short description of what it looks like", "category": "Your Category"},
```

**3. JavaScript renderer** (`static/index.html`)

Add a matching function to the `renderers` object that produces the same visual output:
```javascript
my_effect(t, n) {
    const out = [];
    for (let i = 0; i < n; i++) {
        // Same math as your Python render()
        out.push([r, g, b]);
    }
    return out;
},
```

**4. `SPEC.md`**

- Add a row to the right category table in Section 9 (name, description, notes on how it looks).
- Update the counts: the category heading, the Section 9 summary table, the "47 visible effects" mentions in Section 6, and the effect count in `README.md`.

The grid, cards, preview ring, API validation, and state management all adapt automatically.

### Adding a New Flag Effect

Same as above, plus:
- Implement `render_static(self, num_leds)` in Python (returns colors without animation).
- Add a `staticRenderers.my_flag(n)` function in JavaScript.
- Add `'my_flag'` to the `FLAG_EFFECTS` Set in JavaScript.
- Consider using the `_TricolorFlag` base class if it's a three-stripe flag — you only need to set three color constants.

### Adding a New Category

Add the effects with the new category name, plus a CSS accent rule for `[data-cat="New Category"]` in `index.html`, and add the category to the tables in `SPEC.md` Sections 9 and 10.

### Improving the Spec

If you find something in `SPEC.md` that's unclear, missing, contradictory, or wrong — fix it. The spec is the most important file in this repo. PRs that improve spec clarity are always welcome. When you change a behavior, update every section that mentions it (and README/CONTRIBUTING if they summarize it) so the docs never disagree.

### Bug Reports

If generated code doesn't work as the spec describes, file an issue or submit a spec PR that makes the requirement clearer. Include:
- What you expected (reference the spec section).
- What actually happened, plus your platform/run mode (Linux Docker, Linux native, WSL2, Windows native).
- Steps to reproduce.

Please report security problems (e.g. a way to reach files outside `static/`) privately via a GitHub security advisory rather than a public issue.

## Code Style (for generated code and spec examples)

- **Python:** No external dependencies. Stdlib only. Validate all input (see `SPEC.md` §13).
- **JavaScript:** Vanilla JS. No frameworks, no npm, no build tools. Everything in one HTML file. Use `textContent`, never `innerHTML`, for server-provided strings.
- **CSS:** Use CSS custom properties (the `--var` system). Never hard-code colors that should be themeable.
- **Effects:** Keep the Python and JavaScript renderers in sync. Same math, same variable names where possible.
- **Line endings:** LF everywhere (enforced by `.gitattributes`).

## Effect Design Guidelines

Good effects:
- Look distinct from existing effects at a glance.
- Use the full LED ring (don't leave most LEDs dark unless that's the point).
- Respond to the `t` (time) parameter smoothly — no jarring jumps.
- Look good across the full speed range, 0.25× to 3×.
- Work with any number of LEDs (don't assume a specific count).

Avoid:
- Strobing faster than ~4 Hz at **any** speed setting, including 3× (seizure risk).
- Pure white at full brightness for extended periods (LED longevity).
- Hard-coding LED counts — always use the `num_leds` parameter.

## Commit Convention

Use [conventional commits](https://www.conventionalcommits.org/):

```
feat(effects): add northern lights effect
fix(spec): clarify zone resize behavior after OpenRGB reconnect
docs(spec): clarify zone sizing requirements
chore(spec): update OpenRGB version in Dockerfile
```

## Questions?

Open an issue. We're friendly.
