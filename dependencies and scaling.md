Below is everything I know of from our work, grouped by role. Things I'm not certain are in your code are marked "(check)". At the end there's a prompt for Claude Code to produce the exact, complete list from the repository, including every code library and its version.

## 1. Services the live product runs on (they bill you, or will)

| Provider | What it does in SIM.AI | How it bills |
|---|---|---|
| **Google Gemini API** (AI Studio) | Main live speech-to-speech engine (`gemini-3.5-live-translate-preview`), 79 languages | Prepaid credits, about $0.06 per minute per language (≈ $3.60/hour) |
| **OpenAI API** | Second live engine (`gpt-realtime-translate`), 13 output languages | Per minute of audio, about $0.034/min |
| **Microsoft Azure: Speech** | Speech recognition, Azure voices and the voice catalogue (Germany West Central) | About $1/hour recognition; voices about $15 per million characters |
| **Microsoft Azure: Translator** | Translation fallback, dictionary markup, quick-lookup translation | About $10 per million characters |
| **Microsoft Azure: Azure OpenAI** | GPT-4.1-mini (live interpreter + phrase judge), GPT-4.1 (document preparation) | Per token |
| **LiveKit Cloud** | The real-time audio rooms: floor audio in, translated audio and captions out to phones | Build (free, hard stop) → Ship $50/month + overage |
| **Vercel** | Hosts the dashboard (admin.sim-trans.com), attendee page and ingest agent | Free/Pro plan, per project |
| **Railway** | Runs the worker-hosts (the translation workers) | Usage-based monthly |
| **Neon** | The Postgres database (sessions, glossaries, usage, reports) | Plan + compute hours (you already hit a quota once) |
| **Upstash** (check) | Redis: locks, session ownership, commands between servers | Free tier / usage |
| **Cloudflare** | DNS for the sim-trans.com subdomains, and **R2** storage for recordings, uploads and prepared downloads | DNS free; R2 per GB stored + operations, no download fees |
| **GitHub** | Code repository (SIM-AI); pushes trigger the Vercel and Railway deploys | Free/paid plan |
| **Domain registrar** for sim-trans.com | The domain itself | Yearly renewal |

## 2. Tools used to build it

| Tool | Role | Cost |
|---|---|---|
| **Anthropic Claude** (Claude Code in VS Code, and this chat) | Writing and testing the code, planning, prompts | Subscription + usage credits |
| VS Code, Git | Editor and version control | Free |
| Node.js 22+, pnpm, TypeScript, Next.js | Language, package manager and web framework | Free, open source |
| Docker Desktop | Local test services: Redis, Postgres, MinIO, LiveKit, Toxiproxy | Free for small businesses (check the licence threshold for your company size) |
| Playwright (Chromium), Vitest, ESLint, Prettier, tsx | Automated tests and code quality | Free |
| Drizzle ORM | Database migrations (`pnpm db:migrate`) | Free |
| ffmpeg | Audio conversion, on your laptop and on the workers | Free |
| **ElevenLabs** | Generated some of the test speech samples in `samples/` | Your ElevenLabs plan |

## 3. Main code libraries inside the product (check: exact list from the prompt below)

These include the LiveKit client and server SDKs, the Microsoft Speech SDK (1.51.0), the Google and OpenAI SDKs, a Postgres driver, Zod (validation), a Redis client, an S3 client (for R2), plus QR code, Excel/Word export, PDF/PPTX text extraction and font packages (Noto). They're free open-source libraries, but each has a licence, and that matters if you ever sell on-premise copies.

## 4. Used for marketing and sales materials (not the product)

ChatGPT (brochure images), Codex (an early landing page), Google Drive (email image hosting), Gmail (sending the marketing email), WhatsApp (the contact button), and the brochure, sales briefing and cost card I produced.

## 5. Tested or evaluated, not in use

Azure Live Interpreter (tested, too slow), general real-time voice models used as interpreters (gpt-realtime, Gemini native audio; tested, disqualified), the Jev classifier (TypeSafe), Together AI and Fireworks (looked at), and ElevenLabs and Cartesia voices (alternatives to Azure voices).

## 6. Likely in the future

| Provider or category | When you'd add it |
|---|---|
| **Azure UAE North** region | Clients who need data kept in the UAE |
| **LiveKit Ship/Scale** plan | Any real event over 100 listeners (needed soon) |
| **Error and uptime monitoring** (for example Sentry, Better Stack) | Before regular paid events. Worth adding early: you'd learn of an outage before the client does |
| **Email sending service** (for example Resend, Postmark) | SaaS version: sign-ups, password resets, reports by email |
| **Payments** (for example Stripe) | SaaS version for interpreters and organisers |
| **Azure Document Intelligence** (OCR) | Preparation from scanned PDFs |
| **Self-hosted LiveKit**, Kubernetes or Azure Container Apps | Large scale or on-premise clients |
| Azure Custom Neural Voice / Custom Speech | A client paying for a branded voice or heavy-accent recognition |

## 7. Hardware

A laptop, a USB audio interface (for example a Behringer UMC202HD or Focusrite Scarlett 2i2) with cables (and possibly a DI box), the venue mixer or interpretation console (Bosch, Taiden), and infrared receivers for the relay trick.

## Prompt for Claude Code: the exact list from the code

```
Read-only task. Produce docs/providers-and-dependencies.md: a complete inventory of every
external provider, service and dependency this project uses, from the actual code and config
(not from memory). Do not change anything else.

1. External services: every API or cloud service the code calls, found from env vars
   (.env.example, DEPLOY.md), SDK imports and hostnames in the code. For each: what it is
   used for, which apps/services call it, which env vars configure it, whether it is
   required for live events or optional, and where its cost is configured in cost.ts.
2. Hosting/deploy: Vercel projects, Railway services, how each builds (Dockerfile or not),
   domains and DNS records referenced.
3. Code dependencies: every package in every package.json (production vs dev), grouped by
   purpose, with version and licence (flag any non-permissive licence such as GPL/AGPL or
   commercial terms). Include system tools the code shells out to (e.g. ffmpeg).
4. Fonts, icon sets and other bundled third-party assets, with their licences.
5. Single points of failure: which providers, if down, stop a live event, and what the
   system does in that case today.
Summarise the totals in your reply.
```

I can also turn this list into a tracker (provider, account owner, plan, monthly cost, renewal date, who has access) so nothing renews or runs out by surprise. Given the Gemini credits running out once, that would be useful.
