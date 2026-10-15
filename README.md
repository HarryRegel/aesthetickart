# AestheticKart — Gen Z Aesthetic Fashion Marketplace

Low-cost, mobile-first marketplace for **only aesthetic / trending Gen Z fashion** — from Sarojini Nagar and local-market sellers — with **AI listing verification** and **virtual try-on**.

> This README is written *before* the code exists. The coding agent must keep every command and table below accurate as the project is built (see `AgentPrompt.md`).

| File | What it is |
|---|---|
| `Project-Details.md` | Full spec: features, architecture, AI, payments, data model, security, **phased plan** |
| `AgentPrompt.md` | Operating instructions for the coding agent |
| `.env.example` | Canonical list of every environment variable |
| `README.md` | This file — setup & usage for humans |

---

## 1. What's in the platform

| App | URL (prod) | Who uses it |
|---|---|---|
| **Storefront** `apps/storefront` | `xyz.com` | Shoppers: browse by aesthetic/category, try on virtually, pay with Razorpay |
| **Seller platform** `apps/seller` | `seller.xyz.com` | Shop owners/suppliers: apply, get verified, list products (Snap & List), manage orders & payouts |
| **Admin console** `apps/admin` | `admin.xyz.com` | Internal team: verify/suspend/remove sellers, moderate products, AI settings, finance, audit logs |
| **Worker** `apps/worker` | internal | AI verification, try-on, webhooks, notifications, settlements |

**Key features:** aesthetic taxonomy (classic + modern) · AI aesthetic/trend verification · AI-assisted listing · virtual try-on + skin-tone-aware colour suggestions + size recommendation · multi-seller cart · Razorpay checkout & Route payouts · Shiprocket logistics · returns/refunds · seller earnings dashboard · hardened admin (2FA, RBAC, IP allowlist, audit log).

---

## 2. Architecture (short)

```
            ┌──────────── storefront ────────────┐
Shopper ───▶│ SSR/ISR, PWA, search, PDP, checkout │──┐
            └─────────────────────────────────────┘  │
            ┌────────────── seller ───────────────┐  │      ┌──────────────┐
Seller ────▶│ onboarding, Snap&List, orders, $$$   │──┼────▶ │  PostgreSQL  │
            └─────────────────────────────────────┘  │      │  + pgvector  │
            ┌────────────── admin ────────────────┐  │      └──────┬───────┘
Admin ─────▶│ 2FA, RBAC, moderation, finance, logs │──┘             │
            └─────────────────────────────────────┘          ┌──────┴───────┐
                          │ enqueue (pg-boss)                │   Storage    │
                          ▼                                  │ public+private│
                    ┌──────────┐   AI vision/embeddings       └──────────────┘
                    │  worker  │──▶ try-on provider ──▶ Razorpay / Shiprocket / SMS / Email
                    └──────────┘
```

Tech: TypeScript · Next.js · Tailwind/shadcn · Drizzle · Postgres (Supabase) · pg-boss · Upstash Redis · Razorpay · Shiprocket · Gemini (default AI adapter) · MediaPipe (in-browser face/skin analysis). Full table in `Project-Details.md §5`.

---

## 3. Prerequisites

- **Node.js** (current LTS) and **pnpm** (`npm i -g pnpm`)
- **Git**
- A free **Supabase** project (or Neon) for Postgres — no Docker needed
- Optional (add progressively, each has a mock mode): Razorpay test account, Upstash, Resend, MSG91, Shiprocket, Gemini API key

> **Low-spec laptop tip:** run only the app you're working on (`pnpm dev:storefront`, etc.) — not all four at once. No local ML models are required; AI runs through hosted APIs or mock mode.

---

## 4. Quick start

```bash
git clone <your-repo-url> && cd <repo>
pnpm install

# 1) Create your env file
cp .env.example .env
#    → fill at least DATABASE_URL, DIRECT_URL, AUTH_SECRET, ADMIN_AUTH_SECRET, FIELD_ENCRYPTION_KEY
#      (generate secrets with:  openssl rand -base64 48)

# 2) Check what's configured (prints names only, never values)
pnpm env:check

# 3) Create tables + seed demo data (aesthetics, categories, demo sellers/products)
pnpm db:migrate
pnpm db:seed

# 4) Create the first super-admin (uses ADMIN_BOOTSTRAP_EMAIL / PASSWORD from .env)
pnpm db:seed:admin

# 5) Run an app (one at a time is fine)
pnpm dev:storefront    # http://localhost:3000
pnpm dev:seller        # http://localhost:3001
pnpm dev:admin         # http://localhost:3002
pnpm dev:worker        # background jobs
```

