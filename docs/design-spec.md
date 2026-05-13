# One-Click API-Key Setup — Prototype Spec

**Eric Sun · 2026-05-09 · companion to `Assignment_Final.md`**

---

## TL;DR

A new public Anthropic API surface (`POST https://api.anthropic.com/v1/api-keys/setup`) that **AI agents inside dev tools call autonomously** when they realize they need a Claude API key. The agent never sees the key. The user completes a verified browser handoff at `claude.com/setup`, **copies the key or downloads `.env.anthropic`** from a one-time success page, then is redirected back to a `callback_url` in the dev tool — landing on a prefilled "add secret" page where the secret name is already populated. User clicks Save, returns to chat, says "done."

**v1 is Anthropic-side only** (2–3 days, no partner engineering required). Server-to-server injection of the key value into partner secrets stores is **v2**, gated on a launch partner like Lovable agreeing to expose an inject endpoint.

**Why Lovable specifically.** Lovable already ships built-in AI ("Lovable AI") for chat / image / document use cases — but it only covers Gemini and OpenAI. Every Lovable user who wants Claude has to fight through manual key setup. That's structural disadvantage for Claude in the largest vibe-coding cohort. One-click setup makes choosing Claude as easy as accepting Lovable's defaults — competitive necessity, not just convenience. The same logic applies to Bolt and v0 (both ship built-in AI without Claude).

---

## What's net-new vs. existing primitives

| | NEW in v1 | REUSED |
|---|---|---|
| Public API | `POST /v1/api-keys/setup` (one endpoint) | — |
| Browser surface | `claude.com/setup` consent + copy/download success page | Anthropic Console auth, payment capture |
| Config | Verified-tool registry (`tool=` slugs, display name, logo) | Key-minting infra, audit log, `Settings → API Keys` |

One endpoint, one new page, one config table. All Anthropic-side; no partner engineering. 2–3 days as planned.

**Out of v1 (deferred to v2):** server-to-server inject into partner secrets endpoints, partner HMAC auth, partner registration onboarding flow.

---

## Story arc (the four screenshots)

```
[1] Lovable chat              [2] claude.com/setup            [3] claude.com/setup            [4] Lovable chat
    AI: "I need a Claude          (consent screen)                (success — copy/download)       User: "done"
    key — opening setup           Lovable ✓ verified              [Copy key]                      AI: retries call,
    in your browser…"             [Approve]                       [Download .env.anthropic]        succeeds, ships
                                                                  +$5 credits
```

**Critical property:** the key string never traverses the AI's context or any 3rd-party server. The only surface where it appears is the verified `claude.com/setup` browser tab; from there, the user moves it directly into the destination (Lovable Secrets UI, OS keychain, or project `.env`).

---

## API contract (single endpoint)

```http
POST https://api.anthropic.com/v1/api-keys/setup
Content-Type: application/json

{
  "tool": "lovable",
  "tool_context": {
    "project_id": "proj_abc123",
    "project_name": "my-app"
  },
  "key_name": "Lovable - my-app - May 2026",
  "env_var": "ANTHROPIC_API_KEY",
  "callback_url": "https://lovable.dev/projects/proj_abc123/cloud/secrets/new"
}
```

```json
// Response 200
{
  "setup_url": "https://claude.com/setup?token=stp_a1b2c3..."
}
```

**`callback_url` behavior.** After the user clicks Copy on the success screen, Anthropic redirects to `{callback_url}?secret_name={env_var}&setup_token={one_time_token}` — landing on the partner's add-secret page with the **name field prepopulated**. The user pastes the key (already in clipboard from the Copy click) into the Value field and clicks Save. The key value **never travels through the URL** — putting secrets in URL params would leak them via browser history, HTTP Referer headers, and server logs (the OAuth spec explicitly forbids this for tokens).

The optional `setup_token` is what enables the **v2 zero-paste experience**: the partner's backend exchanges it server-to-server for the key value, prefills the Value field securely, and the user just clicks Save. v1 ships with name-only prefill (no partner backend changes required); v2 adds the exchange-code path once a launch partner integrates.

**Properties:**
- **Unauthenticated** (the AI doesn't have a key yet — that's the whole point). Rate-limited per `tool` slug + IP.
- `tool` must match a registered, verified partner slug; unknown slugs → `400 unknown_tool`.
- One-time `token` baked into `setup_url`; 10-minute TTL; bound to the originating `tool` slug.
- No status / polling / cancel endpoints. **The user signals completion to the AI in chat** ("done"); the AI retries its original API call, which now finds the key in env vars and succeeds.
- Audit-log entry is written at request time (`tool`, `tool_context`, IP, UA, `setup_id`).

