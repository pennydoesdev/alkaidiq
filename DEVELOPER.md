# Alkaid AI — Developer Documentation

## Architecture Overview

Alkaid AI is a multi-model AI chat PWA running on Cloudflare Workers. The entire application is a **single-file SPA** (`src/worker.js`) containing both the backend API handlers and the frontend HTML/CSS/JS inside a JavaScript template literal.

### Stack
- **Runtime**: Cloudflare Workers (V8 isolate)
- **Database**: Cloudflare D1 (SQLite)
- **Storage**: Cloudflare R2 (images, rawfeed, intelligence)
- **AI**: Cloudflare Workers AI (`env.AI`)
- **Bundler**: esbuild (via wrangler)
- **Payments**: Stripe Checkout + Webhooks

### Bindings (wrangler.toml)
| Binding | Type | Purpose |
|---------|------|---------|
| `env.DB` | D1 | User accounts, sessions, logs, conversations |
| `env.R2` | R2 | User-uploaded files |
| `env.RAWFEED` | R2 | Raw conversation feed storage |
| `env.INTELLIGENCE` | R2 | Intelligence data storage |
| `env.AI` | Workers AI | FLUX image generation, whisper transcription |

## Models (Alkaid-Managed)

| Model ID | Name | Provider | Tier |
|----------|------|----------|------|
| `basic-7b` | Alkaid Basic | Featherless (Qwen 2.5 7B) | Free |
| `orb` | Orb | Featherless (Qwen 2.5 72B) | Orb |
| `orb-code` | Orb Code | Featherless (Qwen3 Coder) | Orb |
| `orb-deepseek` | Orb DeepSeek | Featherless (DeepSeek V3.2) | Orb |
| `orb-sight` | Orb Sight | CF Workers AI (FLUX) | Orb |
| `orb-talk` | Orb Talk | CF Workers AI (Deepgram Aura 2) | Orb |

Users can also add unlimited custom providers via Settings > Custom Providers.

## Deployment

```bash
# From the repo root:
wrangler deploy
```

## Security Checklist
- [x] All SQL queries use parameterized `.bind()` — no string concatenation
- [x] XSS protection: dangerous tags stripped from markdown
- [x] Stripe webhook signature verification
- [x] Password hashing via PBKDF2 (crypto.subtle, 100K iterations)
- [x] CORS scoped to `chat.alkaidiq.com`
- [x] CSP header (+ X-Content-Type-Options, Referrer-Policy, X-Frame-Options) via `htmlHeaders()` helper in `src/worker.js`
- [x] ARIA labels for accessibility (icon-only buttons, modals, form inputs, decorative icons)