With the default mode flags (`OTP_MODE=console`, `EMAIL_MODE=console`, `STORAGE_PROVIDER=local`, `AI_MODE=mock`, `TRYON_MODE=mock`, `SHIPPING_MODE=mock`, `PAYMENTS_MODE=test`) the platform runs **without any third-party keys** (only the database is required). OTPs and emails are printed in the terminal.

---

## 5. 🔑 How to fill the environment variables (`.env.example` → `.env`)

1. **Copy:** `cp .env.example .env`
2. **Fill** only what you need for the phase you're on (the agent tells you after each phase: *"Env vars you need to fill now"*).
3. **Restart** the dev server after every change.
4. **Never commit `.env`.** It's git-ignored. Only `.env.example` (no secrets) is committed.
5. Variables starting with `NEXT_PUBLIC_` are **public** (visible in the browser). Everything else is **server-only secret**.
6. If a required variable is missing, the app exits with a friendly table telling you what's missing and where to get it.
7. Run `pnpm env:check` any time to see which variables are set and which mock/live modes are active.

### Where to get each key (summary)

| Variables | Service | How to get it | Card needed? | Needed from |
|---|---|---|---|---|
| `DATABASE_URL`, `DIRECT_URL` | **Supabase** (Postgres) | supabase.com → New project → Project Settings → Database → Connection string (pooler → `DATABASE_URL`; direct → `DIRECT_URL`). Enable the `vector` extension (Database → Extensions) | No | Phase 0 |
| `AUTH_SECRET`, `ADMIN_AUTH_SECRET`, `FIELD_ENCRYPTION_KEY` | You generate | `openssl rand -base64 48` (use `32` for the encryption key). Use **different** values | — | Phase 1 |
| `ADMIN_BOOTSTRAP_EMAIL/PASSWORD` | You choose | Used once by `pnpm db:seed:admin`; rotate password after first login, then delete these lines | — | Phase 1 |
| `ADMIN_ALLOWED_IPS` | You | Your office/home public IP(s), comma-separated. Blank = allow all (**dev only**) | — | Phase 1 |
| `UPSTASH_REDIS_REST_URL/TOKEN` | **Upstash** | upstash.com → Create Redis → REST API tab | No | Phase 1 |
| `OTP_MODE` → `msg91` + `MSG91_*` | **MSG91** | msg91.com → API → Auth Key; create an OTP template & complete DLT for production SMS | Varies | Phase 1 (optional) |
| `RESEND_API_KEY` (+ `EMAIL_MODE=resend`) | **Resend** | resend.com → API Keys; verify your domain for production | No | Phase 1 (optional) |
| `STORAGE_PROVIDER=supabase` + `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | **Supabase Storage** | Project Settings → API (service_role is a **secret**). Create buckets `public-media` (public) and `private-docs` (**private**) | No | Phase 1–2 |
| `GOOGLE_CLIENT_ID/SECRET` | **Google Cloud** | console.cloud.google.com → APIs & Services → Credentials → OAuth client (Web) | No | Phase 1 (optional) |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `NEXT_PUBLIC_RAZORPAY_KEY_ID` | **Razorpay** | dashboard.razorpay.com → switch to **Test Mode** → Account & Settings → API Keys → Generate. The `NEXT_PUBLIC_` one is the same **key id** only — never the secret | No (test) | Phase 4 |
| `RAZORPAY_WEBHOOK_SECRET` | Razorpay | Settings → Webhooks → Add (URL `…/api/webhooks/razorpay`, events listed in `.env.example`) and set your own secret. Locally use a tunnel (cloudflared/ngrok) | No | Phase 4 |
| `ROUTE_ENABLED` + Route setup | Razorpay Route | Must be activated on your Razorpay account; otherwise keep `false` and use manual ledger payouts | Business KYC | Phase 7 |
| `GEMINI_API_KEY`, `AI_VISION_MODEL`, `AI_EMBEDDING_MODEL` (+ `AI_MODE=live`) | **Google AI Studio** | aistudio.google.com/apikey. Pick current model names from the provider docs. Free tier is rate-limited | No | Phase 5 |
| `TRYON_PROVIDER`, `TRYON_API_KEY`, `TRYON_MODEL` (+ `TRYON_MODE=live`) | Try-on provider (fashn / fal / replicate / gemini) | Create an account with the chosen provider and read its current pricing & free-tier/card policy. **Costs real money per image** — keep the budget variables low | Varies | Phase 6 |
| `SHIPROCKET_EMAIL/PASSWORD` (+ `SHIPPING_MODE=shiprocket`) | **Shiprocket** | Create an API user in Settings → API | Business KYC | Phase 7 |
| `WHATSAPP_*` | Meta WhatsApp Cloud API | developers.facebook.com → WhatsApp → API setup | Business verification | Phase 7 (optional) |
| `SENTRY_DSN`, `NEXT_PUBLIC_SENTRY_DSN` | **Sentry** | sentry.io → Projects → Client Keys (DSN) | No | Any |
| `NEXT_PUBLIC_POSTHOG_KEY` | **PostHog** | posthog.com → Project API key | No | Phase 3+ |

> Provider free-tier and card policies change — check each site before signing up. The platform is designed so you can **skip any provider** and stay on its mock mode until you're ready.

### Mode flags (run partly or fully without keys)

| Flag | Values | Default | Effect |
|---|---|---|---|
| `OTP_MODE` | `console` · `msg91` | `console` | OTP printed in terminal vs. real SMS |
| `EMAIL_MODE` | `console` · `resend` | `console` | |
| `STORAGE_PROVIDER` | `local` · `supabase` · `r2` | `local` | Dev-only local files vs. cloud buckets |
| `PAYMENTS_MODE` | `test` · `live` | `test` | Razorpay test vs. live keys |
| `AI_MODE` | `mock` · `live` | `mock` | Fake deterministic AI vs. real provider |
| `TRYON_MODE` | `mock` · `live` | `mock` | Placeholder try-on vs. real provider |
| `SHIPPING_MODE` | `mock` · `shiprocket` | `mock` | |
| `WHATSAPP_MODE` | `off` · `console` · `meta` | `off` | |

### Local webhooks
Razorpay and Shiprocket need a public URL to reach your laptop. Use a tunnel (e.g., `cloudflared tunnel --url http://localhost:3000`) and put that URL in the provider dashboard. Never commit tunnel URLs.

