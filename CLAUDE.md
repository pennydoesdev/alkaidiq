# CLAUDE.md — Alkaid AI

## Project Overview

Alkaid AI is a multi-model AI chat SaaS PWA at **chat.alkaidiq.com**. The entire app — backend API + frontend SPA — lives in a single Cloudflare Worker file (`src/worker.js`, ~7,100 lines, ~342KB). There is no build step, no framework, no package.json. Wrangler bundles it with esbuild and deploys directly.

**Owner:** Penny Tribune (systems@pennytribune.com)
**GitHub:** https://github.com/pennydoesdev/alkaidiq (branch: `main`)
**Deploy:** Cloudflare Git integration — push to `main` auto-deploys to `chat.alkaidiq.com`
**No build command** — leave Cloudflare build command empty. Wrangler handles esbuild bundling.

## Repository Structure

```
src/worker.js       — The entire application (backend + frontend in template literal)
src/admin-html.js   — Admin dashboard SPA (exported as ES module, imported by worker.js)
chat-worker.js      — Mirror copy of src/worker.js (legacy deploy artifact)
wrangler.toml       — Cloudflare Workers config (bindings, routes, vars)
DEVELOPER.md        — Detailed architecture docs, API endpoints, security checklist
```

## Architecture

- **Runtime:** Cloudflare Workers (V8 isolate, no Node.js APIs)
- **Database:** Cloudflare D1 (SQLite) — `env.DB`
- **Storage:** Cloudflare R2 — `env.R2` (uploads), `env.RAWFEED` (conversation logs), `env.INTELLIGENCE` (analytics)
- **AI:** Cloudflare Workers AI — `env.AI` (Whisper STT, Deepgram Aura-2 TTS, FLUX image gen)
- **LLM:** Featherless AI (OpenAI-compatible API) — server-managed key stored in `admin_settings` table
- **Payments:** Stripe Checkout + Webhooks

## ⚠️ CRITICAL: Template Literal Escaping

The frontend HTML/CSS/JS is embedded inside a backtick template literal (`const HTML = \`...\``). esbuild processes this at bundle time and WILL break backslash sequences. **This is the #1 source of bugs.**

### Rules for code INSIDE the template literal:

1. `'\\n'` (double backslash) → serves as `'\n'` at runtime. Never use `'\n'` alone.
2. `\\'` (double backslash + quote) → serves as `\'` in onclick handlers. Never use `\'` alone.
3. **Never use regex with `\s`, `\b`, `\w`, `\d` inline** — esbuild strips the backslash. Instead, use `window._RE` which is injected server-side.
4. To add new regex patterns, find the `reScript` injection block (~line 3568) and add them there.
5. `String.fromCharCode(92)` and `atob()` with backslash content will be inlined by esbuild — don't use them for escaping.

### Code OUTSIDE the template literal (server-side handlers) is normal JavaScript — no escaping issues.

## Models

All models are server-managed via Featherless AI. No BYOK (Bring Your Own Key) — API keys are stored server-side.

| ID | Name | Model | Tier |
|----|------|-------|------|
| `basic-7b` | Alkaid Basic | Qwen/Qwen2.5-7B-Instruct | free |
| `orb` | Orb | Qwen/Qwen2.5-72B-Instruct | orb |
| `orb-code` | Orb Code | Qwen/Qwen3-Coder | orb |
| `orb-deepseek` | Orb DeepSeek | deepseek-ai/DeepSeek-V3.2 | orb |
| `orb-sight` | Orb Sight | FLUX.1-schnell (image gen) | orb |
| `orb-talk` | Orb Talk | Deepgram Aura-2 (TTS) | orb |

Default model: `basic-7b`

## Subscription Tiers

| Tier | DB Value | Access |
|------|----------|--------|
| Free | `null` or `'free'` | basic-7b only |
| Orb | `'orb'` | All models + image gen + voice + priority |
| Superadmin | `'superadmin'` | Everything + admin panel + hidden from user lists |

Stripe handles Orb subscriptions ($9.99/mo) via checkout + webhooks.

## Key File Sections (src/worker.js)

Line numbers are approximate and shift with edits. Use grep to find sections.

