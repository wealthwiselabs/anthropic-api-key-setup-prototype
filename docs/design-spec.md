# One-Click API-Key Setup — Prototype Spec

**Eric Sun · 2026-05-09 (last updated 2026-05-13) · companion to `Assignment_Final.md`**

---

## TL;DR

A new public Anthropic API surface (`POST https://api.anthropic.com/v1/api-keys/setup`) plus a slash command in Claude Code (`/api-key`) that **AI agents inside dev tools invoke autonomously** when they realize they need a Claude API key. The agent never sees the key. The user completes a verified browser handoff at `platform.claude.com/setup`, then with one click sends the key to the local destination (a project `.env` for Claude Code) — or for hosted partners, lands on a prefilled "add secret" page where the name is already populated.

**v1 is Anthropic-side only** (2–3 days, no partner engineering required). Server-to-server injection of the key value into hosted partner secrets stores is **v2**, gated on a launch partner like Lovable agreeing to expose an inject endpoint.

**Why this matters strategically.** Local-CLI users (Claude Code, Cursor, Cline) are Anthropic's home turf — but the fastest-growing developer cohort lives in hosted vibe-coding tools (Lovable, Bolt, v0, Replit) that already ship built-in AI covering Gemini and OpenAI but **not Claude**. Every Lovable user who wants Claude has to fight through manual key setup; that's structural disadvantage for Claude in the cohort that matters most. One-click setup makes choosing Claude as easy as accepting Lovable's defaults — competitive necessity, not just convenience. The prototype demonstrates v1 through Claude Code (the most natural launching surface to ship first); the same primitive extends to Lovable et al. via v2.

---

## What's net-new vs. existing primitives

| | NEW in v1 | REUSED |
|---|---|---|
| Public API | `POST /v1/api-keys/setup` (one endpoint) | — |
| Claude Code | `/api-key` slash command (in-CLI entry point) | Claude Code session/tool surface |
| Browser surface | `platform.claude.com/setup` consent + success page | Anthropic Console auth, payment capture |
| Config | Verified-tool registry (`tool=` slugs, display name, destination type) | Key-minting infra, audit log, `Settings → API Keys` |

One endpoint, one new browser page, one slash command, one config table. All Anthropic-side; no partner engineering. 2–3 days as planned.

**Out of v1 (deferred to v2):** server-to-server inject into hosted partner secrets endpoints, partner HMAC auth, partner registration self-service.

---

## Story arc (the four screenshots)

```
[1] Claude Code (CLI)         [2] platform.claude.com/setup    [3] platform.claude.com/setup    [4] Claude Code (CLI)
    ● No ANTHROPIC_API_KEY        (consent screen)                  (success — one-click            (key auto-detected)
      found. Run /api-key         Claude Code ✓ verified            send + download .env)            ● Resuming task…
    > /api-key                    [Approve]                         [Send to Claude Code]            tool calls succeed
    [Press Enter to open                                            +$5 credits
      platform.claude.com/setup]
```

**Critical property:** the key string never traverses the AI's context or any 3rd-party server. The only surface where the raw key value appears is the verified `platform.claude.com/setup` browser tab. From there, the user either (a) **local-env path**: clicks "Send to Claude Code" and the key is written to the project's `.env` via a local handoff (custom URI scheme handled by the CLI), or (b) **hosted-secrets path** (v2): Anthropic redirects to the partner's `callback_url` with the secret name prefilled and the user pastes/saves.

---

## API contract (single endpoint)

```http
POST https://api.anthropic.com/v1/api-keys/setup
Content-Type: application/json

{
  "tool": "claude-code",
  "tool_context": {
    "project_id": "money-wise-adventures",
    "project_name": "money-wise-adventures"
  },
  "key_name": "money-wise-adventures-key",
  "env_var": "ANTHROPIC_API_KEY",
  "destination": {
    "type": "local_env",
    "target_path": "frontend/.env"
  }
}
```

```json
// Response 200
{
  "setup_url": "https://platform.claude.com/setup?token=stp_a1b2c3..."
}
```

**Two destination shapes:**

- **`local_env`** (Claude Code, Cursor, Cline) — `destination.target_path` is a relative path in the project (or `~/.claude/.env` for global). After the user clicks Approve and lands on the success screen, the **"Send to <tool>"** button opens a custom URI (e.g. `claude://api-key/receive?token=...`) registered by the CLI installer. The local Claude Code daemon exchanges the token server-to-server for the key value and writes it to `target_path`. Falls back to copy/download if the URI handler isn't installed.