---

## 6. Common scripts (agent will implement these)

| Command | Purpose |
|---|---|
| `pnpm dev:storefront` / `dev:seller` / `dev:admin` / `dev:worker` | Run one app/service |
| `pnpm build` | Build everything |
| `pnpm lint` · `pnpm typecheck` · `pnpm test` | Quality gates |
| `pnpm test:e2e` | Playwright end-to-end tests |
| `pnpm db:generate` · `db:migrate` · `db:seed` · `db:seed:admin` | Database lifecycle |
| `pnpm env:check` | Show configured env var **names** + active modes |
| `pnpm ai:eval` | Evaluate aesthetic classifier on the golden set |

---

## 7. Roadmap (build one phase at a time)

| Phase | Theme | Outcome |
|---|---|---|
| 0 | Foundations | Monorepo, CI, env validation, adapters + mocks, design system |
| 1 | Identity & seller onboarding | Auth (OTP), hardened admin (2FA/RBAC/audit), seller application + KYC, admin verification queue |
| 2 | Catalog & seller products | Aesthetic taxonomy, Snap & List, image pipeline, manual moderation |
| 3 | Storefront | Home, aesthetic hubs, PLP/PDP, search, SEO, PWA, wishlist |
| 4 | Cart, checkout & orders | Razorpay flow, webhooks, orders, seller inbox, invoices, refunds |
| 5 | AI verification & trends | Pipeline, decision engine, feedback to sellers, eval harness, trending ranking |
| 6 | Style Profile & Try-On | Consent/18+, skin-tone analysis, try-on, size & colour suggestions (feature-flagged) |
| 7 | Logistics, returns & payouts | Shiprocket, reverse pickup, Razorpay Route/ledger, seller finance |
| 8 | Hardening & launch | Security review, load/backup tests, legal pages, runbooks, staging→prod |
| 9 | Growth | Quiz, personalisation, drops, outfits, visual search, referrals… |

Details and acceptance criteria: `Project-Details.md §14`.

**Driving the agent:** paste the prompts in `AgentPrompt.md §9`. After each phase it will give you a completion report and the exact env vars to fill. Review, then reply **"proceed"**.

---

## 8. Deployment (overview)

- **Apps:** Vercel — one project per app (`storefront`, `seller`, `admin`), each with its own environment variables and domain. Admin gets its own domain + IP allowlist.
- **Worker:** Render / Railway / Fly (small instance).
- **DB & storage:** Supabase (upgrade plan before launch — free projects can pause when inactive).
- **Secrets:** set in each host's dashboard — never in the repo. Separate values for `staging` and `production`.
- **Migrations:** run via a controlled CI job (`pnpm db:migrate`), not at app boot.
- **Domains/DNS:** `xyz.com` → storefront, `seller.xyz.com` → seller, `admin.xyz.com` → admin; set `COOKIE_DOMAIN` appropriately; configure Razorpay/Shiprocket webhooks to production URLs.
- Full runbook: `/docs/runbooks` and `/docs/launch-checklist.md` (created in Phase 8).

