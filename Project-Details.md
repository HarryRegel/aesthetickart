# Project-Details.md — Gen Z Aesthetic Fashion Marketplace

> **Working title:** `BRAND_NAME` (placeholder — never hardcode; read from config/env). Domain placeholder: `xyz.com`.
> **One-line pitch:** A low-cost, mobile-first marketplace for *only* aesthetic / trending Gen Z fashion — sourced from Sarojini Nagar and local-market sellers — with AI that verifies every listing is genuinely on-aesthetic, and a virtual try-on so buyers can see it on themselves before paying.

This is the **single source of truth** for what to build. `AgentPrompt.md` tells the coding agent *how to work*; this file says *what to build*. If the two conflict, ask the human.

---

## 0. How to read this document

| Section | Purpose |
|---|---|
| 1–3 | Vision, principles, users |
| 4–5 | Architecture and tech stack (decisions already made) |
| 6 | Aesthetic taxonomy (the heart of the product) |
| 7 | Feature specs for the three faces (Storefront, Seller, Admin) |
| 8 | AI systems (verification, trends, try-on, recommendations) |
| 9 | Commerce engine (orders, Razorpay, payouts, shipping, returns, tax) |
| 10 | Data model |
| 11–12 | Security, compliance, non-functional requirements |
| 13 | Config & environment strategy |
| 14 | **Phased build plan (Phase 0 → 9)** |
| 15–17 | Testing/deploy, risks & open decisions, out-of-scope |

Priority tags used throughout: **[MVP]** must ship at launch · **[V1.1]** soon after launch · **[LATER]** nice to have.

---

## 1. Vision & positioning

**Problem.** Gen Z wants specific aesthetics (Y2K, coquette, old money, streetwear…) at prices they can afford. Marketplaces are flooded with generic clothing; discovering on-trend pieces is slow, sizing/fit is a gamble, and returns are costly. Meanwhile local market sellers (Sarojini Nagar, Lajpat Nagar, Janpath, Commercial Street, etc.) hold exactly these pieces cheaply but have no online reach or tooling.

**Solution.** A curated, aesthetic-only marketplace where:
1. Only on-aesthetic products get listed (AI + human verification).
2. Shopping is organised by **aesthetic**, not just by garment type.
3. Buyers can **try on virtually** and get colour/size guidance, reducing returns.
4. Local sellers get a **ridiculously easy** mobile listing flow and reliable payouts.

**Differentiators (protect these):**
- Aesthetic-first discovery ("Shop by aesthetic", "Find your aesthetic" quiz).
- AI listing gatekeeper + AI-assisted listing creation for sellers.
- Virtual try-on + skin-tone-aware colour suggestions.
- Low price point (value collections: under ₹499 / ₹799 / ₹999).

**Success metrics (instrument from day one):** seller activation rate (signup → first approved product), listing approval time, AI-vs-human decision agreement, storefront conversion, try-on usage → conversion lift, return rate vs. baseline, repeat purchase rate, average order value.

---

## 2. Product principles

1. **Aesthetic or nothing.** Plain/basic/generic items are rejected unless they clearly belong to an aesthetic.
2. **Mobile-first, low-bandwidth.** Primary users are on mid-range Android over 4G. LCP < 2.5 s, small JS, WebP/AVIF images, PWA.
3. **Dead-simple for sellers.** Many sellers are non-technical. Big buttons, Hinglish option, phone-camera upload, WhatsApp notifications.
4. **Trust & safety by default.** Verified sellers only, no counterfeits, honest AI disclaimers, private user photos.
5. **Low cost to run.** Free/cheap tiers, hosted AI APIs (no self-hosted ML), aggressive caching, hard budget caps on AI spend.
6. **Auditable.** Every sensitive admin action and AI decision is logged and explainable.
7. **Provider-agnostic.** Every external service (payments, AI, SMS, shipping, storage) sits behind an adapter with a `mock` mode so the app runs without keys.

---

## 3. Users & personas

| Persona | Description | Key needs |
|---|---|---|
| **Shopper (Gen Z, 16–28)** | Instagram/Pinterest native, price-sensitive, UPI-first | Discover trending aesthetics, trust the fit, cheap, fast checkout |
| **Seller (market shop owner / supplier)** | Sarojini Nagar etc.; low-tech, phone-only, Hindi/Hinglish | Easy listing, orders on WhatsApp, quick payouts, clear rejection reasons |
| **Admin / Ops** | Founders + small team | Verify sellers, review flagged products, handle disputes/refunds, see everything |
| **Support / Finance / Curator** (sub-roles of admin) | Limited-permission staff | Scoped access only |

> **Minors note:** Gen Z includes under-18s. Browsing/shopping may be allowed per legal advice, but **try-on photo features require 18+ confirmation** (see §11 — India's DPDP Act requires verifiable parental consent for children's data). Get legal advice before launch.

---

## 4. System architecture

### 4.1 The three faces + background worker

| App | Host (prod) | Audience | Notes |
|---|---|---|---|
| `apps/storefront` | `xyz.com` | Public shoppers | SEO-critical, SSR/ISR, PWA, fastest app |
| `apps/seller` | `seller.xyz.com` | Shop owners / suppliers | Mobile-first, Hinglish, behind seller auth, `noindex` |
| `apps/admin` | `admin.xyz.com` (or an unguessable host) | Internal team | **Separate deployable**, IP allowlist, mandatory 2FA, RBAC, audit log, `noindex` |
| `apps/worker` | internal | — | Background jobs: AI verification, try-on, webhooks processing, notifications, settlement, trend snapshots |

Admin is a **separate app/deployment** (not just a route group) so it has its own session cookie, CSP, network rules, and blast radius.

### 4.2 Monorepo layout (pnpm workspaces + Turborepo)

```
/
├─ apps/
│  ├─ storefront/        # Next.js (App Router)
│  ├─ seller/            # Next.js
│  ├─ admin/             # Next.js
│  └─ worker/            # Node service (pg-boss jobs + webhook handlers)
├─ packages/
│  ├─ db/                # Drizzle schema, migrations, seed scripts
│  ├─ core/              # Domain services (pricing, order state machine, commission, inventory)
│  ├─ auth/              # Session, OTP, TOTP, RBAC helpers
│  ├─ validators/        # Zod schemas shared by all apps
│  ├─ ui/                # Design system (Tailwind + shadcn/ui), shared components
│  ├─ ai/                # Adapters: vision, embeddings, try-on, moderation (+ mocks)
│  ├─ payments/          # Razorpay adapter (orders, verify, webhooks, Route transfers, refunds)
│  ├─ shipping/          # Shiprocket (or other) adapter + mock
│  ├─ notifications/     # Email / SMS / WhatsApp adapters + templates
│  ├─ storage/           # Storage adapter (Supabase Storage default, R2 option) + signed URLs
│  └─ config/            # env validation (zod), tsconfig, eslint, tailwind presets, brand config
├─ docs/                 # ADRs, runbooks, API notes, aesthetic guide
├─ .env.example          # Canonical list of ALL env vars (see §13)
├─ AgentPrompt.md  Project-Details.md  README.md
```