- **`hosted_secrets`** (Lovable, Bolt, v0, Replit) — `destination` instead carries a `callback_url`; Anthropic redirects to `{callback_url}?secret_name={env_var}&setup_token={one_time_token}` after the user clicks Copy. The partner's add-secret page opens with the **name field prepopulated**. The user pastes the key (already in clipboard from the Copy click) into the Value field and clicks Save. The key value **never travels through the URL** — putting secrets in URL params would leak them via browser history, HTTP Referer headers, and server logs (the OAuth spec explicitly forbids this for tokens). The optional `setup_token` is what enables the **v2 zero-paste experience**: the partner's backend exchanges it server-to-server for the key value, prefills the Value field securely, and the user just clicks Save.

**Properties:**
- **Unauthenticated** (the AI doesn't have a key yet — that's the whole point). Rate-limited per `tool` slug + IP.
- `tool` must match a registered, verified partner slug; unknown slugs → `400 unknown_tool`.
- One-time `token` baked into `setup_url`; 10-minute TTL; bound to the originating `tool` slug.
- **Local-env tools auto-resume** when the file at `target_path` appears (Claude Code watches `target_path` and reloads env). **Hosted-secrets tools** rely on the user saying "done" in chat once they've clicked Save.
- Audit-log entry is written at request time (`tool`, `tool_context`, IP, UA, `setup_id`).

**Why one POST instead of URL-only:** server-side validation of the tool slug *before* the user sees a consent page, plus replay-resistant one-time token, plus an audit row at request time. Cheap insurance.

### `/api-key` (Claude Code slash command)

Claude Code ships a built-in `/api-key` command that any agent (or user) can invoke. When the agent detects a missing `ANTHROPIC_API_KEY`, it tells the user to run `/api-key`. The command:

1. Calls `POST /v1/api-keys/setup` with `tool=claude-code` and the current project's name + target `.env` path.
2. Renders the setup details inline (key name, env var, target path) and a `Press [Enter] to open …` prompt.
3. On Enter, opens `setup_url` in the user's default browser.
4. Watches `target_path` for the key file and silently resumes the prior request once it appears.

---

## Browser flow at `platform.claude.com/setup`

### Path branches after the user clicks **Approve**

| User state | Path |
|---|---|
| First-time user | Mint key → grant existing $5 trial credits → success screen with "+$5 credits added" |
| Returning, has credits or payment method | Mint key → success screen |
| Returning, no credits & no payment method | Inline payment capture → on payment captured, mint key → success screen |

**Note on credits.** Anthropic already grants $5 trial credits to new accounts (phone verification, no card required). Setup doesn't introduce new credits or expand the offer; it just makes the existing $5 actually reachable for devs who today bounce before reaching Console.

### 3a. Consent screen

```
┌─────────────────────────────────────────────┐
│  ⚡ Set up API key for your project         │
│                                             │
│  Claude Code ✓ verified                     │
│  Request from your MacBook · San Francisco  │
│  is creating a Claude API key for project   │
│  "money-wise-adventures"                    │
│                                             │
│  Key name:  [money-wise-adventures-key] ✎   │
│  Env var:   ANTHROPIC_API_KEY               │
│  Billing:   Your Anthropic account          │
│                                             │
│  🎁 New here? You'll get $5 in free         │
│     credits with your first key.            │
│                                             │
│  This key has standard API access. Revoke   │
│  anytime in Console.                        │
│                                             │
│  [ Deny ]                       [ Approve ] │
└─────────────────────────────────────────────┘
```

Device + coarse location surface phishing/replay anomalies. Key name is editable inline.

### 3b. Success screen (first-time user, local-env destination)

```
┌─────────────────────────────────────────────┐
│  ✓ Your Claude API key is ready             │
│  + $5 free credits added to your account    │
│                                             │
│  money-wise-adventures-key                  │
│  ┌─────────────────────────────────────┐    │
│  │ sk-ant-api03-•••••••••••••••a3f2 👁 │    │
│  └─────────────────────────────────────┘    │
│  [ Send key to Claude Code ]   [ Download .env ] │
│                                             │
│  Next: return to your terminal              │
│  1. Click Send key to Claude Code — the key │
│     is written to frontend/.env             │
│  2. Switch back — Claude Code detects the   │
│     key and resumes automatically           │
│                                             │
│  [ Return to Claude Code ↗ ]   [ Manage in  │
│                                  Console ↗ ]│
└─────────────────────────────────────────────┘
```