- **~1-1000**: Imports, admin-html import, helper functions (uuid, cors, auth)
- **~1005-1012**: `MODELS` dictionary
- **~1016-1100**: `ensureDB()` — auto-creates all D1 tables
- **~1500-1800**: Auth API handlers (signup, login, verify, password reset)
- **~1800-2035**: LLM streaming functions (streamOpenAI, streamAnthropic, streamGoogle, streamCohere)
- **~2035-2044**: `getStreamFn()` — provider → stream function router
- **~2050-2400**: Admin API handlers
- **~2420-2500**: `logToRawFeed()` — R2 logging utility
- **~2507+**: `handleAPI()` — main authenticated API router
- **~2767**: `/api/chat` — streaming chat completion endpoint
- **~3420-3431**: `/api/voice/stt` — Whisper speech-to-text
- **~3433-3445**: `/api/voice/tts` — Deepgram text-to-speech
- **~3447-3610**: `/api/voice/stream` — **Real-time streaming voice pipeline** (STT → LLM → TTS, returns audio chunks via SSE)
- **~3485+**: `export default { fetch() }` — Worker entry point, attaches `ctx` to request
- **~3700**: Start of template literal (`const HTML = \`...`)
- **~4163-4257**: Voice overlay CSS (Supernova animation)
- **~4449-4460**: `authFetch()` — frontend auth wrapper
- **~4462+**: `init()` — app bootstrap
- **~5375**: `renderMessages()` — chat message rendering
- **~5818+**: Voice mode frontend (streaming pipeline client)
- **~6750+**: Voice overlay HTML (Supernova UI)
- **~7050+**: End of template literal, `reScript` injection, service worker

## Voice System (Streaming Pipeline)

The voice system uses a real-time SSE pipeline. Flow:

1. **Client** records audio via MediaRecorder → sends blob to `POST /api/voice/stream`
2. **Server** runs Whisper STT → streams transcript to Featherless LLM → detects sentence boundaries → runs Deepgram TTS on each sentence → sends base64 audio chunks back via SSE
3. **Client** receives SSE events, queues audio chunks, plays them sequentially with gapless playback

SSE event types:
- `{"type":"transcript","text":"..."}` — STT result
- `{"type":"text","content":"..."}` — LLM text token
- `{"type":"audio","data":"base64...","text":"..."}` — TTS audio for one sentence
- `{"type":"conversationId","id":"..."}` — new conversation created
- `{"type":"done","fullText":"...","conversationId":"..."}` — stream complete
- `{"type":"error","message":"..."}` — error

The old batch endpoints (`/api/voice/stt`, `/api/voice/tts`) still exist for single-message TTS playback (read-aloud button on messages).

## Deployment

Push to `main` on GitHub triggers Cloudflare auto-deploy. No build command needed.

```bash
git add -A
git commit -m "description of changes"
git push origin main
```

To verify deployment:
```bash
curl -s "https://chat.alkaidiq.com/?v=$(date +%s)" | grep 'window._RE='
```

## Secrets (set via `wrangler secret put`)

- `FEATHERLESS_API_KEY` — or stored in D1 `admin_settings` table as `featherless_api_key`
- `STRIPE_SECRET_KEY` — Stripe API key
- `STRIPE_WEBHOOK_SECRET` — Stripe webhook signing secret
- `SES_ACCESS_KEY_ID` — AWS SES for email verification
- `SES_SECRET_ACCESS_KEY` — AWS SES secret

## Other Alkaid Workers (not in this repo)

- **alkaid-web** — Landing site at alkaidiq.com (separate worker)
- **alkaid-a-api** — model.alkaidiq.com (separate worker)

These are independent and not yet in any GitHub repo.

## Common Tasks

**Add a new model:** Add entry to the `MODELS` dictionary (~line 1005). If it uses Featherless, set `keyName: 'featherless'`. Set `tier: null` for free, `tier: 'orb'` for paid.

**Add a new API endpoint:** Add an `if (path === '...')` block inside `handleAPI()`. All routes there are authenticated (user object available). Use `json()` helper for responses, `corsHeaders()` for CORS.

**Modify frontend UI:** Edit inside the template literal. Remember the escaping rules. Test locally with `wrangler dev`.

**Add a new regex pattern:** Add to the `reScript` block server-side (~line 3568), then reference via `window._RE.yourPattern` in frontend code.

## Known Issues / TODO

- [ ] CSP header not yet implemented
- [ ] ARIA labels for accessibility needed
- [ ] `chat-worker.js` is a manual mirror of `src/worker.js` — keep them in sync or remove it
- [ ] Consider splitting the single-file architecture as it approaches 8,000+ lines
