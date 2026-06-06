# One-Click API-Key Setup — Prototype

Static prototype of a `/api-key` one-click setup flow that takes a Claude Code user from "no `ANTHROPIC_API_KEY` found" to a working key written into their project's `.env` — without the AI ever seeing the key.

**🔗 Live demo:** https://wealthwiselabs.github.io/anthropic-api-key-setup-prototype/

**📄 Design spec:** [`docs/design-spec.md`](docs/design-spec.md) — API contract, security model, tool registry, prototype scope, open questions.

## Screens

| # | URL | Description |
|---|---|---|
| 1 | [01-claude-code-trigger.html](https://wealthwiselabs.github.io/anthropic-api-key-setup-prototype/01-claude-code-trigger.html) | Claude Code terminal — detects no key, suggests `/api-key`, user invokes the command, setup details stream in, press ⏎ |
| 2 | [02-setup-consent.html](https://wealthwiselabs.github.io/anthropic-api-key-setup-prototype/02-setup-consent.html) | `platform.claude.com/setup` consent — first-time user, $5 credits banner, "Claude Code ✓ verified" |
| 3 | [03-setup-success.html](https://wealthwiselabs.github.io/anthropic-api-key-setup-prototype/03-setup-success.html) | `platform.claude.com/setup` success — one-click "Send key to Claude Code" or download `.env` |
| 4 | [04-claude-code-resume.html](https://wealthwiselabs.github.io/anthropic-api-key-setup-prototype/04-claude-code-resume.html) | Claude Code resumes — spinner waits for key, status block streams in, then Claude makes one test call to `/v1/messages` that returns `200 OK` to verify the key before building |

Two extras for navigation: [`/screens.html`](https://wealthwiselabs.github.io/anthropic-api-key-setup-prototype/screens.html) (developer router with thumbnails) and [`/flow.html`](https://wealthwiselabs.github.io/anthropic-api-key-setup-prototype/flow.html) (composite of all 4 frames, used as the figure in the take-home doc).

## Run locally

```bash
git clone https://github.com/wealthwiselabs/anthropic-api-key-setup-prototype.git
cd anthropic-api-key-setup-prototype
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Screenshots

Use Cmd+Shift+4 (macOS). Set browser zoom to 100%. Capture at 1440×900 (terminal screens) or 1200×800 (setup screens) for consistency.

After capturing each screen, save into `assets/`:

- `screen-01.png` — Claude Code initiates setup (1440×900)
- `screen-02.png` — Setup consent (1200×900)
- `screen-03.png` — Setup success with one-click send (1200×900)
- `screen-04.png` — Claude Code resumes, verifies the key with a test API call (1440×900)

These get embedded in the take-home doc next to the Setup deep-dive section.

## Known mimicry caveats

- Anthropic's Tiempos / Styrene fonts are licensed; we use Charter / Inter as visual stand-ins.
- The Console wordmark icon is a simple "A" tile; close enough for screenshot purposes.
- Lucide icons are loaded from a CDN — internet connection required to render.
- The Claude Code terminal uses JetBrains Mono (loaded via Google Fonts).

## Files

- `index.html` — instant redirect to screen 1 (so the root URL drops viewers into the demo)
- `screens.html` — developer router with thumbnails of all 4 screens (use this to jump around)
- `flow.html` — composite page rendering all 4 frames in one shot for the take-home figure
- `01-claude-code-trigger.html`, `04-claude-code-resume.html` — share `styles/terminal.css` + `styles/stream.js`
- `02-setup-consent.html`, `03-setup-success.html` — share `styles/console.css`
- `styles/shared.css` — Google Fonts imports + base reset (used everywhere)
- `styles/stream.js` — character-by-character streaming engine used by the terminal screens
- `assets/icons/claude-code-mascot.png` — pixel mascot shown in the Claude Code welcome banner
- `assets/reference/` — source screenshots from the real Anthropic Console
- `docs/design-spec.md` — full API + UX + security spec for v1 (with v2 notes)
- `PLAN.md` — original implementation plan (historical; predates the Claude Code rewrite)