The primary action button is conditional on `destination.type`:

- **`local_env` (Claude Code, Cursor, Cline)** — `[Send key to <tool>]` triggers the custom URI handoff to the local CLI which writes to `target_path`. `[Download .env]` is the fallback.
- **`hosted_secrets` (Lovable, Bolt, v0, Replit)** — `[Copy key]` copies to clipboard and activates `[Return to <tool>]`, which deep-links to the partner's prefilled add-secret page.

### Edge-case states (specced, not all mocked)

| State | Treatment |
|---|---|
| Not signed in | Redirect to existing Anthropic login, return to consent on success |
| No payment method, no credits, returning user | Inline Stripe-style card capture between Approve and mint |
| Key not used within 24h | Auto-revoke; user re-runs setup (low cost since the AI is the trigger). |
| Token expired (>10 min) | "This setup request expired. Ask your assistant to try again." |
| User clicks Deny | "Setup cancelled. You can ask your assistant to try again." Audit log entry. |
| Local URI handler not installed | Falls back to clipboard + download; user pastes into `target_path` manually. |

---

## Tool registry (v1 — minimal)

The `tool=` slug drives the verified badge, the destination behavior, and the action labels on the success screen. Internal Anthropic config; manually curated in v1.

| Field | Example (claude-code) | Example (lovable) |
|---|---|---|
| `slug` | `claude-code` | `lovable` |
| `display_name` | `Claude Code` | `Lovable` |
| `verified` | `true` | `true` |
| `logo_url` | `cdn.anthropic.com/tools/claude-code.svg` | `cdn.anthropic.com/tools/lovable.svg` |
| `destination_type` | `local_env` | `hosted_secrets` |
| `uri_handler` | `claude://api-key/receive` | — |
| `secrets_deep_link` | — | `https://lovable.dev/projects/{project_id}/secrets` |
| `instructions_text` | "Writes to your project's `.env`" | "Paste into Lovable → Project Settings → Secrets as `ANTHROPIC_API_KEY`" |

No partner secrets endpoint, no HMAC auth, no inject call — those are v2.