---

## 9. Security & key rotation

- Admin: no public signup, mandatory 2FA, IP allowlist, short sessions, audit log.
- KYC documents and try-on photos live in **private** buckets, accessed via short-lived signed URLs; sensitive fields are encrypted at rest.
- **Rotation:** to rotate a secret, generate a new one → update it in the host dashboard → redeploy → revoke the old one at the provider. `AUTH_SECRET`/`ADMIN_AUTH_SECRET` rotation signs everyone out. Rotating `FIELD_ENCRYPTION_KEY` requires the re-encryption script documented in `/docs/runbooks`.
- If a secret leaks: revoke at the provider immediately, rotate, review logs/audit trail.
- Report vulnerabilities privately to `SUPPORT_EMAIL`.

---

## 10. Legal & compliance heads-up

This project handles payments, personal data, photos and KYC. Before launch, consult a **lawyer and CA** on: DPDP Act 2023 (consent, children's data, retention), Consumer Protection (E-Commerce) Rules, GST/TCS/TDS obligations for marketplace operators, seller agreements, IP/counterfeit takedown policy, and refund/return policy. The software is designed to be configurable for these, but does not constitute legal or tax advice.

---

## 11. Contributing & conventions

Conventional commits · TypeScript strict · PRs must pass lint/typecheck/tests · no secrets in code · update docs/ADRs with behaviour changes.

## 12. License

To be decided by the owner (add `LICENSE`). Until then: all rights reserved.

---

## ✅ Implementation status (built)

This repo now contains the working monorepo described above — all apps build with zero third-party keys:

| Piece | Status |
|---|---|
| `apps/storefront` | Home, aesthetic hubs (`/aesthetic/[slug]`), category + shop PLPs with URL-synced filters/sort, PDP with JSON-LD Product/Breadcrumb, search, weekly drops, find-your-aesthetic quiz, wishlist, cart, Razorpay checkout (mock without keys, Checkout.js with `NEXT_PUBLIC_RAZORPAY_KEY_ID`), orders timeline, 18+ consented virtual try-on (mock provider), phone-OTP account (demo code `123456` in console mode), FAQ/about/legal pages |
| `apps/seller` | Hinglish dashboard, Snap & List 3-step flow with AI pre-fill + verification result, products with friendly rejection reasons, orders with SLA cues, earnings with 12% commission split, KYC onboarding form |
| `apps/admin` | Overview, seller verification queue (approve/suspend), AI moderation queue with explainable scores + overrides, orders, finance/ledger + settlement mode, AI threshold settings + kill switch, append-only audit log, 2FA login shell, `noindex` everywhere |
| `apps/worker` | Job runner with the production contracts (product.verify, tryon.generate, payment.reconcile, settlement.release, trend.snapshot), idempotency keys, retry + dead-letter |
| `packages/*` | `core` (seed catalog: 16 aesthetics, 10 categories, 24 products, 5 sellers; paise pricing/GST/commission, order+product state machines, AI decision engine, trend ranking, size recommender, positive-only colour harmony), `validators` (Zod), `config` (env + brand), `payments` (Razorpay orders + HMAC signature/webhook verification + Route/manual settlement), `ai`, `shipping`, `notifications`, `storage`, `auth` (RBAC + OTP hashing), `db` (Drizzle schema + `migrations/0001_init.sql`), `ui` |
| SEO/GEO/AEO | SSR/SSG, per-page metadata + canonicals, OpenGraph/Twitter, JSON-LD (Organization, WebSite+SearchAction, FAQPage, Product, Breadcrumb, CollectionPage), `sitemap.xml`, `robots.txt`, PWA manifest, and `/llms.txt` for AI answer engines |
| Tests/CI | `pnpm test` (15 unit tests: pricing, GST, state machines, decision engine, RBAC/ownership, signatures), `pnpm ai:eval`, GitHub Actions (secret scan → test → eval → build) |

**Remaining for production (see `docs/backlog.md`):** provision Supabase and swap the seeded repository to Drizzle (ADR-003), pg-boss persistence, sharp upload pipeline, live Gemini/MediaPipe adapters, Sentry/PostHog SDKs, Playwright e2e, invoice PDFs. Demo credentials: shopper OTP `123456`, seller = Seller Hub (pre-verified demo), admin = `/login` demo shell (wire `ADMIN_BOOTSTRAP_*` for live).