**Why one POST instead of URL-only:** server-side validation of the tool slug *before* the user sees a consent page, plus replay-resistant one-time token, plus an audit row at request time. Cheap insurance.

---

## Browser flow at `claude.com/setup`

### Path branches after the user clicks **Approve**

| User state | Path |
|---|---|
| First-time user | Mint key → grant existing $5 trial credits → success screen with copy/download + "+$5 credits added" |
| Returning, has credits or payment method | Mint key → success screen with copy/download |
| Returning, no credits & no payment method | Inline payment capture → on payment captured, mint key → success screen |

**Note on credits.** Anthropic already grants $5 trial credits to new accounts (phone verification, no card required). Setup doesn't introduce new credits or expand the offer; it just makes the existing $5 actually reachable for vibe-coders who today bounce before reaching Console.

### 3a. Consent screen

```
┌─────────────────────────────────────────────┐
│  ⚡ Set up Claude for Lovable               │
│                                             │
│  Lovable ✓ verified                         │
│  Request from your MacBook · San Francisco  │
│  is creating a Claude API key for project   │
│  "my-app"                                   │
│                                             │
│  Key name:  [Lovable - my-app - May 2026] ✎ │
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

### 3b. Success screen (first-time user)

```
┌─────────────────────────────────────────────┐
│  ✓ Your Claude API key is ready             │
│  + $5 free credits added to your account    │
│                                             │
│  Lovable - my-app - May 2026                │
│  ┌─────────────────────────────────────┐    │
│  │ sk-ant-api03-•••••••••••••••a3f2 👁 │    │
│  └─────────────────────────────────────┘    │
│  [ Copy key ]   [ Download .env.anthropic ]  │
│                                             │
│  Next: paste into Lovable → Project         │
│  Settings → Secrets as ANTHROPIC_API_KEY,   │
│  then return to chat and say "done."        │
│                                             │
│  [ Open Lovable Secrets ↗ ]  [ Manage in    │
│                                Console ↗ ]  │
└─────────────────────────────────────────────┘
```

The destination instructions and "Open Secrets" deep link are conditional on `tool` slug:
- **Lovable / Bolt / v0 / Replit** → "paste into [tool] Project Secrets" + deep link to that tool's secrets UI
- **Cursor / Cline / Claude Code** → "drop `.env.anthropic` into your project root" or "paste into your IDE's API key field"

### Edge-case states (specced, not all mocked)

| State | Treatment |
|---|---|
| Not signed in | Redirect to existing Anthropic login, return to consent on success |
| No payment method, no credits, returning user | Inline Stripe-style card capture between Approve and mint |
| Key not used within 24h | Auto-revoke; user re-runs setup (low cost since the AI is the trigger). |
| Token expired (>10 min) | "This setup request expired. Ask your assistant to try again." |
| User clicks Deny | "Setup cancelled. You can ask your assistant to try again." Audit log entry. |

---

## Tool registry (v1 — minimal)

The `tool=` slug drives the verified badge and the destination instructions on the success screen. Internal Anthropic config; manually curated in v1.

| Field | Example |
|---|---|
| `slug` | `lovable` |
| `display_name` | `Lovable` |
| `verified` | `true` |
| `logo_url` | `cdn.anthropic.com/tools/lovable.svg` |
| `destination_type` | `hosted_secrets` (Lovable, Bolt, v0, Replit) or `local_env` (Cursor, Cline, Claude Code) |
| `destination_deep_link` | `https://lovable.dev/projects/{project_id}/secrets` (hosted only) |
| `instructions_text` | "Paste into Lovable → Project Settings → Secrets as ANTHROPIC_API_KEY" |

No partner secrets endpoint, no HMAC auth, no inject call — those are v2.