**v2 additions (when a launch partner signs):**
- `secrets_endpoint` + `auth` fields per hosted partner
- Anthropic → partner inject call (replaces copy/paste for that tool's success screen)
- Partner SDK / partner-onboarding self-service
- mTLS, exchange-code, keypair auth (v3 — see `Assignment_Final.md` A3)

**Always out of scope (also see Assignment A3):**
- Multi-tool key sharing
- Scoped permissions (read-only, model-restricted keys)
- Org/team-account flows

---

## Security model — deltas vs. `Assignment_Final.md` A4

The doc's A4 covers the broader security framing. Specifically for the AI-initiated v1:

| Concern | Mitigation in v1 |
|---|---|
| Unauthenticated request endpoint | Per-`tool`-slug + per-IP rate limits; only registered slugs accepted; one-time short-lived token in the setup URL |
| AI sees key | Eliminated by design — key is never in any API response the AI sees |
| Clipboard sniffing on copy step | Auto-revoke key if not used within 24h; key only visible on the verified `platform.claude.com` tab; user controls timing |
| URL replay / link sharing | One-time token, 10-min TTL, slug-bound; consumed on first load |
| Key in URL params (callback) | **Never.** `callback_url` carries `secret_name` + an opaque `setup_token` only. Putting the key value in URL params would leak it via browser history, Referer headers, and server logs (OAuth spec forbids this for tokens). |
| Local URI handler hijacking | The `claude://` handler is registered by the CLI installer; the success page validates the handoff token against the original `setup_id` before any write. |
| Phishing fake "setup" page | `platform.claude.com` is the trust anchor; consent screen surfaces device + location + verified-tool badge |
| `.env` committed to git | Success page reminder text + secret-scanning auto-revoke partnerships (GitHub, GitLab). Target path defaults to a path the project's existing `.gitignore` already excludes; for projects without a `.env` pattern, the success page warns. |
| Key proliferation | Console UI groups auto-created keys by tool; one key per (tool, project) tuple, not per session |

**Policy compatibility (Jan-2026 OAuth ban).** Setup mints standard API-tier-billed keys, not consumer-subscription tokens — same posture as `Assignment_Final.md` argues. The new AI-initiated framing doesn't change this: the user still authenticates to Anthropic, the key is still API-tier billed.

---

## Prototype scope — exactly what gets screenshot

**4 primary screenshots** (the hero flow as built):

1. **Claude Code — `/api-key` trigger.** User had asked Claude Code to build a lesson generator. Claude reads the project, then reports: *"No `ANTHROPIC_API_KEY` found in your environment or `frontend/.env`. Run `/api-key` to start a one-click setup with Anthropic."* User types `/api-key`. Claude renders the setup details (key name, env var, target path) and a `Press [Enter] to open https://platform.claude.com/setup` prompt. Output streams character-by-character to feel like a real terminal.
2. **`platform.claude.com/setup` — consent screen.** First-time-user variant with the $5 credits banner, "Claude Code ✓ verified," device line, editable key name. Uses Section 3a layout.
3. **`platform.claude.com/setup` — success screen.** First-time-user variant with "+$5 credits added," masked key + `[Send key to Claude Code]` and `[Download .env]` buttons. Send-to-Claude-Code's success state reads "✓ Sent to Claude Code" and activates the Return CTA. Uses Section 3b layout.
4. **Claude Code — resumed.** Spinner: *"Waiting for key from platform.claude.com/setup…"* then the status block fades in (key received, fingerprint, $5 credit) and Claude resumes the original request — `Bash` calls `/v1/messages`, `Write` creates `src/lessons/budgeting-101.json`, `Edit` shows a unified diff on `src/journey/path.tsx`, and a final summary bullet wraps the work. All streams in like real Claude Code output.

**3 secondary screenshots** (specced, mocked only if time permits):

5. Returning-user consent variant (no $5 banner).
6. Inline payment capture (returning user without credits/payment).
7. Hosted-secrets variant of 3b — `[Copy key]` + `[Return to Lovable]` with the Lovable Cloud add-secret prefilled landing.

**Format.** Live deployed via GitHub Pages at `https://wealthwiselabs.github.io/anthropic-api-key-setup-prototype/` for the take-home submission. The trigger and resume screens use a small JS streaming engine (`styles/stream.js`) so text lands character-by-character; static screenshots can be captured at any moment in the playback.

---

## Open questions to validate on day 1

- **Unauthenticated endpoint abuse vectors.** What's the realistic abuse pattern (mass setup-token minting → spam consent screens)? Rate-limits + slug verification likely sufficient, but security review on day 1.
- **Trial-credit fraud risk via verified-channel signup.** Does routing new signups through a verified-tool channel raise or lower fraud risk vs. open Console signup? Hypothesis: lower (the tool itself adds a friction layer); validate with fraud team.
- **Custom URI scheme reliability.** What % of macOS/Linux/Windows installs cleanly register `claude://` such that the success page's `[Send to Claude Code]` button just works? Need fallback UX for the long tail.
- **`.env` collision detection.** If the project already has `ANTHROPIC_API_KEY` set in `.env` or the shell, should the local handoff refuse to overwrite? Hypothesis: yes, warn on the success page before the button activates.
- **v2 launch-partner readiness.** Will Lovable (or Bolt / v0 / Replit) agree to expose a secrets-inject endpoint to upgrade their flow from copy/paste to zero-click? Concrete partnership conversation post-v1 launch.

---

## Sequencing

| Day | Work |
|---|---|
| 1 | Backend: `POST /v1/api-keys/setup`, tool-registry config (3–5 launch tools: Claude Code, Cursor, Cline, Lovable, Bolt), one-time-token store, audit log integration |
| 1 | Claude Code: `/api-key` slash command + `claude://` URI handler; file-watcher for `target_path` to auto-resume |
| 1–2 | Frontend: `platform.claude.com/setup` consent + success screens (reuse Console design system); destination-conditional action buttons |
| 2–3 | Edge cases (payment capture inline, expired token, 24h auto-revoke, URI-handler fallback), QA, launch behind feature flag with Claude Code + Cursor as local-env launch tools |

Cohort comparison vs. manual-setup baseline; 6–8 week launch window per `Assignment_Final.md`.