### 4.3 High-level data flow

```
Seller (mobile) ──upload──▶ seller app ──▶ storage + DB (product: pending_ai)
                                            │ enqueue
                                            ▼
                                      worker: verification pipeline
                      (quality → safety/IP → aesthetic classify → trend score → decision)
                                            │
              auto-approve ◀────────────────┼────────────────▶ auto-reject (reasons to seller)
                    │                       ▼
                    │                 admin review queue
                    ▼
               product LIVE ──▶ storefront (search, hubs, trending) ──▶ cart ──▶ Razorpay ──▶ order
                                                                              │
                         try-on (user photo + product image) ─▶ worker ─▶ result ┘
                                                                              ▼
                          sub-orders per seller ─▶ shipping ─▶ delivery ─▶ return window ─▶ payout (Route)
```

### 4.4 Architectural rules

- **Domain logic lives in `packages/core`**, never in UI components or route handlers. Apps are thin.
- **Zod validation at every boundary** (forms, route handlers, webhooks, job payloads, env).
- **Async by default for slow things** (AI, try-on, notifications, settlement) via the job queue. UI shows job status (poll or SSE).
- **Idempotency everywhere money or webhooks are involved** (idempotency keys, unique constraints on provider event IDs).
- **State machines** for orders, products, seller accounts, try-on jobs — transitions are explicit and logged.
- **Feature flags** (DB-backed) for COD, try-on, visual search, etc.

---

## 5. Tech stack (decisions made — change only with an ADR in `/docs`)