**v2 additions (when a launch partner signs):**
- `secrets_endpoint` + `auth` fields per partner
- Anthropic → partner inject call (replaces copy/download for that tool's success screen)
- Partner SDK / partner-onboarding self-service
- mTLS, exchange-code, keypair auth (v3 — see `Assignment_Final.md` A3)

**Always out of scope (also see Assignment A3):**
- Multi-tool key sharing
- Scoped permissions (read-only, model-restricted keys)
- Org/team-account flows

---

## Security model — deltas vs. `Assignment_Final.md` A4

The doc's A4 covers the broader security framing. Specifically for the AI-initiated, copy/download v1:

| Concern | Mitigation in v1 |
|---|---|
| Unauthenticated request endpoint | Per-`tool`-slug + per-IP rate limits; only registered slugs accepted; one-time short-lived token in the setup URL |
| AI sees key | Eliminated by design — key is never in any API response the AI sees |
| Clipboard sniffing on copy step | Auto-revoke key if not used within 24h; key only visible on the verified `claude.com` tab; user controls timing |
| URL replay / link sharing | One-time token, 10-min TTL, slug-bound; consumed on first load |
| Key in URL params (callback) | **Never.** `callback_url` carries `secret_name` + an opaque `setup_token` only. Putting the key value in URL params would leak it via browser history, Referer headers, and server logs (OAuth spec forbids this for tokens). |
| Phishing fake "setup" page | `claude.com` is the trust anchor; consent screen surfaces device + location + verified-tool badge |
| `.env.anthropic` committed to git | File is named `.env.anthropic` (matches the conventional `.env*` `.gitignore` pattern most projects already use; also doesn't clobber an existing `.env`); reminder text on success screen; secret-scanning auto-revoke partnerships (GitHub, GitLab) |
| Key proliferation | Console UI groups auto-created keys by tool; one key per (tool, project) tuple, not per session |

**Policy compatibility (Jan-2026 OAuth ban).** Setup mints standard API-tier-billed keys, not consumer-subscription tokens — same posture as `Assignment_Final.md` argues. The new AI-initiated framing doesn't change this: the user still authenticates to Anthropic, the key is still API-tier billed.

---

## Prototype scope — exactly what gets screenshot

**5 primary screenshots** (the hero flow):

1. **Lovable chat — AI trigger.** AI says: *"You asked for Claude. To use it, I need an Anthropic API key set as `ANTHROPIC_API_KEY` in your project secrets. Click below to set it up."* Shows a primary `[Open setup at Claude Console ↗]` button.
2. **`claude.com/setup` — consent screen.** First-time-user variant with the $5 credits banner, "Lovable ✓ verified," device line, editable key name. (Uses Section 3a layout.)
3. **`claude.com/setup` — success screen.** First-time-user variant with "+$5 credits added," masked key + Copy + Download buttons; Return-to-Lovable CTA gates until the user takes action. (Uses Section 3b layout.)
4. **Lovable Cloud — prefilled secret.** Anthropic redirects via `callback_url` to Lovable Cloud → Secrets → Add new secret with the secret **name** (`ANTHROPIC_API_KEY`) prepopulated. User pastes the key from clipboard (or clicks "Paste from clipboard"), Save activates, user clicks Save. (v2 with exchange-code: value is prepopulated server-side and Save is the only click.)
5. **Lovable chat — handoff.** User: "done." AI: *"Got it — re-running your request now…"* and the original task succeeds (key now in env vars).

**3 secondary screenshots** (specced, mocked only if time permits):

5. Returning-user consent variant (no $5 banner).
6. Inline payment capture (returning user without credits/payment).
7. Local-tool destination variant (Cursor — "drop `.env.anthropic` into project root or paste into Cursor's API key field").

**Format.** Static screenshots only; no live deploy. Hosted alongside other prototypes at `anthropic.growbi.app/setup` for the take-home submission.

---

## Open questions to validate on day 1

- **Unauthenticated endpoint abuse vectors.** What's the realistic abuse pattern (mass setup-token minting → spam consent screens)? Rate-limits + slug verification likely sufficient, but security review on day 1.
- **Trial-credit fraud risk via verified-channel signup.** Does routing new signups through a verified-tool channel raise or lower fraud risk vs. open Console signup? Hypothesis: lower (the tool itself adds a friction layer); validate with fraud team.
- **Key naming convention.** `<Tool> - <project> - <Month Year>` is opinionated; check with design that it doesn't collide with existing Console naming.
- **v2 launch-partner readiness.** Will Lovable (or Bolt / v0 / Replit) agree to expose a secrets-inject endpoint to upgrade their flow from copy/paste to zero-click? Concrete partnership conversation post-v1 launch.

---

## Sequencing

| Day | Work |
|---|---|
| 1 | Backend: `POST /v1/api-keys/setup`, tool-registry config (3–5 launch tools: Lovable, Bolt, Cursor, Cline, Claude Code), one-time-token store, audit log integration |
| 1–2 | Frontend: `claude.com/setup` consent + copy/download success screens (reuse Console design system); destination instructions per `tool` slug |
| 2–3 | Edge cases (payment capture inline, expired token, 24h auto-revoke), QA, launch behind feature flag with Lovable + Cursor as launch tools |

Cohort comparison vs. manual-setup baseline; 6–8 week launch window per `Assignment_Final.md`.
