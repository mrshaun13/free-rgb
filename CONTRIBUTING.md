# Contributing to Free RGB

Thanks for your interest in contributing! This project is designed to be AI-generated from a spec, but human contributions are welcome for bug fixes, new effects, and spec improvements.

## Quick Start

```bash
git clone https://github.com/yourusername/free-rgb.git
cd free-rgb
docker compose up -d
# Open http://localhost:7777
```

## How to Contribute

### Adding a New Effect

This is the most common contribution. Every effect requires changes in exactly three places:

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

That's it. The grid, cards, preview ring, API, and state management all adapt automatically.

### Adding a New Flag Effect

Same as above, plus:
- Implement `render_static(self, num_leds)` in Python (returns colors without animation).
- Add a `staticRenderers.my_flag(n)` function in JavaScript.
- Add `'my_flag'` to the `FLAG_EFFECTS` Set in JavaScript.
- Consider using the `_TricolorFlag` base class if it's a three-stripe flag — you only need to set three color constants.

### Improving the Spec

If you find something in `SPEC.md` that's unclear, missing, or wrong — fix it. The spec is the most important file in this repo. PRs that improve spec clarity are always welcome.

### Bug Fixes

If something doesn't work as described in the spec, file an issue or submit a PR. Include:
- What you expected (reference the spec section).
- What actually happened.
- Steps to reproduce.

## Code Style

- **Python:** No external dependencies. Stdlib only. Follow the existing patterns in the codebase.
- **JavaScript:** Vanilla JS. No frameworks, no npm, no build tools. Everything in one HTML file.
- **CSS:** Use CSS custom properties (the `--var` system). Never hard-code colors that should be themeable.
- **Effects:** Keep the Python and JavaScript renderers in sync. Same math, same variable names where possible.

## Effect Design Guidelines

Good effects:
- Look distinct from existing effects at a glance.
- Use the full LED ring (don't leave most LEDs dark unless that's the point).
- Respond to the `t` (time) parameter smoothly — no jarring jumps.
- Look good at both speed=0.5x and speed=2x.
- Work with any number of LEDs (don't assume a specific count).

Avoid:
- Strobing faster than ~4Hz (seizure risk).
- Pure white at full brightness for extended periods (LED longevity).
- Hard-coding LED counts — always use the `num_leds` parameter.

## Commit Convention

Use [conventional commits](https://www.conventionalcommits.org/):

```
feat(effects): add northern lights effect
fix(server): handle OpenRGB disconnect during effect switch
docs(spec): clarify zone sizing requirements
chore: update OpenRGB version in Dockerfile
```

## Questions?

Open an issue. We're friendly.