| Concern | Choice | Why / notes |
|---|---|---|
| Language | TypeScript (strict) everywhere | One language, shared types |
| Web framework | Next.js (App Router), React | SSR/ISR for SEO, server actions + route handlers |
| Styling/UI | Tailwind CSS + shadcn/ui + Radix | Fast, accessible; shared in `packages/ui` |
| DB | **PostgreSQL (Supabase free tier by default)** + `pgvector` | Relational + embeddings in one DB; no extra infra |
| ORM/migrations | Drizzle ORM + drizzle-kit | Type-safe, good pgvector support |
| Queue / jobs | **pg-boss** (Postgres-backed) | No Redis required for jobs; zero extra cost |
| Cache / rate limit | Upstash Redis (free tier) | Serverless-friendly rate limiting & caching |
| Storage | Supabase Storage by default behind `packages/storage` adapter; Cloudflare R2 supported | Public bucket for product images, **private bucket** for KYC docs & user try-on photos (signed URLs) |
| Image processing | `sharp` (resize → WebP/AVIF, strip EXIF) | Generate responsive variants on upload |
| Auth | Server-side DB sessions, httpOnly cookies (Better Auth or Auth.js v5 — agent's pick, documented in an ADR) | Separate cookie names/lifetimes per app |
| OTP / SMS | MSG91 (India, DLT-compliant) behind adapter; `console` mode for dev | Phone OTP is the primary Indian login |
| Email | Resend (free tier) behind adapter | Transactional email |
| WhatsApp | Meta WhatsApp Cloud API behind adapter (optional, V1.1) | Seller order alerts |
| Payments | **Razorpay** Standard Checkout + **Razorpay Route** for marketplace split/settlement | See §9 |
| Shipping | Shiprocket API behind adapter (mock in dev) | Pincode serviceability, labels, tracking webhooks |
| AI – vision/classification | Hosted multimodal LLM (default **Google Gemini** Flash-tier via API; provider-agnostic) | Cheap, structured JSON output, free tier exists |
| AI – embeddings | Hosted image-embedding API → stored in `pgvector` | Similarity to aesthetic reference sets, duplicate detection, "similar items" |
| AI – try-on | Pluggable provider (`fashn`, `fal`, `replicate`, `gemini`, `mock`) | Cost-sensitive; see §8.3 |
| AI – face/skin analysis | **MediaPipe Face Landmarker in the browser (WASM)** | Runs client-side: free, private, no raw-face server calls for analysis |
| Search | Postgres FTS + `pg_trgm` + filters (MVP); Meilisearch/Typesense later | Avoid extra infra early |
| Observability | Sentry (errors), PostHog (product analytics), structured logs (pino) | Free tiers |
| i18n | `next-intl` — English + Hindi (seller app Hinglish-friendly copy) | |
| Testing | Vitest (unit), Playwright (e2e), Testcontainers *not required* | |
| CI/CD | GitHub Actions: lint, typecheck, test, build, migrate | |
| Hosting | Vercel (3 Next apps) + Render/Railway/Fly (worker) + Supabase | Start on free tiers |

### 5.1 Development-environment constraints (important)

The developer may be on a **low-spec machine (e.g., 8 GB RAM)** and prefers **free API keys that need no credit card upfront**. Therefore:
- **No local ML models, no Docker requirement.** All AI is via hosted APIs; DB is a hosted free Postgres.
- Each app must run **independently** (`pnpm dev:storefront`, `pnpm dev:seller`, `pnpm dev:admin`, `pnpm dev:worker`). Do not require all four at once. Prefer Turbopack and lean dev tooling.
- Every external provider has a **`mock` mode** (flags in §13) so development can proceed with zero keys; real keys are added progressively.
- Razorpay **test mode** keys are free; use them throughout development.
- Document, per provider, whether a card is required for signup — and offer a no-card alternative or mock where it is.

---

## 6. The Aesthetic Taxonomy

Aesthetics are **data, not code** — stored in the `aesthetics` table, managed by admin, versioned. Each has: `slug`, `name`, `group` (`classic` | `modern`), `description` (used in AI prompts), `visual_keywords`, `color_palette`, `typical_garments`, `reference_images[]` (embedding seeds), `trend_weight` (0–1, admin/automated), `status`.

**Seed set (admin can add/retire):**

*Classic aesthetics*: Old Money / Quiet Luxury · Vintage / Retro (60s–90s) · Preppy · Dark Academia · Light Academia · Cottagecore · Boho / Hippie · Minimalist / Clean Girl · Parisian Chic · Indo-Western Fusion · Traditional-Modern Ethnic (curated, aesthetic-only).

*Modern (current Gen Z) aesthetics*: Y2K · Streetwear · Baggy / Skater · Grunge · Coquette · Balletcore · Gorpcore / Outdoors · Techwear · Baddie · Indie Sleaze · K-fashion / Korean Street · Mob Wife · Athleisure-Aesthetic · Cyber / Alt · Downtown Girl · Soft Girl.

**Top-level shopping dimensions** (a product has one category and ≥1 aesthetic):
- **Aesthetic** (primary navigation).
- **Category**: Tops, T-shirts & Baby Tees, Shirts, Co-ords, Dresses, Skirts, Bottoms (jeans/cargos/trousers), Jackets & Outerwear, Sweaters & Knits, Ethnic-Fusion, Accessories (bags, belts, scarves, jewellery — *accessories are no-try-on*), Footwear (LATER).
- **Gender/Fit**: Women, Men, Unisex.
- **Price bands**: Under ₹499 · ₹500–999 · ₹1000–1999 · ₹2000+.
- **Other filters**: size, colour, fabric, occasion, new arrivals, trending, seller location/market.

**Aesthetic Guide** (`/docs/aesthetic-guide.md`): the agent must create a human-readable guide per aesthetic (what qualifies, what doesn't, example descriptors). It feeds sellers (listing guidance), admins (review consistency) and AI prompts.

---

## 7. Feature specification by face

### 7.1 FACE 1 — Storefront (`xyz.com`) — shopper-facing

#### Discovery
- **[MVP] Home**: hero banners (admin-curated), "Trending now", "New drops", "Shop by aesthetic" tiles, price-band shortcuts, category grid, recently viewed, curated collections.
- **[MVP] Aesthetic hubs** (`/aesthetic/[slug]`): intro, mood imagery, products, sub-filters.
- **[MVP] Category pages** (`/c/[category]`) and **listing page** with filters (aesthetic, category, price, size, colour, fabric, gender), sort (trending, new, price asc/desc), infinite scroll or pagination, URL-synced filters.
- **[MVP] Search**: typo-tolerant text search (FTS + trigram), suggestions, recent searches, zero-result fallbacks.
- **[V1.1] "Find your aesthetic" quiz**: 6–8 image-based questions → top-3 aesthetics → personalised feed.
- **[V1.1] Weekly Drops**: time-boxed capsule collections with countdown.
- **[V1.1] Shop the look / Outfit boards**: curated outfits made of live products ("complete the look").
- **[LATER] Visual search**: upload a Pinterest/Instagram screenshot → similar products (reuses embeddings).

#### Product detail page (PDP)
- Image gallery (zoom, swipe), title, price, MRP, discount %, GST-inclusive note, seller/shop badge (name, market location, rating) — *required by Indian e-commerce rules to show seller details*.
- Size selector with **size guide** (garment measurements supplied by seller) and **size recommendation** (from user's body params).
- Aesthetic chips (clickable), fabric/care, delivery ETA by pincode, return/exchange policy summary.
- **Try-on button** (if `tryon_ready`, feature enabled, user eligible) — see §8.3.
- **"Colours that suit you"** badge (if the user has a style profile) — positive-only language.
- Reviews with photos, Q&A (V1.1), "similar items" (embedding kNN), "more from this aesthetic".
- Add to cart / Buy now / Wishlist / Share (WhatsApp, Instagram).

#### Account
- **[MVP] Login/signup**: phone OTP (primary), Google (secondary). Guest browsing allowed; cart persists across login.
- Profile, saved addresses, **order history & tracking**, wishlist, returns/exchanges, notification prefs.
- **Style Profile** (for try-on): photo(s), height, usual size, body type (optional), skin tone/undertone (auto-detected, user-editable), consent controls, **delete my photo/data** button.
- **[V1.1]** Referral code, price-drop & back-in-stock alerts, saved looks.

#### Cart & checkout (see §9 for engine)
- **[MVP]** Multi-seller cart (items grouped by seller visually, single payment). Stock validation, price-change warnings, coupon code, free-shipping progress bar.
- **[MVP]** Address (pincode autofill + serviceability), contact verification, order summary with GST, shipping, discounts.
- **[MVP]** **Razorpay Checkout** (UPI, cards, netbanking, wallets). Handle success, failure, retry, abandoned payment, and webhook-confirmed final state.
- **[V1.1]** COD (feature-flagged, risk-controlled).
- Order confirmation page + email/SMS/WhatsApp; **order tracking page** with timeline.
- **[MVP]** Cancel before seller-ship; return/exchange request flow with reason + photos.

#### Other pages / trust
- About, How try-on works, Aesthetic guide (public lite), Size guide, Shipping, **Returns/Refund policy**, Terms, Privacy Policy (DPDP-aligned), Seller T&Cs, **Grievance Officer contact** (required), Contact/Support (form + WhatsApp link), FAQ.
- SEO: SSR, canonical URLs, Open Graph, JSON-LD (`Product`, `BreadcrumbList`, `Organization`), sitemap, robots, clean slugs. PWA manifest + install prompt. Cookie/consent banner for analytics.

### 7.2 FACE 2 — Seller Platform (`seller.xyz.com`)

#### Onboarding & verification
- **[MVP] Signup**: phone OTP + (optional) email/password. Language toggle (English / हिंदी-Hinglish).
- **[MVP] Seller application** (multi-step, save-and-resume): shop name, owner name, market/location (Sarojini Nagar, Lajpat Nagar, Janpath, Karol Bagh, other—free text), shop photos, pickup address, categories sold, Instagram link (optional), ID proof + PAN, GSTIN (optional if below threshold — flag for admin), bank/UPI details for payouts, T&C + commission agreement acceptance (store version + timestamp + IP).
- **[MVP] Status tracker**: `draft → submitted → under_review → approved | rejected(with reasons, can resubmit) | suspended`. WhatsApp/SMS/email on each change.
- KYC docs go to the **private bucket**, encrypted metadata, viewed only by authorised admins via short-lived signed URLs.
- Payout account creation via **Razorpay Route linked account** (see §9.4).

#### Product management
- **[MVP] "Snap & List" mobile flow**: take/upload 3–8 photos (guided: front, back, detail, flat-lay on plain background for try-on) → AI pre-fills title, description, category, colours, fabric guess, aesthetic suggestions → seller reviews/edits → sets price/MRP, sizes, stock per size, garment measurements, → submit.
- **[MVP]** Full form alternative + **bulk upload (CSV + zip of images)** [V1.1 for bulk].
- **[MVP] Listing statuses**: `draft → pending_ai → pending_review → live | rejected(reasons+tips) → resubmit`; plus `paused`, `out_of_stock`, `removed`.
- **[MVP] Instant, human-readable AI feedback**: "Rejected: looks like a plain unbranded white tee with no styling — doesn't match any supported aesthetic. Tip: pair as an outfit / use a baby-tee cut…" with photo-quality tips.
- Edit price/stock quickly (inline), bulk pause/resume, duplicate listing, variant management (size × colour, SKU).
- **Try-on readiness hint**: "Add one clear front-facing photo on a plain background to enable Try-On (more sales)."
- **Photo helper**: on-device checks (blur, brightness, resolution) *before* upload; auto-background clean-up option [V1.1].

#### Orders & fulfilment
- **[MVP]** Order inbox (new / to pack / shipped / delivered / returns), **confirm within SLA (default 24 h)** else auto-cancel + refund + seller penalty tracking.
- **[MVP]** Packing slip & shipping label (via shipping adapter), mark packed/ready, pickup scheduling, tracking visible.
- **[MVP]** Return/exchange handling: accept/dispute with evidence; outcomes decided per policy + admin escalation.
- **[MVP]** Notifications: WhatsApp (V1.1) / SMS / email / in-app for new order, cancellation, return, payout.

#### Money
- **[MVP]** Earnings dashboard: pending (in return window), available, paid; per-order breakdown (item total, commission, shipping, GST/TCS notes, net). Payout history & statements (CSV).
- **[V1.1]** Sales analytics: views, add-to-cart, conversion, top aesthetics, "what's trending — list more of this" nudges.

#### Account & quality
- Shop profile page content (public shop page on storefront: `/shop/[slug]`), trust level (`new → verified → trusted`), rating, strikes/penalties display, support chat/ticket, learning hub ("How to photograph clothes", "Aesthetic guide").

### 7.3 FACE 3 — Admin Console (`admin.xyz.com`) — super-secure

#### Access & security (see also §11)
- **No public signup.** Admins are created by a super-admin invite or by a CLI seed script.
- **Mandatory TOTP 2FA** (+ recovery codes); optional WebAuthn passkeys. Short idle timeout (30 min), absolute max (8–12 h), **re-authentication for destructive actions** (suspend seller, refund > threshold, delete, change AI thresholds, export data).
- **IP allowlist** (env-driven) + rate limiting + lockout. Separate cookie/CSP. `noindex`.
- **RBAC roles**: `super_admin`, `ops` (seller + product verification), `support` (orders/users, limited), `finance` (refunds, payouts, settlements), `curator` (content/aesthetics/banners). Permission matrix defined in code and unit-tested.
- **Immutable audit log**: who, what, when, IP, before/after diff, reason. Viewable & filterable; append-only (DB-level protections).

#### Modules
1. **Dashboard**: GMV, orders, AOV, new sellers, approval queue sizes, AI auto-approve/reject rates, try-on usage & spend vs. budget, payment failure rate, return rate, top aesthetics, alerts.
2. **Seller verification**: queue with KYC docs viewer, checklist, risk flags (duplicate PAN/phone/bank, mismatched names), approve / request changes / reject (templated reasons) / **suspend / permanently remove**, set trust level & commission override, notes, history.
3. **Product moderation**: queue of `pending_review` with AI verdict side-by-side (scores, detected aesthetics, flags, reasoning), approve/reject/edit tags, bulk actions, **takedown** (counterfeit, IP, policy) with reason → seller strike; override feedback is stored as labelled data for the AI feedback loop (§8.1).
4. **Aesthetic & taxonomy manager**: CRUD aesthetics/categories, upload reference images, edit descriptions/keywords, set trend weights, merge/retire, **AI threshold config** (auto-approve/review/reject) with change history.
5. **Trends board**: manual "trending now" curation + system-computed trend scores + external signal notes.
6. **Orders & disputes**: search/filter, full timeline, intervene (cancel, refund, re-ship), dispute resolution workflow between buyer and seller.
7. **Payments & settlements (finance)**: Razorpay payments/refunds ledger, Route transfer status, hold/release payouts, reconcile, export, commission settings, reports for CA (GST/TCS views).
8. **Users**: search, view (PII minimised/masked by default; reveal requires permission + audit), block/unblock, delete-on-request workflow (DPDP), try-on data purge.
9. **Content/CMS**: banners, homepage sections, collections/drops, outfit boards, static pages, announcements.
10. **Reviews & reports**: moderate reviews, handle user/seller reports.
11. **Try-on ops**: usage, failures, cost per job, provider health, kill switch, per-user/day limits, monthly budget cap.
12. **System**: feature flags, settings (commission %, shipping rules, SLA hours, return window), webhook/job monitor (failed jobs, retry), email/SMS templates, admin user management, audit log.

---

## 8. AI systems

> General AI rules: all calls go through `packages/ai` adapters (`mock` | provider). Every call is **logged** (model, version, latency, tokens/cost, input hash — never raw private photos in logs), **cached by image hash**, **rate-limited**, and covered by a **monthly budget circuit breaker** that degrades gracefully to "manual review"/"try-on temporarily unavailable". Prompts are versioned in the repo. Structured (JSON-schema) outputs only; validate with Zod; retry/repair once; never trust raw model text.

### 8.1 Aesthetic & Trend Verification Pipeline (the gatekeeper)

Triggered when a seller submits a product (and on edits that change images/title/tags). Runs in the worker. Persisted as a `verification_run` with per-stage results.

**Stage 1 — Technical quality** (cheap, deterministic): min resolution (≥ 800 px short side), blur score (Laplacian variance), brightness, aspect ratio, min 3 images with ≥1 clear front image, file type/size limits, EXIF stripped. Fail → *auto-reject with fix tips* (no AI cost spent).

**Stage 2 — Safety & IP**: NSFW/nudity/minor-safety moderation; visible-watermark/stolen-image check (**perceptual hash** vs. existing catalog + other sellers → duplicate/stolen flag); prominent third-party brand logos/counterfeit risk flag; prohibited-item check (text + image). Hard fails → reject/escalate to admin (never auto-approve).

**Stage 3 — Aesthetic classification** (hybrid, two independent signals):
- **VLM judge**: multimodal model receives images + title/description + the *aesthetic guide* and returns strict JSON:
  `{garment_type, colors[], pattern, fabric_guess, detected_aesthetics:[{slug, confidence, reasoning}], is_generic_basic: bool, styling_notes, try_on_suitability, quality_issues[], safe: bool}`.
- **Embedding similarity**: image embedding → kNN against per-aesthetic reference sets (`pgvector`) → similarity scores per aesthetic.
- **Combine**: `aesthetic_score = w1·vlm_confidence + w2·embedding_similarity` (weights in settings, default 0.6 / 0.4). Penalise `is_generic_basic`. Compare seller-declared aesthetics vs. detected; mismatch → flag.

**Stage 4 — Trend relevance**: `trend_score` from aesthetic `trend_weight`, recency of the style, platform signals (views/wishlist/orders velocity for similar items), and admin trend board. Used for ranking, not rejection (except "obsolete & off-aesthetic").

**Decision engine** (thresholds in settings, editable by super-admin):
| Condition | Outcome |
|---|---|
| Hard safety/IP fail | `rejected` or `pending_review` flagged high-risk |
| `aesthetic_score ≥ 0.80` **and** seller trust ≥ `verified` **and** no flags | **auto-approve → live** |
| `0.50 ≤ score < 0.80`, or new seller (first N=10 listings), or any soft flag | **human review queue** |
| `score < 0.50` or generic basic | **auto-reject** with specific, actionable reasons |

**Explainability:** store reasons, scores, model + prompt versions; show to seller (friendly) and admin (detailed).

**Feedback loop & evaluation:**
- Build a **golden set** (≥ 300 labelled images across aesthetics, with hard negatives) → script `pnpm ai:eval` reports precision/recall/agreement per aesthetic and per model/prompt version. Gate prompt/model changes on it.
- Every admin override is stored; periodically mined into few-shot examples and new reference images.
- Track AI-vs-human agreement on the dashboard; auto-approve threshold must be raised/lowered by humans based on data, not guesses.

**Also:** AI-assisted listing (title/description/tags generation) shares the same VLM call where possible to save cost; seller always confirms.

### 8.2 Trending / ranking engine
- `trend_snapshots` computed by a scheduled job (hourly/daily): per product & per aesthetic → velocity of views, wishlist adds, add-to-cart, orders (time-decayed), combined with `trend_weight` and a freshness boost.
- Feeds: "Trending now", home ranking, hub ordering. Admin can pin/boost/bury.
- Personalisation [V1.1]: user's quiz aesthetics + browsing → re-rank. Cold-start safe (fallback to global trending).

### 8.3 Virtual Try-On & Style Profile

**Goal:** let a shopper see a product on themselves so they can buy confidently.

**Eligibility & consent (hard requirements):**
- Logged-in users only; **explicit 18+ confirmation**; explicit, granular, revocable consent (what is collected, why, retention); a plain-language privacy notice; default = **photo is NOT stored** unless the user opts in to "save my photo for faster try-ons".
- **No face recognition / no biometric templates / no use for model training.** Never expose photos to sellers or admins' general views (admin access only for abuse investigations, audited).
- Only the user's own photo is permitted (ToS + moderation: reject nudity, minors-appearing subjects, multiple people, public figures where detectable).
- One-tap **delete all my photos & try-on results**; auto-expiry (default 30 days; env-configurable).

**Style Profile capture (client-side where possible):**
1. Photo capture/upload with guidance overlay (front-facing, good light, plain background, arms slightly away).
2. **In-browser** MediaPipe Face Landmarker → face present, single person, pose/blur/brightness checks; sample skin pixels (cheek/forehead regions, lighting-normalised) → estimate **skin tone** (map to a standard scale e.g., Monk 10-point) and **undertone** (warm/cool/neutral). Show the result with a **manual correction slider** — never force.
3. Optional inputs: height, weight range, usual size, body type, fit preference (fitted / regular / oversized), hair colour.
4. Store only derived values (tone, undertone, measurements) + (optionally) the photo in the **private bucket**.

**Try-on generation (server/worker):**
- Inputs: user photo + product's *try-on source image* (flat-lay or clean model shot, background removed) + garment category (tops / bottoms / dresses / outerwear; **accessories & footwear excluded in MVP**).
- Provider adapter returns an image (or job id → poll/webhook). Show a progress UI; **low-res preview first** if the provider supports it.
- **Caching:** key = `hash(user_photo_id, product_image_id, provider_version)`; repeat views are free.
- **Cost controls:** per-user daily limit (default 5), global daily cap, **monthly budget circuit breaker** (kill switch → "Try-on is resting, check back soon"), usage and cost logged per job, admin dashboard.
- **Failure UX:** clear reasons ("we couldn't find your full torso — try a different photo"), retry, auto-fallback provider (optional), never charge/penalise.
- **Honesty UX:** persistent label *"AI preview — real fit, drape and colour may differ."* Show it next to size recommendation, not as a replacement.
- Results are private (signed URLs), downloadable/shareable by the user only; watermark on shared images.

**Size recommendation (rules-based, MVP):** compare user's height/weight/usual size/fit preference with the seller-supplied garment measurements & size chart → "Likely your size: M (relaxed fit)" with confidence + explanation. Improve using return-reason data later.

**"Colours that suit you" (positive-only, MVP):** from skin tone + undertone → palette suggestions; per product compute a *colour harmony* match with the product's dominant colours → show a soft badge ("Great with your tone") — **never** show negative judgments ("bad on you"). Make this clearly a suggestion, inclusive across all skin tones, and reviewed for bias with diverse test images.

**Seller side for try-on:** `tryon_ready` is computed from image checks (single garment, clean background, front-facing, resolution, category supported). Sellers see a checklist; admin can force-disable per product.

**Quality bar before launching the feature publicly:** run internal evaluation on a diverse set of photos (skin tones, body types, garment categories); document failure modes; ship behind a feature flag and roll out gradually. If provider quality/cost isn't acceptable, ship Phase 6 as "beta" or defer — do not block marketplace launch on it.

### 8.4 Recommendations & search intelligence [V1.1]
- "Similar items" via embedding kNN; "complete the look" via outfit boards + co-purchase; synonym map (e.g., "baby tee" ⇄ "crop tee"); query → aesthetic mapping ("coquette bow top" → coquette).

---

## 9. Commerce engine

### 9.1 Pricing & catalog rules
- Price stored in **paise (integers)**; MRP, selling price, GST rate by HSN (configurable table — **confirm slabs with a CA**), discount %. Prices displayed GST-inclusive.
- Product → variants (size, colour, SKU, stock). Inventory per variant. **Stock reservation** during checkout (default 10 min TTL), released on failure/expiry.

### 9.2 Cart → Order model
- One **Order** (buyer-facing, one payment) → many **SellerOrders** (one per seller, each with its own shipment, SLA, returns, payout) → **OrderItems**.
- Order status machine (parent derived from children):
  `pending_payment → paid → partially_confirmed/confirmed → packed → shipped → out_for_delivery → delivered → completed (after return window)`; branches: `cancelled`, `payment_failed`, `return_requested → return_approved → return_in_transit → returned → refunded`, `exchange_*`.
- Seller SLA: confirm ≤ 24 h, ship ≤ 48 h (configurable). Breach → auto-cancel + full refund + seller strike.

### 9.3 Razorpay integration (Standard Checkout)
**Rules — the agent must read the current official Razorpay docs and not guess API shapes.**
1. Server creates a Razorpay **Order** (amount in paise, `receipt` = our order id, `notes`), stores `razorpay_order_id`. For marketplace splits, include **Route `transfers`** (see 9.4).
2. Client opens **Checkout.js** with the *public* key id and order id.
3. On success callback, client posts `razorpay_payment_id`, `razorpay_order_id`, `razorpay_signature` → server **verifies HMAC-SHA256 signature** using the key secret (constant-time compare). Mark "pending confirmation".
4. **Webhooks are the source of truth** (`payment.captured`, `payment.failed`, `order.paid`, `refund.processed`/`refund.failed`, `transfer.*` events as available): verify `X-Razorpay-Signature` against raw body with the webhook secret, store event id (unique) for **idempotency**, process in worker, handle out-of-order/duplicate delivery.
5. Handle: user closes modal, payment failed/retry, abandoned orders (expire & release stock), amount tampering (always recompute server-side), double submit, network drop after payment (reconciliation job polls Razorpay for orders stuck in `pending_payment`).
6. **Refunds** via API (full/partial) with ledger entries; refund to original method.
7. Test with Razorpay **test mode** keys and documented test cards/UPI; provide a Playwright e2e of the happy + failure paths.

### 9.4 Marketplace settlement (Razorpay Route)
- Each approved seller gets a **linked account** (Route) created via API with their business/bank details (Razorpay runs its own KYC on linked accounts; mirror its status in our DB).
- Payment split: platform commission retained; seller share transferred with **hold until delivered + return window** (e.g., `on_hold` with release by settlement job, using the Route API's hold/release capabilities).
- **Commission**: global % (default placeholder 12 %, configurable), per-category and per-seller overrides, min fee. Shipping fee policy configurable (e.g., free over ₹799).
- **Ledger** (double-entry style `ledger_entries`): payment in, commission, shipping, tax components, refunds, transfers, adjustments → reconcilable; reports exportable.
- **Verify with Razorpay** that Route is enabled on the merchant account (it requires activation/approval) — if unavailable at launch, fall back to *manual weekly payouts* driven by the ledger (admin finance module) behind the same `SettlementService` interface.

### 9.5 Shipping & logistics
- Shiprocket (or similar aggregator) adapter: **pincode serviceability + ETA**, rate quote, create shipment, generate label/AWB, schedule pickup, **tracking webhooks** → update order timeline.
- Default model: seller ships from their shop using platform-generated labels/pickups. Admin can switch per seller.
- Weight/dimension defaults per category; return shipments (reverse pickup) supported.
- **COD [V1.1, flag OFF]**: only below a ₹ cap, serviceable pincodes, OTP/WhatsApp confirmation, prepaid incentive, per-user risk score (RTO history), COD fee configurable.

### 9.6 Returns, exchanges, refunds
- Default (configurable) policy: **7-day return/exchange for wrong size/defect/not-as-described**; non-returnable categories configurable (innerwear, accessories, final-sale).
- Flow: request (reason + photos) → seller/admin decision → reverse pickup → QC → refund/exchange → ledger adjustment. Track **return reason codes** (size, quality, colour mismatch, "looks different from try-on") → analytics + seller quality score.

### 9.7 Tax & compliance (flag for CA/lawyer — do not hardcode assumptions)
- GST rate/HSN configurable; invoices: generate **tax invoice per seller order** (PDF) including GSTIN where applicable; platform commission invoice to seller.
- E-commerce operator obligations (e.g., TCS under GST, TDS) — design the ledger to capture the data needed; **do not implement tax filing**, but export reports.
- Displays required by Indian e-commerce rules: seller name/address, country of origin, MRP, return policy, grievance officer.

### 9.8 Reviews & ratings
- Verified-purchase only, star + text + photos, moderation queue, seller rating aggregation, abuse reporting. Incentives only if disclosed (LATER).

### 9.9 Notifications matrix
| Event | Buyer | Seller | Admin |
|---|---|---|---|
| Order placed/paid | email+SMS/WA | WA/SMS/in-app | — |
| Shipped / out for delivery / delivered | email+SMS/WA | in-app | — |
| Return requested/decided | email | WA/in-app | queue if disputed |
| Seller application status | — | WA/SMS/email | queue |
| Product approved/rejected | — | WA/in-app (with reasons) | — |
| Payout released | — | WA/email | — |
| Try-on ready (long jobs) | in-app | — | — |
| System alerts (budget, failures) | — | — | email/in-app |

Templates are versioned; all sends logged (`notifications` table) and deduped.

---

## 10. Data model (logical — agent designs exact Drizzle schema)

**Identity & access:** `users`, `customer_profiles`, `addresses`, `sessions`, `otp_challenges`, `seller_accounts`, `sellers_team_members` [LATER], `admin_users`, `admin_roles`, `admin_permissions`, `admin_totp`, `admin_recovery_codes`, `audit_logs` (append-only).

**Seller & KYC:** `shops`, `seller_kyc_documents` (encrypted refs), `seller_bank_accounts`, `razorpay_linked_accounts`, `seller_agreements` (version, accepted_at, ip), `seller_strikes`, `seller_trust_events`.

**Catalog:** `aesthetics`, `aesthetic_reference_images` (+embedding), `categories`, `products`, `product_variants`, `product_images` (+`phash`, `embedding vector`, `kind`, `tryon_source` flag), `product_aesthetics` (source: seller/ai/admin, confidence), `product_search` (tsvector), `collections`, `collection_items`, `outfits`, `outfit_items`, `banners`.

**AI:** `verification_runs` (stage JSON, scores, decision, model/prompt versions), `review_queue_items`, `ai_calls` (cost/latency log), `ai_overrides` (human corrections), `trend_snapshots`, `golden_set_items`.

**Try-on:** `tryon_profiles` (consent version/time, 18+ confirmation, derived tone/undertone, measurements, photo asset ref nullable), `tryon_jobs` (status, provider, cost, result asset, cache key, expires_at), `tryon_consents`.

**Commerce:** `carts`, `cart_items`, `stock_reservations`, `orders`, `seller_orders`, `order_items`, `payments`, `payment_events` (webhook idempotency), `refunds`, `shipments`, `shipment_events`, `returns`, `return_items`, `settlements`, `transfers`, `ledger_entries`, `coupons`, `coupon_redemptions`, `invoices`.

**Engagement:** `wishlists`, `reviews`, `review_photos`, `reports`, `recently_viewed`, `search_events`, `product_events` (views/clicks), `notifications`, `referrals` [V1.1].

**Platform:** `feature_flags`, `settings` (typed, versioned, audited), `jobs` (pg-boss tables), `webhook_events`.

**Data rules:** UUID v7 PKs; `created_at/updated_at`; soft-delete where legal retention requires; money in integer paise; all FK + indexes planned for listing/search paths; migrations only via drizzle-kit; seed script creates aesthetics, categories, demo sellers/products, a demo admin (credentials from env), and golden-set scaffolding.

---

## 11. Security & compliance

**Application security**
- OWASP ASVS L2 as a target. Strict **CSP**, HSTS, secure/httpOnly/sameSite cookies, CSRF protection for mutations, input validation (Zod), output encoding, parameterised queries only (Drizzle), file-upload validation (magic-byte check, size/type limits, re-encode images with `sharp`, strip EXIF/GPS), virus-scan hook [LATER].
- **Rate limiting** (Upstash): OTP send/verify (per phone + IP), login, search, try-on, uploads. OTP: hashed storage, short TTL, attempt limits, resend cooldown.
- **AuthZ**: every server action/route checks role + resource ownership (sellers can only touch their own shop's data). Add tests that prove cross-tenant access fails.
- **Secrets**: only via env; never in client bundle (only `NEXT_PUBLIC_*` is public); secrets scanning in CI; rotate keys documented in README.
- **KYC & photos**: private buckets, AES-256-GCM application-level encryption for sensitive fields, short-lived signed URLs, access always audited.
- **Admin hardening**: see §7.3. Optional extra: separate DB role with least privilege for admin app, network-level restriction, mandatory re-auth for sensitive operations, anomaly alerts (new IP/device login → email).
- **Webhooks**: signature verification, replay protection, idempotency, IP allowlist if provider publishes ranges.
- **Dependency hygiene**: lockfile, Dependabot/Renovate, `pnpm audit` in CI.
- **Backups**: DB PITR/backups, tested restore runbook; object storage versioning for private buckets where available.

**Privacy & legal (consult counsel — this list is a prompt, not legal advice)**
- India **DPDP Act 2023**: notice + consent, purpose limitation, retention limits, deletion on request, grievance redressal, **children's data → verifiable parental consent** (hence 18+ gate on photo features). Data-processing records.
- Consumer Protection (E-Commerce) Rules: seller disclosures, grievance officer, clear return/refund, no misleading "AI" claims.
- IT Rules: takedown handling process for IP/counterfeit complaints (admin module supports it).
- Sellers' IP: marketplaces must handle counterfeit/brand complaints — policy + takedown flow.
- Store policy documents (Terms, Privacy, Seller Agreement, Refund Policy) as versioned content with acceptance logging.

---

## 12. Non-functional requirements

- **Performance:** LCP < 2.5 s and CLS < 0.1 on mid-range Android/4G for home/PLP/PDP; JS budget per route documented; image CDN + responsive `srcset`; ISR for catalog pages with on-demand revalidation when products go live/change.
- **Availability/Resilience:** graceful degradation when AI or provider down (manual review fallback, try-on hidden, mock-free); retries with backoff + dead-letter handling in worker; reconciliation jobs for payments & shipments.
- **Scalability (start small, don't paint into a corner):** stateless apps, DB indexes, queue-based async, no in-memory state.
- **Accessibility:** WCAG 2.1 AA targets (contrast, focus, labels, alt text — AI-generated alt suggestions allowed).
- **SEO:** SSR/ISR, structured data, sitemaps, canonical, fast.
- **Observability:** Sentry in all apps, request IDs through to worker jobs, job dashboard in admin, uptime check, alerting (payment failure spike, queue backlog, AI/try-on budget).
- **Quality:** TypeScript strict, ESLint, Prettier, commit hooks, ≥ 80 % unit coverage on `packages/core` & `packages/payments` logic, e2e for critical flows.
- **Internationalisation:** all user-facing strings via `next-intl`; INR formatting; date/time in IST.
- **Cost guardrails:** documented monthly cost model; AI budget caps; image variants generated once; cache everything cacheable.

---

## 13. Configuration & environment strategy

- **`.env.example`** at repo root is the canonical, fully-commented list of every variable (a ready-made file ships with this spec). Per-app `.env.example` files may exist only if variables are app-specific, but the root file must list **everything**.
- **Never commit `.env*`** (except `.env.example`). `.gitignore` enforced.
- `packages/config/env.ts`: Zod schemas split into `server` and `client` (`NEXT_PUBLIC_*`) per app; **validated at boot**; on failure print a *friendly table* of missing/invalid variables with the matching README section — then exit.
- **Mode flags** let the app run without keys (see `.env.example`): `OTP_MODE`, `EMAIL_MODE`, `STORAGE_PROVIDER`, `AI_MODE`, `TRYON_MODE`, `SHIPPING_MODE`, `WHATSAPP_MODE`, `PAYMENTS_MODE` (`test`|`live`; Razorpay test keys are free).
- **Whenever the agent adds a new variable**, it must: add it to `.env.example` (with comment: purpose, where to get it, free-tier/card note, which phase needs it), add it to the Zod schema, and add/update the README table. At the end of each phase it must print **"Env vars you need to fill now"** with step-by-step instructions.
- Separate env sets for `development`, `staging`, `production` (documented; hosts store secrets in their dashboards).

---

## 14. Phased build plan

> Rule: **finish a phase completely (code + tests + docs + demo path) before starting the next.** After each phase, the agent stops and reports using the checklist in `AgentPrompt.md`. Each phase's "Done when" list is the acceptance criteria.

### Phase 0 — Foundations & scaffolding
**Scope:** monorepo (pnpm + Turborepo), TS strict config, ESLint/Prettier, Tailwind + shadcn base in `packages/ui`, brand config, `packages/config` env validation, Drizzle setup + first migration, adapters skeletons with **mock implementations**, logging (pino), Sentry hooks, GitHub Actions CI, `.env.example`, seed script skeleton, README quick-start, health endpoints for each app, i18n skeleton.
**Done when:** `pnpm i && pnpm dev:storefront` renders a styled placeholder with no keys; CI green; env validation prints a friendly error when `DATABASE_URL` is missing; ADR files exist for stack decisions.

### Phase 1 — Identity, roles & seller onboarding
**Scope:** DB schema for users/sellers/admin; customer + seller auth (phone OTP via adapter, Google optional), sessions; **admin auth** (invite/seed only, TOTP 2FA, recovery codes, IP allowlist, idle timeout, re-auth), RBAC matrix + tests, audit log infra; seller application multi-step form with KYC upload to private bucket (encrypted), seller status machine; admin seller-verification queue (approve/reject/suspend, notes, templated reasons); notifications (email/SMS adapters, console mode).
**Done when:** a seller can apply → admin (with 2FA) approves → seller sees approved dashboard; cross-tenant access tests pass; audit entries exist for all admin actions; KYC files are only reachable via signed URLs.

### Phase 2 — Catalog, aesthetics & seller product management
**Scope:** aesthetics/categories CRUD (admin) + seed taxonomy + `docs/aesthetic-guide.md`; product/variant/image schema; **Snap & List** mobile flow and full form; image pipeline (sharp variants, EXIF strip, phash); product statuses; admin product moderation queue (manual for now); inventory; seller product list with inline edit; Hinglish copy for seller app.
**Done when:** an approved seller can create a product from a phone-sized viewport; admin approves it; it's stored with variants and optimised images; duplicate-image detection works; tests cover status transitions.

### Phase 3 — Storefront browsing & search
**Scope:** home, aesthetic hubs, category/PLP with filters/sort, PDP, search (FTS + trigram), SEO (JSON-LD, sitemap, OG), PWA basics, wishlist, recently viewed, shop pages, CMS-driven banners/collections (admin), ISR + on-demand revalidation, analytics events, performance budget enforcement (Lighthouse CI).
**Done when:** seeded catalog browsable end-to-end; Lighthouse mobile ≥ 90 perf on home/PLP/PDP with seeded data; filters are URL-synced; search handles typos.

### Phase 4 — Cart, checkout, Razorpay & orders
**Scope:** cart (guest + logged-in merge), stock reservation, address + pincode serviceability (mock/Shiprocket), coupons, pricing/GST engine, **Razorpay Order → Checkout → signature verify → webhooks → idempotent order finalisation**, reconciliation job, order state machine, seller order inbox + confirm/SLA auto-cancel, buyer order pages, notifications, invoices (PDF), admin order views, refunds (full/partial) via Razorpay.
**Done when:** Playwright e2e passes: success, failure, retry, abandoned (stock released), duplicate webhook, tampered amount rejected; refund works in test mode; ledger entries balance.

### Phase 5 — AI aesthetic verification & trend engine
**Scope:** `packages/ai` adapters (vision, embedding, moderation) + mocks; verification pipeline stages 1–4; decision engine + thresholds in admin; seller-facing feedback UI; AI-assisted listing (auto-fill); reference-image seeding & embedding job; `ai:eval` script + golden-set tooling; override capture; budget circuit breaker & `ai_calls` cost log; trend snapshot job and "Trending" ranking; admin AI dashboard.
**Done when:** a new listing flows through the pipeline asynchronously to auto-approve / review / auto-reject with explainable reasons; eval script outputs metrics; kill-switch drops everything to manual review; trust-level logic works.

### Phase 6 — Style Profile & Virtual Try-On (behind feature flag)
**Scope:** consent + 18+ gate, in-browser face/skin analysis (MediaPipe), Style Profile UI, private photo storage + expiry/purge jobs, try-on provider adapters (+ `mock`), job pipeline with caching/limits/budget cap, PDP try-on UX (progress, failure, disclaimer), size recommendation, "colours that suit you", `tryon_ready` detection for products, admin try-on ops page, privacy pages updated.
**Done when:** with a real provider key (or mock) a user can create a profile, try on supported garments, see cached repeats instantly, hit limits gracefully, and delete all data; evaluation notes on diverse test photos are in `/docs/tryon-evaluation.md`.

### Phase 7 — Logistics, returns, settlements & seller finance
**Scope:** Shiprocket adapter live (labels, pickup, tracking webhooks), return/exchange flows + reverse pickup, QC outcomes, **Razorpay Route** linked accounts + on-hold transfers + release job (or manual-payout fallback), ledger + seller earnings dashboard + statements, admin finance module & reconciliation, strikes/penalties automation, COD behind flag (optional), WhatsApp notifications (optional).
**Done when:** full lifecycle in test mode: order → ship → deliver → return window → payout; returns refund correctly with ledger adjustments; finance can reconcile a day's activity.

### Phase 8 — Hardening, compliance & launch readiness
**Scope:** security review against §11 checklist, pen-test-style tests (IDOR, rate limits, CSRF, upload abuse), load test on key paths, backup/restore drill, runbooks (`/docs/runbooks`), legal pages wired (Terms/Privacy/Refund/Seller Agreement/Grievance), cookie consent, DPDP data-subject request flows, accessibility pass, staging env, production deploy pipeline, monitoring/alerts, cost model doc, admin onboarding guide, seller onboarding guide, seed real aesthetic reference sets, soft-launch checklist.
**Done when:** launch checklist in `/docs/launch-checklist.md` is fully ticked; staging mirrors prod; a restore from backup was demonstrated.

### Phase 9 — Growth features (post-launch backlog, order by data)
Find-your-aesthetic quiz · personalised feed · weekly drops · outfit boards/shop-the-look · visual search · referrals · price-drop/back-in-stock alerts · bulk seller upload & AI background clean-up · seller analytics & trend nudges · Q&A · Instagram/Pinterest content hooks · seller-side team accounts · Meilisearch/Typesense · native-app wrapper (PWA→TWA) · multi-language expansion.

---

## 15. Testing, CI/CD & deployment

- **Unit (Vitest):** pricing/GST/commission math, order & product state machines, RBAC matrix, signature verification, decision engine thresholds, try-on cache keys, size recommender.
- **Integration:** DB-backed tests for services (use a dedicated test schema/branch on the hosted DB), webhook idempotency, job handlers.
- **E2E (Playwright):** shopper journey (browse → cart → pay in test mode), seller journey (apply → list), admin journey (2FA login → approve), failure paths.
- **AI tests:** `ai:eval` on the golden set; contract tests for provider adapters using recorded fixtures (no live calls in CI).
- **CI:** install → lint → typecheck → unit → build all apps → e2e (smoke) → migration dry-run. Block merges on failure.
- **Deploy:** storefront/seller/admin on Vercel (separate projects & env scopes), worker on Render/Railway/Fly, DB/storage on Supabase. Migrations run via a controlled CI job, not on app boot. Preview deployments for PRs with seeded data and **mock** providers.
- **Release hygiene:** conventional commits, changelog, tagged releases per phase.

---

## 16. Risks & open decisions (confirm with the human before the relevant phase)

| # | Topic | Risk / Question | Default assumption |
|---|---|---|---|
| 1 | **Seller supply & cold start** | Need enough on-aesthetic inventory at launch | Hand-onboard 15–30 sellers from Sarojini Nagar; seed collections |
| 2 | **Try-on quality & cost** | Quality varies by garment/body/skin tone; per-image cost can sink margins | Feature-flagged beta, strict budgets, mock default |
| 3 | **Razorpay Route** | Requires account activation; settlement rules | Build behind `SettlementService`; manual payout fallback |
| 4 | **COD** | RTO losses on fast fashion | OFF at launch |
| 5 | **Return policy** | Fast-fashion returns kill margins | 7-day, size/defect only; confirm |
| 6 | **Minors** | Teen users & DPDP | 18+ for photo features; legal review for checkout age policy |
| 7 | **Commission %** | Business decision | 12 % placeholder, configurable |
| 8 | **Logistics model** | Who pays/ships | Seller ships using platform labels; free shipping > ₹799 placeholder |
| 9 | **AI false approvals** | Brand/counterfeit leakage | New sellers always human-reviewed; IP flags escalate |
| 10 | **GST/TCS handling** | Compliance complexity | CA consulted; ledger captures data; no filing automation |
| 11 | **Brand name/domain** | Placeholder | `BRAND_NAME` config |
| 12 | **Free-tier limits** | Supabase pause on inactivity, API quotas | Document limits; upgrade path in README |

---

## 17. Out of scope for v1

Native mobile apps · live-stream shopping · international shipping/currencies · seller-side warehousing/fulfilment centres · marketplace ads/promoted listings · resale/thrift marketplace · AR (camera-based) try-on · social feed/UGC network · loyalty points · B2B wholesale ordering.

---

### Appendix A — Definition of "production-ready" for every phase
1. Feature works end-to-end on seeded data in a clean checkout (`pnpm i` → documented steps).
2. Types, lint, unit + e2e (for critical paths) pass in CI.
3. Authorisation verified (tests for forbidden access).
4. Errors handled with user-friendly messages; no unhandled promise rejections; Sentry hooked.
5. Env vars documented in `.env.example` + README; app runs in mock mode without keys where applicable.
6. Docs updated (README, ADRs, runbook notes). No TODO/FIXME left without a linked issue in `/docs/backlog.md`.
7. Accessibility and mobile viewport checked.
8. Security checklist items for that phase ticked.
