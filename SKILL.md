---
name: cloudflare-website-builder
description: 'Build a production website on Cloudflare (Workers + D1 + R2) with Next.js 16. Use when a user wants a new site, store, blog, portfolio, or admin-driven web app hosted on Cloudflare. Triggers: "new website", "new project", "Cloudflare site", "Next.js + Cloudflare", "D1", "R2", "Paystack site", "shop with admin", "deploy to Cloudflare", "custom domain". The skill bootstraps from an empty folder (initializes git, self-installs at .github/skills/cloudflare-website-builder/SKILL.md, can fetch inspiration websites the user supplies and derive theme/content from them), conducts a structured discovery interview (45+ questions covering hosting, data, storage, auth, payments, blog, cart, admin, theming, SEO, analytics, custom domain, novice/expert mode), then generates a phased implementation plan (12 phases by default, each with multiple stages and steps), enforces phase-gated execution with commit checkpoints, and ends with a code review + next-step recommendation. Offers a Novice Fast-Path (3 questions + smart defaults) and an Advanced Custom path (full interview + user-authored prompt). Includes step-by-step instructions for actions the LLM cannot perform (GitHub<->Cloudflare CI hookup, DNS, custom domain, Paystack live keys, secrets in dashboard).'
---

# Cloudflare Website Builder

Build a Next.js 16 site on Cloudflare Workers (or alternatives) end-to-end. Distilled from four shipped projects (`Repo A`, `Repo B`, `Repo C`, plus a `Repo D` Vercel variant) and their iterated plans.

> **Reading order for the agent:** (1) Run **Bootstrap** if the workspace is empty / has no git repo. (2) Run **Pre-Flight Scope Picker** to size the project. (3) Offer the **Novice Fast-Path** OR run **Phase 0 — Discovery** in full (Advanced). (4) Run **Inspiration Intake** if the user has reference sites. (5) Generate the plan from the **Plan Template** using the answers. (6) Execute phases one at a time under the **Execution Protocol**. (7) Run the **Final Review** when phases are complete. (8) Always cross-check against [**Lessons from Shipped Repos**](#lessons-from-shipped-repos-real-world-divergences) before recommending defaults — those rules supersede generic advice.

---

## Bootstrap — empty workspace / no git repo (run FIRST if applicable)

This skill is designed to be **dropped into an empty folder**: the user opens VS Code on a fresh directory, says "build me a website", and the skill takes over from zero. Run these checks before anything else:

### B1 · Detect workspace state

Use `list_dir` on the workspace root and `run_in_terminal` (`git rev-parse --is-inside-work-tree 2>/dev/null`) to determine which case applies:

| State | Action |
|---|---|
| Empty folder, no `.git` | Go to B2 (full bootstrap). |
| Has files but no `.git` | Ask: "I see existing files — do you want me to initialize a git repo here, or are you adding this site to an existing project?" Then B2 or B5. |
| `.git` exists, no remote | Go to B3 (remote setup). |
| `.git` exists with remote | Go to B4 (verify + skip ahead). |
| Existing Next.js project | Go to B5 (audit existing code before planning — see [Auditing existing code](#auditing-existing-code-when-the-user-already-started)). |

### B2 · Initialize git locally

1. Ask the user (concise, one question): "Do you want me to initialize a git repository here for you? (yes / no — if no, I'll skip git checkpoints.)"
2. If yes, run `git init -b main`.
3. **Always confirm the git identity for this repo — never silently assume the system default.** Do NOT skip this step even when `git config --global user.name` is already set; the global identity often belongs to a different person/account than the one who should author commits in this project.
   - Read the current global default: `git config --global user.name` and `git config --global user.email`.
   - Ask the user (use `vscode_askQuestions` if available) with this exact shape:
     > "Commits in this repo will be authored as **`<global name> <<global email>>`** (your current global git identity). Use this identity, or set a different name/email for this repo only? Please reply with: (a) **use default**, (b) **use this:** `Name <email>`, or (c) **GitHub noreply** — I'll construct `<username>@users.noreply.github.com` from your GitHub handle."
   - If no global identity exists, ask for both name and email outright (no default to offer).
   - Apply the chosen identity at the **repo scope only** (so it doesn't pollute other repos): `git config user.name "<name>" && git config user.email "<email>"`. Never write to `--global` unless the user explicitly says "set globally".
   - Echo back the final identity for confirmation before the first commit: `git config user.name && git config user.email`.
4. Create a minimal `.gitignore` immediately (node_modules, .next, .open-next, .dev.vars, .env*.local, .wrangler).
5. Make the initial commit: `git commit --allow-empty -m "chore: initial commit"`.

### B3 · Create / connect GitHub remote

Ask: "Do you have a GitHub repository for this project yet?"

- **"No, please create one"** — if `gh` CLI is installed (`command -v gh`), offer: `gh repo create <name> --private --source=. --push`. If `gh` is not installed, give the user the manual steps:
  1. Open https://github.com/new , create an empty repo named `<project>`, do NOT initialize with README/license.
  2. Copy the `git remote add origin <url>` and `git push -u origin main` commands shown by GitHub.
  3. Confirm back to me when pushed and I'll continue.
- **"Yes, here's the URL"** — run `git remote add origin <url>` then `git push -u origin main`.
- **"Skip, I don't want GitHub"** — note in the plan that auto-deploy via Cloudflare Workers Builds is unavailable; the user must `wrangler deploy` manually.

### B4 · Self-install the skill into the repo

Once a git repo exists, copy this SKILL.md into `<repo>/.github/skills/cloudflare-website-builder/SKILL.md` so it travels with the project and future sessions can re-load it. Steps:

1. Check whether `.github/skills/cloudflare-website-builder/SKILL.md` already exists in the workspace (most likely it does because the user dropped the skill folder in). If yes, skip.
2. If not, create the directory and copy the skill file. The agent can use `create_directory` + `create_file`. The skill content to copy is the full text of the loaded SKILL.md.
3. Add a one-line note in the README: `> Project conventions live in .github/skills/cloudflare-website-builder/SKILL.md`.
4. Commit: `git add .github/ && git commit -m "chore: vendor cloudflare-website-builder skill"`.

### B5 · Auditing existing code (when the user already started)

If the workspace already contains a Next.js app or partial implementation, do **not** scaffold over it. Instead:

1. Read `package.json`, `wrangler.jsonc`/`wrangler.toml`, `next.config.*`, and any existing `IMPLEMENTATION*.md`.
2. List existing routes (`src/app/**/page.tsx`), DB migrations (`migrations/*.sql`), and admin pages.
3. Echo back: "Here's what's already built: [...]. What's missing or broken that you want me to focus on?"
4. Skip Phase 0 questions whose answers are already evident from the code; only ask about gaps.
5. Generate a **gap plan** instead of a greenfield plan — phases address what's missing, not what's already shipped.
6. **Write the gap plan to a NEW versioned file — never edit the old one in place.** See [Plan Versioning Rule](#b6--plan-versioning-rule-never-overwrite-old-plans) below.

### B6 · Plan versioning rule — NEVER overwrite old plans

Implementation plans are append-only project history. Every time the agent (or a human) adds significant new scope after a previous plan was marked complete or shipped, the new work goes into a **new versioned file** alongside the old one. **Do not delete, rewrite, or "consolidate" prior plan documents.**

**File naming (use exactly these patterns so future LLMs can find them):**

- v1: `Implementation.md` OR `IMPLEMENTATION.md` OR `IMPLEMENTATION_PLAN.md` (whichever the project started with — keep the existing case/extension).
- v2: `IMPLEMENTATION_V2.md` (gap plan after v1 ships or stalls).
- v3: `IMPLEMENTATION_V3.md`.
- vN: `IMPLEMENTATION_V<N>.md`.

**When creating a new version (vN > 1):**

1. **Add a banner at the very top of the OLD plan** (before any other content). The banner must:
   - Identify the doc as historical for that version.
   - Name the new active plan file explicitly.
   - Tell future agents not to edit the old doc except to log historical context.
   - Example template:
     ```markdown
     # <Project> — Implementation Plan (v<N-1> — historical)

     > **Audit note (<date>):** <short summary of why a new plan was needed — e.g.
     > "Phases 1-9 marked complete but X, Y, Z shipped as placeholders; Phase 10 never
     > started.">
     >
     > **The active plan from this point on is `IMPLEMENTATION_V<N>.md`.** Do not edit
     > this v<N-1> doc except to log historical context (e.g. retroactive `[PENDING]`
     > or `[SUPERSEDED]` tags on phases that were never finished).
     ```
2. **Write the new plan to the new file.** Carry over (or link back to) any unfinished phases from the prior plan as their own renumbered phases in vN — e.g. v1 Phase 10 (deploy) becomes v2 Phase 19 with a "replaces v1 Phase 10" note.
3. **Re-tag superseded or never-started phases in the OLD plan** with status markers (`[PENDING]`, `[SUPERSEDED \u2014 see vN Phase X]`) but **leave the body intact** — it's evidence of what was originally intended.
4. **Update the project README** so its "Implementation plan" link points at the newest active vN. Older versions stay linked under a "Historical plans" subheading.
5. **Cross-reference both ways.** The new plan's front-matter has a `Replaces:` line naming the old file; the old plan's banner names the new file.
6. **Commit the banner update + new plan in the same commit** so a single `git log` entry tells the story: `docs(plan): start IMPLEMENTATION_V<N>.md (gap plan after v<N-1> audit)`.

**Why this is mandatory:**

- A future LLM session opening the workspace sees both files and can reconstruct the history. Deleting v1 erases the rationale for every shipped decision.
- Phase Completion Logs in the old plan are the ground truth for "why does the schema look like this?" / "why is bcrypt not used?" — losing them costs hours of re-discovery.
- Diff-based code review can't compare v2 against a deleted v1.
- The user almost always wants to refer back to "what did we originally plan?" mid-v2.

**Never:**

- Delete an old plan file.
- Rewrite an old plan's body to match new reality (only add status banners + tags).
- Skip the banner — silently writing a `_V2.md` next to an unannotated v1 will get the v1 picked up as authoritative on the next session.
- Use freeform names like `IMPLEMENTATION_NEW.md`, `IMPLEMENTATION_FINAL.md`, `IMPLEMENTATION_REVISED.md` — they break alphabetical sort and confuse future sessions. Only use `_V<N>.md`.

---

## Pre-Flight — Scope Picker (60 seconds, before Phase 0)

First classify the project so you don't ask 36 questions for a one-page site. Pick **one**:

| Scope | Description | Phases needed | Skip Phase 0 questions |
|---|---|---|---|
| **A · Marketing one-pager** | Single landing page, contact form, no DB writes, no admin. | 1, 2, 4, 5, 10, 11 (6 phases) | Skip Q11–Q19, Q21 |
| **B · Blog only** | Multi-post blog, optional comments, no shop. | 1–6, 9 (admin only if user wants editor), 10, 11 | Skip Q17–Q19 |
| **C · Portfolio / brochure** | A few pages, a gallery, contact form, maybe a CMS-lite admin. | 1, 2, 3, 4, 5, 9 (lite), 10, 11 | Skip Q17–Q19, Q21 |
| **D · E-commerce shop** ★ | Full shop: products, cart, checkout, payments, admin, optionally blog. | All 12 phases | Ask everything |
| **E · Booking / reservations** | Calendar slots, customer details, deposit payment, admin. | 1–5, 7 (slots not cart), 8, 9, 10, 11 | Skip Q17 (cart) — booking page replaces it |
| **F · Directory / listings** | Searchable list of items (places, businesses, jobs). | 1–6, plus search phase, 9, 10, 11 | Skip Q17–Q19 |
| **G · SaaS dashboard** | Multi-user app with auth, per-user data. | 1–4, full auth, app-specific phases | Skip Q17–Q22 |
| **H · Other / mix** | Combination — confirm with user which of A–G are closest. | Custom | Custom |

Also ask:
- **Speed-to-launch goal** — "shipped this week" (use defaults aggressively, defer everything optional) vs. "polished launch in a month" (do every phase properly).
- **MVP cut-off** — what's the *one* thing the site MUST do at launch? Anything else can be a follow-up phase.

Use this to prune the Phase 0 question set and the plan template. Tell the user which questions you're skipping and why.

---

## Two paths into Phase 0 — Novice Fast-Path vs. Advanced Custom

After the Scope Picker, offer the user a clear choice (single `vscode_askQuestions` call works well):

**Path 1 — Novice Fast-Path (recommended for most users).** Three quick questions, then the skill picks every default marked ★ from Phase 0 and prints a one-paragraph summary for confirmation. The three questions are:

1. **Project name + one-sentence purpose** (covers Q1).
2. **Country / currency** (covers Q2; drives payment defaults).
3. **Do you have any inspiration websites I should look at?** (URL list — triggers [Inspiration Intake](#inspiration-intake-optional-but-powerful) below). If "no", offer 3–4 preset visual themes from Q24 to pick from.

With those three answers + the scope from the picker, the skill fills every other Phase 0 answer with the ★ default, prints the summary, and asks **"Confirm or change anything?"**. The user can accept everything or override individual items conversationally.

**Path 2 — Advanced Custom (for experienced users / unusual requirements).** Run the full Phase 0 interview below. The user may also paste a free-form spec/brief at any point — the skill should parse it, map answers to Q-numbers, and only ask follow-ups for items it cannot infer. Always confirm inferred answers before generating the plan.

> Default to **Novice Fast-Path** unless the user explicitly asks for the full interview, self-identifies as an expert/intermediate developer in their initial message, or the Scope Picker landed on **D (e-commerce)** or **G (SaaS)** — those scopes have too many decisions to safely default. For those scopes, run the Advanced path (which asks Q5 explicitly).

---

## Inspiration Intake (optional, but powerful)

If the user supplies one or more URLs as inspiration / competitor / prior-site references:

1. Use `fetch_webpage` (or `open_browser_page` + screenshot if available) to retrieve each URL. Extract: page titles, headings, hero copy, navigation labels, dominant colours (from CSS where visible), font families, image style, sections present (hero / features / testimonials / pricing / FAQ / footer), products or blog post examples, contact methods.
2. For each site, summarise back to the user in 3–5 bullets: "On `<url>` I see X, Y, Z. Vibe: <minimal/luxury/playful>. Palette: <colours observed>."
3. Then ask **targeted follow-ups** that turn inspiration into decisions, e.g.:
   - "Site A uses a serif heading + warm cream background. Want the same direction, or different?"
   - "Site B has a pricing page and a blog. Do you want both?"
   - "Site C's hero has a full-bleed video. Replicate (heavier) or use a still image (faster, cheaper bandwidth)?"
   - "Anything from these sites you specifically want to AVOID?"
   - "Anything missing from all of them that you want yours to have?"
4. Roll the resolved answers into Phase 0 (especially Q4 pages, Q24 colours, Q25 typography, Q26 vibe, Q20 blog presence, Q38 legal pages).
5. Save the inspiration notes to `IMPLEMENTATION.md` under a `## Inspiration` section so future sessions can reference the source intent.

If the user can't browse to a site (auth-walled, private), ask for screenshots or a description instead.

---

## Phase 0 — Discovery Interview (Advanced path — run in full unless Novice Fast-Path was chosen)

Ask the user these questions **before** scaffolding anything. If the `vscode_askQuestions` tool is available, use it (group related questions into one call). Otherwise ask in chat. Do not proceed past this phase until **every** question has an answer or an explicit "skip / use default".

If the user answers vaguely ("just build it"), present the recommended defaults (marked **★**) and ask them to confirm or change.

### A. Project identity & scope

1. **Project name** and one-sentence purpose? (e.g. "Repo B Homestead — small-batch fermentation supplies").
2. **Primary audience country / language / currency**? (drives locale, currency formatting, payment provider choice).
3. **Site type** — pick all that apply: marketing/landing only · blog · e-commerce shop · portfolio · booking/trips · directory · SaaS dashboard · other.
4. **Pages required at launch** (list them: Home, About, Contact, Shop, Blog, FAQ, Services, etc.).

### B. Skill level & autonomy

5. **Self-described skill level** — `novice` · `intermediate` · `expert`?
   - **novice → agent will run `git add/commit/push` automatically after each green phase.**
   - **intermediate/expert → agent will pause and instruct the user to commit/push themselves.**
6. **Package manager preference** — `npm` ★ (default; lowest CI risk) · `pnpm` (faster, used by Repo B; needs `pnpm-workspace.yaml` for Workers Builds CI) · `bun` (untested on Workers Builds in reference repos) · `yarn`?
6a. **Route grouping** — flat `src/app/` · group by audience (`(main)/`, `(shop)/`, `(auth)/`) ★ once the app has >5 pages? Repo B and Repo C both use route groups; Repo A is flat.
7. **Local dev OS** (macOS / Linux / Windows / WSL) — affects shell command examples.

### C. Hosting & runtime

8. **Hosting target** — Cloudflare Workers (via `@opennextjs/cloudflare`) ★ · Cloudflare Pages (legacy, static-only) · Vercel · self-hosted Node? If unsure, default to Cloudflare Workers (matches the rest of this skill).
9. **Cloudflare account** — already exists? If yes, account ID and email. If no, instruct user to sign up at https://dash.cloudflare.com/sign-up before Phase 1 ends.
10. **Custom domain** — yes/no? If yes: domain name, registrar, and whether DNS is already on Cloudflare.

### D. Data layer

11. **Database choice** — Cloudflare D1 (edge SQLite) ★ · JSON files in repo (zero-DB, fine for ≤50 items, no admin writes) · Cloudflare KV (key-value only) · external Postgres (Neon/Supabase) · Durable Objects? Explain the trade-off: D1 = SQL + writes from edge; JSON = simplest, no admin CMS; Postgres = needed for complex relational queries or large datasets.
12. **Expected data volume / write frequency**? (helps validate the choice in Q11).

### E. Media & file storage

13. **Image / file storage** — Cloudflare R2 ★ · `public/` folder in repo (no admin upload) · external (Cloudinary, Bunny, S3)? R2 needs ~3 extra steps (bucket, public dev URL, upload route) but enables an admin-managed media library.
14. **Will admins upload images** through the dashboard, or are all images committed to the repo by developers?

### F. Authentication

15. **Auth needed?** none · admin-only (single-account dashboard) ★ for shops · multi-user customer accounts · social login?
16. **Auth implementation preference** — **NextAuth.js v5 (`next-auth@beta`)** ★ for novices and simple sites (used in `Repo B` and `Repo C`; still on `5.0.0-beta.30` as of 2026-04-18 — accept beta risk; provider plugins make OAuth trivial) · **`jose` custom JWT** for everything else (used in `Repo A`; lowest-risk for edge, ~150 LOC, no beta dependency) · NextAuth v4 stable · Clerk (paid above 10k MAU) · Auth.js + D1 adapter? **Always use PBKDF2 via Web Crypto for password hashing** — bcrypt does NOT run on Workers.
16a. **OAuth providers needed at launch** — none ★ · Google · GitHub · Apple · Facebook · multi-provider mix? (Picking any non-"none" forces Q16 to NextAuth v5 or Clerk; `jose` custom JWT supports only email/password.)

> **Auth default rule:** novices and simple sites (Scope A/B/C/F or Q5 = novice) → **NextAuth v5**. Larger / advanced projects, or anyone who wants to avoid a beta dep → **`jose`**. Override with explicit user choice.

### G. Commerce & payments

17. **Shopping cart needed?** yes/no.
18. **Payment providers** — pick all: Paystack (cards + mobile money, KES/NGN/GHS/ZAR) · Stripe (global cards) · WhatsApp click-to-chat order ★ for KE · M-Pesa till/paybill (manual confirm) · Flutterwave · PayPal · cash on delivery · bank transfer · none.
19. **Order fulfilment** — digital only · physical (need shipping address + courier) · service booking (calendar slot)?
19a. **Inventory tracking** — none (just in-stock/out-of-stock toggle, simplest) · per-product stock count ★ · per-variant stock count · low-stock alerts? **All three shipped shop repos used a boolean `in_stock` and regret it** — recommend at minimum a numeric stock_quantity column from day one.
19b. **Product variants** — none · sizes only · sizes + colours · full attribute matrix? Variants change the schema (`product_variants` table, price overrides, per-variant stock) — answer this NOW, retrofitting is painful.
19c. **Discount codes / coupons** — none ★ for v1 · percentage codes · fixed-amount · BOGO? Often deferred to v2 — confirm explicitly.
19d. **Customer accounts** — none (guest checkout only) ★ · optional account at checkout · required account? Every shipped shop deferred this and shipped guest-only — that's fine, just confirm.
19e. **Order receipts via email** — none (rely on Paystack receipt) · plain-text confirmation · branded HTML · PDF invoice attachment? Needs Q31 answered.
19f. **Shipping zones / costs** — flat rate · free over X · per-zone (county/region/country) ★ once you ship physical goods beyond one city · weight-based (requires `weight_grams` on products) · external (Sendy/DHL API)? Repo B ships a `shipping_zones` table with base rate + per-kg + free-above threshold — use that as the reference schema if the user picks "per-zone" or "weight-based".
19g. **Tax handling** — none (price-inclusive) ★ · VAT line item · region-based?
19h. **Returns / refunds workflow** — none for v1 · admin-marks-refunded · automated (Paystack refund API)?
19i. **Reviews / ratings on products** — none ★ for v1 · star ratings · text reviews moderated by admin · third-party (Trustpilot, Judge.me)?
19j. **Low-stock alerts** — none ★ · admin-only warning icon when `stock < threshold` · email notification to admin (requires Q31) · public "only 2 left!" banner on product page? Repo B ships an `inventory_alerts` table with per-product thresholds and last-notified timestamp.

### H. Content features

20. **Blog?** yes/no. If yes: MDX files in repo (used in `Repo B`) · D1-backed posts with admin rich-text editor ★ (used in `Repo A`) · headless CMS (Sanity/Contentful/Payload)?
20a. **Blog post statuses** — published/unpublished only · draft → published → archived ★ · also "scheduled (publish at future date)" · also "private (link-only preview)"? **All shipped repos have only boolean published — recommend the 4-state enum from day one to avoid migration later.**
20b. **Blog extras** — featured posts · related-posts on detail page · RSS/Atom feed · author profiles (multi-author) · reading-time estimate · table of contents on long posts?
21. **Comments on blog?** none · simple anonymous · threaded with rate-limit + honeypot ★ (battle-tested in `Repo A` with `parent_id` self-reference) · requires login · third-party (Disqus, Giscus)?
21b. **Comment nesting depth** — single-level (no replies) · one level of replies only · unlimited nesting ★ (Repo A ships this with recursive render)?
21a. **Sample blog posts** — should I generate 3–5 realistic placeholder posts (lorem-style with on-brand topics, ~400 words each) so the site doesn't ship empty? yes ★ / no.
22. **Newsletter signup** — none · just collect emails to D1 · external (Mailchimp, Buttondown, ConvertKit, Resend audiences, Beehiiv, Substack import)? **Don't ship the form unless the backend is also wired** — every shipped repo had an unwired newsletter form.
23. **Contact form** — store in D1 ★ · also email via Resend/Postmark/SES/MailChannels · POST to webhook (Zapier, Make.com, n8n)?
23a. **Anti-spam on public forms** — honeypot field ★ · Cloudflare Turnstile (free) · hCaptcha · simple math question · rate-limit by IP only?
23b. **Search** — none ★ for v1 · client-side filter on already-loaded list · D1 FTS5 full-text search · Cloudflare Vectorize (semantic) · Algolia (paid, fastest)? Common omission — every shipped repo has only category filtering.
23c. **Sample product / catalog data** — should I seed 8–20 realistic-looking placeholder products in your category? yes ★ / no — *or* upload your real CSV?
23d. **Sample images** — use Unsplash/Pexels placeholders (with attribution) ★ · use coloured gradient placeholders · skip images entirely · I'll provide my own?

### I. Look & feel

24. **Brand colour palette** — provide hex codes, or pick a preset: *Earthy Walnut* (warm browns/sage), *Coastal Sage* (greens/teal), *Modern Charcoal* (greys/copper), *Sunset Amber* (orange/cream), *Custom* (ask for primary + accent + neutral).
25. **Typography** — serif heading + sans body (Playfair Display + DM Sans/Inter ★) · all-sans (Inter) · all-serif · custom Google Fonts?
26. **UI density / vibe** — minimal & airy · dense & utilitarian · playful · luxury · brutalist?
27. **Dark mode** required? auto (system) · toggle · light-only ★?

### J. SEO, analytics, ops

28. **SEO must-haves** — sitemap.xml + robots.txt + Open Graph ★ · JSON-LD structured data (Organization, Product, BlogPosting) · multilingual hreflang · dynamic OG images (`opengraph-image.tsx`)?
28a. **Image optimization strategy** — `<img>` tags only (simplest, what every shipped repo did) · `next/image` with custom R2 loader ★ recommended (we have a ready-made loader) · external image CDN (Cloudflare Images $5/mo, Imgix, Bunny Optimizer)? **Repos that deferred `next/image` never came back to it. Decide now.**
29. **Analytics** — none · Cloudflare Web Analytics (free, no cookies) ★ · Plausible ($9/mo) · Umami (self-host or $9/mo) · GA4 · PostHog (free up to 1M events)?
30. **Error monitoring** — none ★ for v1 · Sentry (free 5k errors/month) · Cloudflare Logpush (paid Workers plan) · Axiom (free 0.5GB/mo)?
31. **Email-sending need** — order receipts, contact form replies, password resets, magic links? If yes: Resend ★ (3k free/mo, used in `Repo B`) · Postmark ($15 starter) · MailChannels (free via Workers, harder DKIM setup) · SES (cheap at scale, complex setup).
31a. **Email templates** — plain text · React Email components ★ (works with Resend) · MJML · external builder (Mailchimp templates)?

### K. Repo & CI

32. **GitHub repository** — already exists? If yes, URL. If no, will be created in Phase 1.
33. **Auto-deploy on push to `main`** via Cloudflare Workers Builds (recommended) — yes ★ / no? Note: connecting GitHub ↔ Cloudflare is **manual** (browser OAuth) — see [manual steps](#manual-steps-the-llm-cannot-perform).

### L. Internationalization & accessibility

34. **Languages** — English only ★ · multilingual (list locales) · auto-translate via API? Multilingual changes routing (`[locale]` segment) — answer NOW.
35. **Currency display** — single ★ · multi-currency with FX conversion?
36. **Timezone for date display** — server timezone ★ · auto-detect from browser · user-selectable?
37. **Accessibility target** — WCAG AA ★ (default) · WCAG AAA · best-effort?

### M. Legal, ops & misc

38. **Required legal pages** — Privacy Policy · Terms of Service · Refund/Shipping policy · Cookie banner (only if you set non-essential cookies)?
39. **Multi-environment?** — single prod ★ · staging + prod · preview-per-PR (Cloudflare Pages-style branch deploys)?
40. **Backup strategy** — none ★ for v1 · weekly `wrangler d1 export` to R2 · automated cron via GitHub Actions?
41. **Admin role granularity** — single "admin" role ★ · admin + editor (editor can't manage users) · full RBAC? Every shipped repo has a single role.
42. **Audit log** — none ★ · log admin actions to a `audit_log` table (recommended once you have >1 admin)?
43. **CSV import/export** for products/orders — none ★ · admin can export orders to CSV · admin can bulk-import products from CSV?
44. **Anything explicitly out of scope** for v1 (so we don't over-build)?
45. **Anything that MUST be ready by launch** that we haven't covered yet?

> **After receiving answers**, echo a one-paragraph summary back to the user and ask **"Confirm these choices and I'll generate the implementation plan?"** Do not write the plan until they say yes.

---

## Plan Template (generate after Phase 0 is confirmed)

Write the plan to `IMPLEMENTATION.md` at the repo root. The plan **must** contain at minimum the 12 phases below (some may be skipped per the Scope Picker — e.g. Phase 6 for non-blog sites, Phases 7–8 for non-shop sites). Add or split phases based on Phase 0 answers (e.g. extra phase for multilingual, comments, newsletter integration). Each phase has **multiple stages**, each stage has **multiple concrete steps**.

### Plan front-matter (always include)

```markdown
# <Project Name> — Implementation Plan

> Stack: Next.js 16 (App Router) · Tailwind CSS v4 · <currency>
> Hosting: <choice from Q8> · Database: <Q11> · Storage: <Q13>
> Payments: <Q18> · Auth: <Q15/Q16>
> Date drafted: <today>
> GitHub repo: <Q32 or "to be created">
> Cloudflare account ID: <Q9 or "to be created">
> Skill level: <Q5> — <commit-policy: "agent commits & pushes" | "user commits & pushes">
```

### Execution Protocol (paste verbatim into every plan)

```markdown
## Execution Protocol — applies to every phase

1. **Phase Gating.** No code from a future phase may be written until the current phase is `[COMPLETE]` and the user has said "go ahead with Phase N+1".
2. **Independent Testability.** Each phase delivers a slice that runs and is verified on `localhost` *in isolation* — Phase 3 must boot even if Phases 4–10 don't exist yet. Ship vertical slices, not horizontal ones.
3. **Phase Context block.** Every phase begins with a "Phase Context" table summarising: codebase state, DB state, storage state, deployment state, prior-phase diversions.
4. **Local Verification Check.** Every phase ends with a copy-pasteable shell block that runs `dev`, `lint`, `tsc --noEmit`, `build`, and the manual click-through checklist.
4a. **End-of-Phase Handoff to User \u2014 REQUIRED.** After the agent finishes coding a phase, it MUST post a structured handoff message before asking for the commit. The handoff has these fixed sections (omit a section only if truly N/A; never silently skip):

   1. **What I built** \u2014 1\u20133 bullets summarising the slice that now works.
   2. **What I already verified automatically** \u2014 list the commands run and their results (`lint` clean, `tsc --noEmit` clean, `build` green, any unit/integration tests added). Include exit codes / brief output.
   3. **What you need to test manually (the LLM cannot verify these)** \u2014 a numbered checklist. For EACH item give:
      - **Setup:** any prep (env var to set, seed to run, browser to open, account to log into).
      - **Steps:** click-by-click or curl-by-curl instructions.
      - **Expected result:** exact text / status code / DB row / visual outcome the user should see.
      - **Failure signal:** what \"broken\" looks like, so the user can report back precisely.
      - Cover at minimum: visual rendering on desktop + mobile viewport, keyboard/a11y where relevant, third-party integrations (Paystack/Stripe sandbox, Resend test send, OAuth round-trip, R2 upload viewable in dashboard, D1 row visible via `wrangler d1 execute ... \"SELECT ...\"`), webhook receipt (use Paystack test event or `webhook.site` mirror), email deliverability (check inbox + spam), payment receipts, and any \"happy path\" user journey the phase enables.
   4. **Cross-device / cross-browser checks** \u2014 explicitly list which browsers + viewport sizes to test (default: latest Chrome + Safari + mobile Safari at 375px). Skip only if the phase is purely backend.
   5. **Data to inspect** \u2014 the exact `wrangler d1 execute ... \"SELECT ...\"` query, R2 object key to look up, or KV key to fetch so the user can confirm persistence.
   6. **Known limitations / deferred items** \u2014 anything noticed during the phase that is intentionally out of scope, with the phase number where it'll be addressed.
   7. **Questions for the user before we move on** \u2014 explicit yes/no or multi-choice questions the agent needs answered to start the next phase (e.g. \"Confirm the Paystack test charge appeared on your dashboard?\", \"Should the cart persist across browsers or only per-device?\", \"Pick one of these three error-page copy options\u2026\"). If there are zero open questions, write \"None \u2014 ready for commit + Phase N+1\" so the user knows nothing is blocking.
   8. **How to report back** \u2014 a one-line template the user can copy-paste, e.g. `\"Phase N test report: [pass/fail per checklist item], notes: \u2026\"`. This makes the next session start cleanly.

   The agent must wait for the user's test report (or an explicit \"skip testing, proceed\") before running the Commit Checkpoint. Never auto-commit a phase whose manual checks haven't been acknowledged.
5. **Commit Checkpoint.** After local verification passes **and** the user has reported back on the manual checklist (or explicitly waived it):
   - **Novice mode:** the agent runs `git add -A && git commit -m "feat(phase-N): <summary>" && git push` automatically and reports the commit SHA.
   - **Intermediate/Expert mode:** the agent stops and prints the exact `git` commands for the user to run, then waits for confirmation.
6. **Phase Completion Log.** Append `[COMPLETE]` to the phase heading and fill in: Date · Diversions · Decisions affecting future phases. Commit the plan update *with* the phase commit.
7. **One Phase Per Session — REQUIRED, not a suggestion.** At the end of every completed phase the agent MUST tell the user: "Open a fresh chat session before starting Phase N+1. Re-using this session will degrade output quality because the context window is now full of completed-phase noise." If, in the next turn, the user simply says **"proceed"**, **"continue"**, **"go ahead"**, **"next phase"**, or any equivalent without confirming a new session, the agent MUST **interject and stop** before doing any work:
   > "Quick check before I start Phase N+1 — are you in a **new chat session**? Re-using the previous session for a new phase noticeably degrades quality because the prior phase's context, file reads, and tool outputs crowd out the working memory needed for the next phase. Reply **'yes, new session'** to continue, or open a fresh chat and paste the plan path + 'start Phase N+1' there."
   Only proceed once the user explicitly confirms a new session (or explicitly overrides with "I know, just proceed in this session"). Never silently continue.
8. **Recovery Rule.** If a phase is stuck after two failed attempts at the same approach, **stop**, commit what works, log the blocker in the Completion Log, and ask the user for guidance instead of brute-forcing.
9. **No Unicode in plan/comments.** Use plain ASCII (hyphens, straight quotes) so diffs stay clean.
10. **Build must be green before commit.** `npm run lint` and `npx tsc --noEmit` must both pass.
```

### Phase 1 — Project scaffolding & local dev environment

**Goal:** A Next.js 16 + Tailwind v4 app that boots on `localhost:3000` with Cloudflare bindings declared (even if empty).

Stages:
- **1.1 Tooling check** — verify Node ≥ 20, install package manager (Q6), install Wrangler globally.
- **1.2 Scaffold** — `npx create-next-app@latest <name> --typescript --tailwind --app --src-dir --eslint --no-turbopack` (adjust flags). Move generated `AGENTS.md`/`CLAUDE.md` if present.
- **1.3 Install Cloudflare adapter** — `npm i @opennextjs/cloudflare` and `npm i -D wrangler`.
- **1.4 Create `wrangler.toml`** ★ (default — prefer `.toml` over `.jsonc` unless there is a concrete reason such as needing JSON Schema-driven IDE validation, sharing config with a tool that only reads JSON, or a Cloudflare upgrade that explicitly requires `.jsonc`). Include `name`, `main = ".open-next/worker.js"`, `compatibility_date`, `compatibility_flags = ["nodejs_compat", "nodejs_compat_populate_process_env"]`, `[assets]`, and empty placeholders for `[[d1_databases]]` and `[[r2_buckets]]`. **`nodejs_compat_populate_process_env` is required if you want to read `[vars]` and secrets via `process.env.FOO` (Node-style) instead of always going through `getCloudflareContext().env.FOO`.** Without it, `process.env.*` returns `undefined` in Worker code at runtime even though Next/`@opennextjs/cloudflare` typings suggest otherwise — see [Process env runtime gotcha](#process-env-runtime-gotcha-cloudflare-workers) below. **If you choose `wrangler.jsonc` instead, you MUST (a) state the reason out loud before creating the file, and (b) document the why in the Phase 1 completion log under a `### Config format decision` heading so future sessions and reviewers know it wasn't accidental.**
- **1.5 Create `open-next.config.ts`** — `export default defineCloudflareConfig({ incrementalCache: r2IncrementalCache });` (omit cache if not using R2).
- **1.6 Create `cloudflare-env.d.ts`** with `interface CloudflareEnv { ... }`.
- **1.7 `next.config.ts`** — call `initOpenNextCloudflareForDev()` in a try/catch (bindings may not exist yet).
- **1.8 npm scripts** — `dev`, `build`, `lint`, `preview` (`opennextjs-cloudflare build && opennextjs-cloudflare preview`), `deploy`, `cf-typegen`.
- **1.9 `.gitignore`, `.env.local.example`, `.dev.vars`, `README.md`.**
- **1.10 Local verification** — `npm run dev` shows the default page; `npm run build` succeeds.
- **1.11 Commit checkpoint.**

### Phase 2 — Branding, theme & design system

Stages: tokens (CSS variables in `globals.css` via `@theme inline`), Google Fonts via `next/font`, primitive components (`Button`, `Card`, `Container`, `SectionHeading`, `Input`, `Badge`), optional `/styleguide` page, dark-mode wiring (only if Q27 = yes).

### Phase 3 — Data layer (D1 / JSON / Postgres)

Stages depend on Q11:
- **D1 path:** create DB (`wrangler d1 create <name>`), write `migrations/0001_initial_schema.sql` (users, categories, products, product_images, blog_posts, blog_categories, orders, order_items, contact_messages, comments — include only the tables needed by Phase 0 answers), `npm run db:migrate`, write seed `scripts/seed.sql`, `lib/db.ts` with `getCloudflareContext()` accessor, `lib/queries/*.ts` modules.
- **JSON path:** create `data/*.json` files, typed loaders in `lib/data/*.ts`, no admin write capability.
- **Postgres path:** Neon/Supabase project, `DATABASE_URL` env var, `drizzle-orm` or `kysely`, migrations folder.

End-of-phase check: a server component reads from the data layer and prints a value on `/__data-test`.

### Phase 4 — Global layout & landing page

Stages: root `layout.tsx` (header, footer, mobile nav, metadata, fonts, providers), `Hero`, feature sections per Q4, `NewsletterForm` stub if Q22, `Footer` with socials.

### Phase 5 — Static content pages

About, Contact (with API route or server action), legal pages from Q38. Contact form persists per Q23.

### Phase 6 — Blog system *(skip if Q20 = no)*

Stages: `/blog` listing with pagination, `/blog/[slug]` detail, server-side D1 fetch, OG/JSON-LD per post, `@tailwindcss/typography` plugin, optional comments per Q21, optional categories.

### Phase 7 — Shop & cart *(skip if Q17 = no)*

Stages: `/shop` listing + filters + sort, `/shop/[slug]` with `ProductGallery`, `CartContext` + `localStorage` persistence, `CartIcon`/`CartDrawer` in header, `/cart` page, formatPrice utility (currency from Q2).

### Phase 8 — Checkout & payments *(skip if Q17 = no)*

Stages per Q18:
- **Paystack:** `/api/paystack/initialize` route, `/checkout/callback` server-component verifier, `/api/paystack/webhook` with HMAC-SHA512 signature verification via Web Crypto.
- **Stripe:** Checkout Sessions + webhook with `STRIPE_WEBHOOK_SECRET`.
- **WhatsApp order:** `lib/whatsapp.ts` builds `wa.me/<num>?text=<encoded>` link, order saved as `pending`.
- **M-Pesa till manual:** display till number, customer confirms via WhatsApp.
- Order rows written with `RETURNING *`. `/checkout/success` confirmation page.

### Phase 9 — Auth & admin dashboard *(skip if Q15 = none)*

Stages:
- **9.1 Auth library** — install `jose` (custom JWT) **or** `next-auth@beta` per Q16. PBKDF2 hash via Web Crypto API in `lib/auth-utils.ts`. Seed an admin user (password change required before launch).
- **9.2 Login page** `/admin/login` + server action.
- **9.3 Middleware** — `src/middleware.ts` protects `/admin/*`. Reads `AUTH_SECRET` from `process.env`. (Default to `middleware.ts` — see Gotcha #13 / AP12 / L15; do NOT preemptively rename to `proxy.ts`.)
- **9.4 Admin layout & dashboard overview** — counts, revenue, recent orders.
- **9.5 Products CRUD** — list, new, edit, delete; image upload to R2.
- **9.6 Blog CRUD** — same shape; cover image upload.
- **9.7 Orders management** — list, detail, status update.
- **9.8 Categories / comments / contact-messages** as enabled in Phase 0.
- **9.9 R2 upload route** `/api/upload` + media-serving route `/api/media/[key]` with immutable cache headers.

### Phase 10 — SEO, performance & polish

`sitemap.ts` (force-dynamic on Workers — D1 not available at build), `robots.ts` (disallow `/admin`), JSON-LD (Organization, Product, BlogPosting), `not-found.tsx`, `error.tsx`, favicon + manifest, accessibility pass (semantic HTML, ARIA, contrast, keyboard nav), Lighthouse target ≥ 90 on Performance / SEO / A11y.

### Phase 11 — Cloudflare provisioning & first deploy

Stages:
- **11.1 Provision remote resources:** `wrangler login`, `wrangler d1 create <name>` (paste returned ID into `wrangler.toml` — or `wrangler.jsonc` if that's what Phase 1 documented), `wrangler r2 bucket create <name>`, enable R2 public dev URL in dashboard.
- **11.2 Apply migrations remotely:** `wrangler d1 migrations apply <name> --remote` (or `execute --remote --file=...`).
- **11.3 Set secrets:** `wrangler secret put PAYSTACK_SECRET_KEY`, `AUTH_SECRET`, etc. (one per provider chosen in Q18).
- **11.4 First deploy:** `npm run deploy` → confirm `https://<name>.<account>.workers.dev` loads.
- **11.5 GitHub ↔ Cloudflare CI** — see [manual steps](#manual-steps-the-llm-cannot-perform).
- **11.6 Custom domain (if Q10 = yes)** — see manual steps.
- **11.7 Smoke test in production:** repeat Phase 4–9 click-throughs against the live URL.

### Phase 12 — Launch hardening (optional but recommended)

Rate-limiting on contact/comment routes, Turnstile or honeypot on forms, real Paystack live keys, swap admin seed password, Cloudflare Web Analytics beacon, error monitoring (Q30), backup/export plan for D1 (`wrangler d1 export`), uptime monitoring.

> **Add additional phases** if Phase 0 surfaced: multilingual (i18n routing), bookings (calendar/reservations), search (D1 FTS5 or Cloudflare Vectorize), email templates, mobile app companion, etc.

---

## Cloudflare-specific gotchas the plan MUST encode

These are pulled from real diversion logs across the four projects — bake them into the relevant phase up-front to save the user from rediscovering them.

| # | Gotcha | Where to handle |
|---|---|---|
| 1 | `@opennextjs/cloudflare` deploys as a **Worker**, NOT a Pages project. `wrangler.toml` (preferred) or `wrangler.jsonc` uses `main + assets`, never `pages_build_output_dir`. | Phase 1 |
| 2 | Use `getCloudflareContext()` from `@opennextjs/cloudflare`, **not** `getRequestContext()` (that's the older `next-on-pages` API). | Phase 3 |
| 3 | All server components/route handlers that touch D1 or R2 need `export const runtime = "edge"`. | Phases 3, 6, 7, 8, 9 |
| 4 | `open-next.config.ts` must use `defineCloudflareConfig({...})` — a bare object export fails silently. | Phase 1 |
| 5 | `next.config.ts` must call `initOpenNextCloudflareForDev()` for `npm run dev` to access D1/R2 locally. Wrap in try/catch until bindings exist. | Phase 1 |
| 6 | D1 is **not available at build time**. Drop `generateStaticParams` for D1-backed routes; mark sitemap/feed/dynamic pages `export const dynamic = "force-dynamic"`. | Phases 6, 10 |
| 7 | `next/image` needs a custom loader for R2 URLs. Default to plain `<img>` until the optimisation pass, or write a loader that points at `/api/media/[key]`. | Phase 6 onward |
| 8 | `pnpm-workspace.yaml` with `packages: ["."]` is required if using pnpm + Workers Builds CI. Add `onlyBuiltDependencies: [esbuild, workerd, sharp, unrs-resolver]`. | Phase 1 (only if Q6 = pnpm) |
| 9 | Webhook signatures must use **Web Crypto** (`crypto.subtle.importKey` + `verify`) — Node `crypto` is not available on Workers without `nodejs_compat`. | Phase 8 |
| 10 | Cookie auth: set `httpOnly`, `secure`, `sameSite: "lax"`. JWT verification with `jose` works natively on Workers; bcrypt does **not** — use PBKDF2 via Web Crypto. | Phase 9 |
| 11 | A second R2 bucket for the OpenNext incremental cache (`NEXT_INC_CACHE_R2_BUCKET`) is an **optimization**, not a default — only 1 of 3 audited repos uses it. Offer it in Phase 11 for sites with >50 pages or slow cold builds. | Phase 11 (optional) |
| 12 | Cloudflare Workers Builds CI deploys can return a transient `502` from the deploy API. Document "retry once" in Phase 11. | Phase 11 |
| 13 | **Do NOT preemptively rename `middleware.ts` → `proxy.ts`**. All 3 audited Next.js 16 Cloudflare repos ship `middleware.ts`. Verify current convention via `node_modules/next/dist/` before any rename. | Phase 9 |
| 14 | `compatibility_date` in `wrangler.*` must be within ~6 months of today. Stale values (e.g., `2024-09-23` observed in Repo C) silently miss runtime bug fixes. | Phase 11 |
| 15 | Paystack `pk_test_*` publishable keys are safe to commit, but keep them in `.dev.vars` rather than `wrangler.toml [vars]` to avoid accidental prod leak. Live keys always via `wrangler secret put`. | Phase 8 |
| 16 | <a id="process-env-runtime-gotcha-cloudflare-workers"></a>**`process.env.*` requires the `nodejs_compat_populate_process_env` compatibility flag** on Cloudflare Workers. Without it, `process.env.MY_VAR` returns `undefined` at runtime even though `wrangler types` generates a `NodeJS.ProcessEnv` interface that suggests otherwise. With the flag, every `[vars]` entry, secret (`wrangler secret put`), and `.dev.vars` value is mirrored into `process.env`, which means Node-style libraries (NextAuth, jose, Resend, Paystack SDKs, ad-hoc `process.env.XXX` checks in shared lib code) "just work" without rewiring through `getCloudflareContext().env`. **Add this flag in Phase 1** alongside `nodejs_compat`. The flag is additive and safe to enable on any Worker that already has `nodejs_compat`. After enabling, run `wrangler types` to regenerate `worker-configuration.d.ts` so TypeScript matches runtime behaviour. Reference: [Cloudflare docs — Node.js compatibility flags](https://developers.cloudflare.com/workers/runtime-apis/nodejs/). | Phase 1 |

---

## Asset generation — images, icons, and video

When the project needs placeholder or production assets (hero images, product photos, blog cover art, logos, social OG images, short promo video, etc.), the agent should **generate them itself** rather than blocking on the user to supply files. Pick the format based on Phase 0 storage answers (Q13) and the use case:

### Default: SVG everywhere it works
- Hand-author SVG for logos, icons, illustrations, hero graphics, empty-state art, category tiles, and decorative shapes. SVG is text, so the agent can write it directly into `public/` (or upload to R2) and edit it later without external tooling.
- For product/photo-style placeholders where SVG looks wrong, generate a **gradient + iconography SVG** keyed off the brand palette from Q24 — much more on-brand than grey boxes.
- Always include `viewBox`, `role="img"`, and a `<title>` for a11y.

### When PNG/JPG is required (OG images, favicons, raster fallbacks)
- Convert SVG → PNG locally. Install a lightweight, **local** converter — do NOT call hosted APIs:
  - Node: `npm i -D sharp` then a tiny `scripts/svg-to-png.mjs` that reads SVG and writes PNG at the needed sizes (16, 32, 180, 512, 1200×630 for OG).
  - Python alternative: `pip install cairosvg pillow` and a `scripts/svg_to_png.py`. Prefer this if Node tooling is being painful or if `sharp` won't install on the user's platform.
  - CLI alternative: `rsvg-convert` (via `brew install librsvg`) for one-off conversions.
- For favicons specifically, also produce `favicon.ico` (multi-size) — `sharp` + `to-ico` or Python `pillow` both work.
- Commit the generated PNGs into `public/` (or upload to R2 if Q13 = R2 and the asset is admin-managed). Keep the source SVG alongside so the asset can be regenerated.

### When video is required (hero loop, product demo, social teaser)
- Build the video from assets the agent already has — never block on the user to record something:
  1. Generate a sequence of SVG frames (or a single SVG with CSS/SMIL animation).
  2. Rasterize each frame to PNG via the script above.
  3. Stitch with `ffmpeg` (install: `brew install ffmpeg` / `apt install ffmpeg`): `ffmpeg -framerate 30 -i frame_%04d.png -c:v libx264 -pix_fmt yuv420p -movflags +faststart out.mp4`. Add `-vf scale=1280:720` for sizing, `-an` to drop audio.
  4. For a looping hero clip, also produce a WebM (`libvpx-vp9`) for smaller size and a poster JPEG (first frame).
- Alternative paths when ffmpeg is unavailable: Python `moviepy` (`pip install moviepy`) for programmatic composition, or `manim` for math/diagram animations.
- Keep generated videos under ~2 MB for hero use; longer demos should live on YouTube/Vimeo with an embed (record the URL in `IMPLEMENTATION.md`).

### Storage & delivery decision matrix

| Q13 storage | Static SVG/icons | Generated PNG/JPG | Generated video |
|---|---|---|---|
| `public/` folder | `public/images/` (committed) | `public/images/` (committed; watch repo size) | `public/video/` only if <2 MB; otherwise YouTube embed |
| Cloudflare R2 ★ | `public/icons/` (build-time, no R2 needed) | Upload to R2 via `/api/upload`; serve via `/api/media/[key]` or `NEXT_PUBLIC_R2_BASE_URL` | Upload to R2; reference by URL in `<video>` tag |
| External CDN | Same as `public/` for icons | Push to CDN as part of release | Reference CDN URL |

### Python scripts are a first-class tool
The agent is **encouraged to write small Python scripts** (in `scripts/`) for any asset/data task where Python is the shortest path to the goal — SVG→PNG conversion, batch image resizing, frame-to-video stitching, CSV→SQL seed generation, scraping inspiration sites for colour palettes, generating placeholder copy, exporting D1 backups, etc. Rules:
- Put scripts under `scripts/` with a clear name (`scripts/generate_og_images.py`).
- Add a one-line docstring describing what it does and the exact command to run it.
- Pin dependencies in a single `scripts/requirements.txt` (or per-script `# /// script` PEP 723 inline metadata for `uv run`).
- Never check secrets into scripts — read from `.dev.vars` / env.
- Add the script's command to `IMPLEMENTATION.md` so future sessions know how to re-run it.

Goal trumps purity: if a Python one-liner unblocks the user, write the Python one-liner.

### All LLM-generated assets are PLACEHOLDERS \u2014 say so, and prompt the user to replace them

Every image, icon, video, logo, or audio file the agent produces is a **template / placeholder** intended only to unblock development and shipping a visually-complete-looking site. None of it is final brand art. The plan and the running site must make this obvious so the user doesn't accidentally launch with stand-in art:

1. **Mark placeholders in code.** When generating an asset, name it with a `placeholder-` prefix (e.g. `public/images/placeholder-hero.svg`, `public/images/placeholder-product-1.png`, `public/video/placeholder-hero-loop.mp4`). Add a leading SVG comment / metadata tag `<!-- PLACEHOLDER: generated by cloudflare-website-builder skill on <date>. Replace before launch. -->`. For PNG/JPG/MP4, write a sibling `.README.md` next to the asset folder listing every placeholder file and where it's used.
2. **Track placeholders in `IMPLEMENTATION.md`.** Add a top-level section `## Placeholder assets to replace before launch` and append a row each time a placeholder is created: `| File | Used on | Suggested final | Replaced? |`. Keep the table accurate \u2014 update it when the user supplies real art.
3. **Prompt the user up front (Phase 2 or whichever phase first generates assets).** Use `vscode_askQuestions` with something like:
   > \"For each of these I can either (a) generate a stylised SVG/PNG placeholder now so the site looks complete, or (b) wait for you to drop the real file in `public/images/` (paste the file or a link). Logo? Hero image? Product photos (count: __)? Blog cover images? Founder/about photo? OG share image? Hero video? Tell me which to generate vs which you'll supply.\"
4. **Provide a drop-in workflow for the user's real assets.** In the same handoff, give the user the exact paths/sizes/formats they should provide:
   - Logo: SVG preferred (or PNG \u2265 512\u00d7512 transparent). Drop at `public/logo.svg`.
   - Hero image: 1920\u00d71080 JPG/WebP. Drop at `public/images/hero.jpg`.
   - Product photos: square 1200\u00d71200 JPG/WebP, named `product-<slug>-1.jpg`, `-2.jpg`\u2026 Drop at `public/images/products/` (or upload via admin if Q13 = R2).
   - Blog cover: 1600\u00d7900 JPG/WebP per post.
   - OG share image: 1200\u00d7630 PNG/JPG. Drop at `public/og-default.png`.
   - Hero video: \u22642 MB MP4 (H.264, 720p, no audio, looped). Drop at `public/video/hero.mp4` plus a JPG poster.
   - Tell the user: \"Drop the file at the path above with the same name and the placeholder is automatically superseded \u2014 nothing else to change.\" If the path/name differs, instruct how to update the reference in code.
5. **Surface placeholders during End-of-Phase Handoff.** Rule 4a's handoff message must list any placeholder assets created in that phase under a \"Placeholders awaiting your real files\" heading, with the suggested final spec for each.
6. **Block launch on remaining placeholders.** The Final Review production-readiness checklist must include: `[ ] No files matching public/**/placeholder-* remain referenced in source (run \\`grep -r \"placeholder-\" src/ public/\\`).` If any remain, list them and ask the user to confirm \"ship with placeholders\" or supply real files first.
7. **Apply the same rules to seeded text content.** Lorem-style blog posts, product names like \"Sample Mug\", and stub testimonials are placeholders too \u2014 mark them with `[PLACEHOLDER]` in the title or `is_placeholder = 1` in the DB row, and add them to the same replacement table.

The intent: the user is never surprised at launch by stock-looking art or test credentials they thought were final. Every placeholder asset, seeded row, and test-mode key has an obvious name, a tracked row, and a one-step path to replacement — but the choice to ship with them stays with the user.

---

## Manual steps the LLM cannot perform

The plan must surface these as explicit user-action checklists with screenshots/links — never silently assume the agent will do them.

### M1 · Cloudflare account setup (Phase 1)
- Sign up at https://dash.cloudflare.com/sign-up.
- Capture the **Account ID** from the dashboard sidebar.
- Run `npx wrangler login` locally — opens the browser for OAuth. The agent cannot complete this.

### M2 · Connect GitHub repo to Cloudflare Workers Builds (Phase 11)
1. Push the repo to GitHub first (Phase 1 commit).
2. Cloudflare dashboard → **Workers & Pages** → **Create** → **Import a repository** → authorise the Cloudflare GitHub app on the target repo.
3. Set **Build command:** `npm run deploy` (or `pnpm run deploy`). **Deploy command:** *empty* (the build script already deploys via `opennextjs-cloudflare deploy`).
4. **Root directory:** `/` (or the subdirectory containing `package.json`).
5. **Environment variables:** add `NEXT_PUBLIC_*` vars and any non-secret values.
6. **Secrets:** add via `wrangler secret put <NAME>` from the terminal — they appear in the dashboard but cannot be added by the agent.
7. Trigger the first build; if it returns 502, retry from the dashboard.

### M3 · Custom domain (Phase 11)
1. **Move DNS to Cloudflare** (if not already): change nameservers at the registrar to the two ns1/ns2 strings shown in the Cloudflare dashboard. Propagation takes minutes-to-hours — agent cannot speed this up.
2. **Workers & Pages → your worker → Settings → Domains & Routes → Add Custom Domain** → enter the domain. Cloudflare auto-issues an SSL cert.
3. Update env vars: set `NEXT_PUBLIC_SITE_URL=https://<domain>` and `metadataBase` in `layout.tsx`.
4. Update the Paystack webhook URL in the Paystack dashboard to the new domain.

### M4 · Paystack live mode (Phase 12)
- Toggle the dashboard to **Live**, copy `pk_live_*` and `sk_live_*`.
- `wrangler secret put PAYSTACK_SECRET_KEY` (paste live key). Update `NEXT_PUBLIC_PAYSTACK_KEY` in dashboard env vars.
- Register the production webhook URL `https://<domain>/api/paystack/webhook`.

### M5 · R2 public dev URL
- Cloudflare dashboard → R2 → bucket → **Settings** → **Public Access** → enable `r2.dev` URL. Copy and set as `NEXT_PUBLIC_R2_BASE_URL` env var.

### M6 · OAuth providers (if Q15 social)
- Google: https://console.cloud.google.com → OAuth consent + credentials. Authorised redirect: `https://<domain>/api/auth/callback/google`.
- Agent prepares the code; user must create the credentials.

### M7 · Email sender verification
- Resend / Postmark / SES: verify the sending domain via DNS TXT records (must be added in Cloudflare DNS).

### M8 · R2 bucket creation + public access (Phase 11)
- Create the media bucket: `wrangler r2 bucket create <project>-media`.
- **Enable public access** for dev URLs: Cloudflare dashboard → R2 → bucket → **Settings** → **Public access** → **Allow Access**, copy the `pub-<hash>.r2.dev` URL, set as `NEXT_PUBLIC_R2_BASE_URL`.
- For production, bind a custom subdomain (e.g. `media.yourdomain.com`) → R2 bucket Settings → **Custom Domains** → add. Cloudflare issues the SSL cert automatically once DNS resolves.
- Note: R2 itself is free up to 10 GB storage + 10 M Class A + unlimited egress on the free plan; but **you must have added a payment method** to Cloudflare before R2 will accept any writes. Open https://dash.cloudflare.com/?to=/:account/billing and add a card (it will not be charged within free limits).

### M9 · D1 database creation + migrations (Phase 11)
- Create: `wrangler d1 create <project>-db`. Paste the returned `database_id` into `wrangler.*`.
- Apply migrations locally: `wrangler d1 migrations apply <project>-db --local` (used by `npm run dev`).
- Apply to production: `wrangler d1 migrations apply <project>-db --remote` — the agent **cannot skip `--remote`**; every manual run must specify it explicitly.
- Seed sample content (if enabled): `wrangler d1 execute <project>-db --remote --file=scripts/seed.sql`.

### M10 · Cloudflare Workers Builds CI — exact build & deploy commands
When connecting the GitHub repo in the Cloudflare dashboard (M2), paste these exactly:

- **Build command (npm):** `npm ci && npm run deploy`
- **Build command (pnpm):** `pnpm install --frozen-lockfile && pnpm run deploy`
- **Deploy command:** *(leave empty)* — `deploy` already runs `opennextjs-cloudflare deploy`.
- **Root directory:** `/` (or the sub-folder containing `package.json`).
- **Node version:** 20 (set under Build variables: `NODE_VERSION=20`).
- **Build variables (non-secret):** `NEXT_PUBLIC_SITE_URL`, `NEXT_PUBLIC_R2_BASE_URL`, `NEXT_PUBLIC_PAYSTACK_KEY` (`pk_test_*` or `pk_live_*`), `NEXT_PUBLIC_WHATSAPP_NUMBER`, `AUTH_TRUST_HOST=true`.
- **Secrets (cannot be set in the dashboard by the agent):** `wrangler secret put PAYSTACK_SECRET_KEY`, `wrangler secret put AUTH_SECRET`, `wrangler secret put RESEND_API_KEY` (if emailing). Agent prints the commands; user runs them.

### M11 · Local dev commands the agent should give the user verbatim
- First run (one-time): `npm install` → `npx wrangler login` → copy `.dev.vars.example` → `.dev.vars` and fill values.
- Daily: `npm run dev` — runs `next dev` with Cloudflare bindings via `initOpenNextCloudflareForDev()`. Local D1 + R2 are auto-provisioned under `.wrangler/state/`.
- Preview the real Worker locally: `npm run preview` (runs `opennextjs-cloudflare build && opennextjs-cloudflare preview`).
- Deploy: `npm run deploy`.
- Type-regen after wrangler changes: `npm run cf-typegen`.
- Reset local D1: delete `.wrangler/state/v3/d1/` and re-run migrations with `--local`.

### M12 · Paying for Cloudflare features that require a card
Cloudflare's free tier is generous but some features require a payment method on file **even when usage stays within free limits**:

- **R2** — card required before first write (10 GB + 10 M ops free).
- **Workers Paid plan ($5/mo)** — needed only if you exceed 100 k requests/day OR use Durable Objects / Queues / Cron Triggers >3/day.
- **D1** — free up to 5 GB + 5 M reads/day; card not required for the free tier.
- **Custom domain on Workers** — no extra fee (the domain fee is separate, paid to the registrar).
- **Cloudflare Images** ($5/mo, 100 k images) — optional; only needed if using the R2 image loader's `/cdn-cgi/image/` transforms.

Open https://dash.cloudflare.com/?to=/:account/billing and add a card before Phase 11 to avoid mid-deploy blockers.

### M13 · Paystack dashboard setup (step-by-step, agent cannot do this)
1. Sign up at https://dashboard.paystack.com/#/signup. Choose the country that matches Q2.
2. **Test keys:** Settings → **API Keys & Webhooks** → copy `pk_test_*` + `sk_test_*`. Put `pk_test_*` in `NEXT_PUBLIC_PAYSTACK_KEY` (safe to commit/env-var), `sk_test_*` via `wrangler secret put PAYSTACK_SECRET_KEY`.
3. **Test webhook:** same page → Webhook URL: `http://localhost:8787/api/paystack/webhook` for local via `wrangler dev`, OR use https://webhook.site to inspect events first.
4. **Go live:** complete KYC (business registration + bank details). Support can take 1–5 business days. When live-mode is unlocked, repeat step 2 with `pk_live_*` / `sk_live_*` and register the production webhook `https://<domain>/api/paystack/webhook`.
5. **Required webhook events:** `charge.success` (mandatory). Optional: `transfer.success`, `refund.processed`.
6. **HMAC secret:** for webhook signature verification, use the **secret key** (`sk_*`) as the HMAC-SHA512 key — Paystack does not provide a separate webhook secret.

---

## Commit policy (drives novice vs. expert mode)

After every successful Phase Verification:

```text
if user_skill == "novice":
    run: git add -A
    run: git commit -m "feat(phase-N): <one-line summary>"
    run: git push origin main          # only if remote configured
    report SHA + GitHub URL to user
    say: "Pushed commit <sha>. Open Phase N+1 in a fresh chat when ready."
else:
    print exact commands the user should run
    wait for user reply: "committed" / "done" / SHA
    do not start next phase until confirmed
```

For both modes, **never** commit if `lint` or `tsc --noEmit` failed. **Never** use `--no-verify` or `git push --force` without explicit user instruction.

---

## Final Review (run after the last planned phase)

When the user says "all phases done" or the last phase is `[COMPLETE]`, perform this review and present it as a single report:

1. **Inventory** — list every route under `src/app/`, every API route, every D1 table, every R2 bucket, every env var/secret, every npm script.
2. **Phase-0 vs reality diff** — for each Phase-0 answer, mark: ✅ shipped · ⚠️ partial (with what's missing) · ❌ skipped. Reference each item to the file path that delivers it.
3. **Production-readiness checklist:**
   - [ ] All `placeholder` / `TODO` / `XXX` strings removed (grep).
   - [ ] Admin seed password changed.
   - [ ] All secrets set in production (`wrangler secret list`).
   - [ ] Paystack on live keys; webhook URL registered.
   - [ ] DNS / custom domain resolves with valid SSL.
   - [ ] Sitemap + robots return correct content on prod.
   - [ ] Lighthouse Performance / SEO / A11y all ≥ 90.
   - [ ] Privacy / Terms pages linked in footer.
   - [ ] Backups: `wrangler d1 export <name> --remote --output backup-<date>.sql` documented in README.
4. **Foreseen next steps** — propose a short list (2-5 items) of what the user *probably* wants next, drawn from common Phase-0 answers they declined (e.g. "you said 'no newsletter for v1' — want to add Resend now?"; "comments are off — enable threaded comments?"; "no analytics — add Cloudflare Web Analytics beacon (free, 2 lines)?"). Ask **"Which of these should I plan next?"** and offer to draft a v2 of the plan.
5. **End-to-end manual test plan handed to the user.** The agent MUST produce a single document (append to `IMPLEMENTATION.md` under `## Launch Test Plan`) covering everything the LLM cannot verify itself. Use the structure below for each user journey:

   - **Journey name** (e.g. "Guest places a paid order via Paystack").
   - **Pre-conditions** — env vars, seeded data, test card numbers (e.g. Paystack `4084 0840 8408 4081` test card), test phone numbers (M-Pesa sandbox), test email account to receive receipts.
   - **Steps** — numbered, click-by-click on the live URL.
   - **Expected results** — visual + DB + email + webhook outcomes per step.
   - **Where to look if it fails** — Cloudflare Workers log (`wrangler tail`), Paystack dashboard events tab, R2 dashboard for upload, browser devtools console + network tab, D1 query to inspect row state.
   - **Rollback / cleanup** — how to delete the test order/comment/upload so it doesn't pollute prod data.

   Cover at minimum (skip only those not in scope): home page renders + nav works · contact form submission appears in D1 + email · blog list paginates and a post detail loads · comment posts and replies nest correctly · product list filters and detail page gallery works · cart persists across reload + survives navigation · checkout completes with a Paystack test card and arrives as `paid` in admin · webhook signature validation rejects a tampered payload · admin login + logout works and middleware blocks unauthenticated `/admin` · admin can create/edit/delete a product, blog post, and order status · R2 image upload appears via public URL · sitemap.xml + robots.txt return 200 · 404 page renders · mobile (375px) layout works · keyboard nav + visible focus rings · Lighthouse run (the agent gives the exact `npx lighthouse <url> --view` command but the user opens the report).

6. **Deliverability + third-party smoke tests the user must run.** List with exact commands / dashboard links:
   - Send a real Resend test email and confirm it arrives (and doesn't land in spam — check `mail-tester.com` score).
   - Trigger a Paystack test webhook from their dashboard ("Send test event") and confirm a `200` in `wrangler tail`.
   - Open the live site from a phone on cellular (not WiFi) and confirm hero loads in under 3s.
   - Run `curl -I https://<domain>` and confirm `cf-cache-status`, `strict-transport-security`, and `content-security-policy` headers are present.
   - Verify backups: run `wrangler d1 export <db> --remote --output backup.sql` and confirm the file has rows.

7. **Reporting template the user pastes back.** Provide a fenced block they copy into the next chat:

   ```
   Launch test report — <date>
   - Journey 1 (<name>): pass | fail — notes:
   - Journey 2 (<name>): pass | fail — notes:
   - Lighthouse: Perf __ / SEO __ / A11y __ / Best Practices __
   - Deliverability: Resend ok? Paystack webhook 200? curl headers ok?
   - Blockers preventing launch:
   - Nice-to-haves discovered:
   ```

---

## Anti-patterns to refuse

- Deleting, renaming, or rewriting a prior `IMPLEMENTATION*.md` instead of adding a new `IMPLEMENTATION_V<N>.md` per the [Plan Versioning Rule](#b6--plan-versioning-rule-never-overwrite-old-plans). Old plans are append-only history.
- Writing a new plan version without adding the `> The active plan from this point on is IMPLEMENTATION_V<N>.md` banner to the prior plan — future sessions will pick up the wrong file.
- Naming a new plan `IMPLEMENTATION_NEW.md` / `IMPLEMENTATION_FINAL.md` / `IMPLEMENTATION_REVISED.md`. Only `IMPLEMENTATION_V<N>.md` is allowed.
- Switching `wrangler.jsonc` <-> `wrangler.toml` partway through a project to "clean up" — pick one in Phase 1 and keep it. If migrating later, do it as its own phase with a diversion-log entry, never silently.
- Skipping Phase 0 entirely because "it's just a small site" — at minimum run the Novice Fast-Path (3 questions) and confirm the defaults summary.
- Asking all 36 Phase 0 questions when the user picked Novice Fast-Path or supplied inspiration sites that already answered them — infer, then confirm.
- Scaffolding a Next.js app on top of an existing project without first running Bootstrap step B5 (audit existing code).
- Skipping git initialization "to save time" — without a repo, every Commit Checkpoint becomes a no-op and rollback is impossible.
- Fetching inspiration URLs and silently copying their copy/images — always summarise observations and ask the user what to keep, change, or avoid.
- Writing horizontal slices ("first all the DB, then all the UI, then payments") — phases must be vertical & independently demonstrable.
- Using `next-on-pages` examples from older docs — we are on `@opennextjs/cloudflare`.
- Hardcoding secrets in source. Always `.dev.vars` locally + `wrangler secret put` in prod.
- Using bcrypt, Node `crypto`, or any non-edge-compatible library on Workers without verifying `nodejs_compat` covers it.
- Generating `generateStaticParams` for D1-backed routes — D1 is not available at build time.
- Continuing to the next phase without a green commit. Always stop at the gate.
- "Improving" code outside the current phase's scope (refactoring, renaming, adding unrelated features).

---

## Source projects (real-world reference plans)

Four shipped projects produced the patterns above, audited 18 April 2026. Read their `IMPLEMENTATION*.md` files for concrete examples of completed phases, diversion logs, and SQL schemas:

- `Repo A/Implementation.md` — cleanest 10-phase template; custom JWT (`jose`), threaded comments, blog categories, two R2 buckets, MVP shop. Phases 1–9 complete; Phase 10 (deploy) pending as of audit.
- `Repo B/app/IMPLEMENTATION.MD` — richest feature set: NextAuth v5 beta + Google OAuth, Resend emails, product variants, coupons, shipping zones, reviews, wishlists, MDX blog, single R2 bucket. Logs the Pages → Workers pivot and the transient 502.
- `Repo C/IMPLEMENTATION_PLAN.md` + `IMPLEMENTATION_PLAN_V2.md` — v1 was Django + Next; v2 rewrote to Next-only on Cloudflare. Uses NextAuth v5 beta, single R2 bucket, `wrangler.toml` (stale `compatibility_date = 2024-09-23`). Route groups `(main)/` and `(auth)/`.
- `Repo D/Implementation.md` — Vercel + Zustand variant (AI kitchen design app). Reference only for "what Q8 ≠ Cloudflare looks like"; no D1, no R2.

---

## Lessons from Shipped Repos (real-world divergences)

Findings from auditing the four reference projects' shipped code against their plans. **These rules supersede generic defaults above.**

### L1 · Patterns that recurred in 3+ repos (high-confidence defaults)

> **Adoption numbers below are from an 18 April 2026 audit of the four reference repos: `Repo A`, `Repo B`, `Repo C`, and `Repo D` (the Vercel variant / Vercel variant). Only the first three are Cloudflare; `Repo D` is a Vercel counter-example.**

| Pattern | Adoption | Use as default unless user overrides |
|---|---|---|
| Next.js 16 App Router + TypeScript + Tailwind v4 | 4/4 | Always |
| `@opennextjs/cloudflare` v1.17+ → Cloudflare **Workers** (NOT Pages) | 3/3 Cloudflare | Always for Cloudflare |
| Tailwind v4 with `@theme inline` in `globals.css` (no `tailwind.config.js`) | 3/3 | Always |
| Prices stored as `INTEGER` in minor units (cents) — display via `formatPrice()` helper | 3/3 | Always for any currency |
| Web Crypto API (PBKDF2) for password hashing — never bcrypt | 1/3 verified (Repo A); NextAuth repos (Repo B, Repo C) hide the hash behind the lib | Always enforce on Workers |
| Phase-gated plan with completion logs documenting diversions and forward-decisions | 3/3 | Always — copy structure verbatim |
| Multi-payment funnel for African markets: Paystack + WhatsApp click-to-chat + M-Pesa till manual | 3/3 | Default for KES/NGN/GHS/ZAR markets |
| Blog separated from product catalog (different schema, different routes) | 3/3 | Always when both exist |
| `AUTH_TRUST_HOST=true` set in `[vars]` of wrangler config | 3/3 Cloudflare | Required when any cookie/session is used on Workers |
| Route groups for layout isolation: `(main)/`, `(shop)/`, `(auth)/` | 2/3 (Repo B, Repo C) | Recommended once the app has >5 pages |
| `/api/seed` dev-only route for reloading sample data | 2/3 | Optional but handy during Phase 3 / 9 |

### L1a · Patterns that did **NOT** recur — revise earlier guidance accordingly

| Prior claim | Reality from audit | Revised guidance |
|---|---|---|
| "Two R2 buckets (media + `NEXT_INC_CACHE_R2_BUCKET`) are the default" | **Only 1 of 3** Cloudflare repos (`Repo A`) ships two buckets. Repo B and Repo C use a single `MEDIA` bucket and rely on default (in-Worker) incremental cache. | Treat the second bucket as a **performance optimization**, not a default. Offer it in Phase 11 if the user has >50 pages or complains about cold builds. |
| "`wrangler.jsonc` over `wrangler.toml`" | Only `Repo A` uses `.jsonc`. Repo B and Repo C use `.toml` and ship fine. **Default to `wrangler.toml`** — it's what the majority of audited repos use, plays nicely with every wrangler version, and is the format Cloudflare's own docs lead with. Pick `.jsonc` only when there is an explicit reason (JSON Schema IDE validation, tool that consumes JSON, a Cloudflare upgrade demanding it). When `.jsonc` is chosen, document the reason in the Phase 1 completion log under `### Config format decision`. **Once chosen in Phase 1, do not switch formats mid-project.** Every plan doc, README, script, CI build command, and grep references the filename; a silent rename breaks all of them. If a switch is genuinely required later, do it as a one-stage phase with `git mv` + a diversion-log entry, not a side edit. Also keep the plan docs in sync — if the repo ships `.jsonc` but the plan still says `.toml`, mark the stale lines with `*(superseded — shipped as .jsonc)*` rather than rewriting history. |
| "Next.js 16 renamed `middleware.ts` → `proxy.ts`" | **3/3 Cloudflare repos use `middleware.ts`** (either at repo root or `src/`). None use `proxy.ts`. The rename is either a different context or was rolled back. | **Default to `middleware.ts`.** Still verify against `node_modules/next/dist/docs/` at Phase 9, but do not preemptively rename. |
| "NextAuth v5 is the default auth" | 2/3 use NextAuth (both on `5.0.0-beta.30` as of audit); 1/3 uses `jose`. NextAuth v5 is still **beta** in April 2026. | Present as a genuine coin-flip in Q16. For a truly stable release, `jose` is lower-risk until NextAuth 5.0.0 GA ships. |

### L2 · Recurring anti-patterns to actively prevent

| # | Anti-pattern observed | How the skill must prevent it |
|---|---|---|
| AP1 | Boolean `in_stock` column with no quantity tracking | Use the [Updated Schema Templates](#l4--updated-schema-templates) below, not the simple `in_stock INTEGER` from the original plans |
| AP2 | Deferring `next/image` — every repo deferred it, none came back | Provide the [R2 image loader](#l5--ready-made-r2-image-loader) in Phase 6, not Phase 10 |
| AP3 | Newsletter form with no backend — every repo shipped a dead form | If Q22 is "none" then **delete the form from the layout**. Don't leave decorative-only inputs |
| AP4 | Boolean `published` column on blog posts (no draft/scheduled/archived) | Use the 4-state `status` enum below |
| AP5 | Deploying to Cloudflare Pages first, then pivoting to Workers (caused 502s, wasted CI cycles in 2 repos) | Phase 1 must explicitly state "Workers, not Pages" with the `wrangler.toml` (preferred) or `wrangler.jsonc` template |
| AP6 | NextAuth on Workers without `AUTH_TRUST_HOST=true` → silent redirect-loop in prod | Phase 9 must add this env var if Q16 = NextAuth |
| AP7 | Order persisted only after payment success — orders lost if user closes tab mid-Paystack | Persist order with `status='pending'` BEFORE redirecting to Paystack, update on webhook |
| AP8 | Admin seed password (`admin123`) never changed before going live | Final Review must grep for known seed passwords and block launch |
| AP9 | Hardcoded placeholder phone/till numbers (`254700000000`, `XXXXXXX`) committed to source | Final Review must grep for these |
| AP10 | `generateStaticParams` on D1-backed routes — works locally, breaks build on Workers | Phase 6 template uses `export const dynamic = "force-dynamic"`, never `generateStaticParams` |
| AP11 | Skipping the data-test page at end of Phase 3 → schema bugs surface in Phase 6 instead | Phase 3 must have a verifiable read |
| AP12 | Assuming `middleware.ts` has been renamed to `proxy.ts` in Next.js 16 without checking | All 3 audited Cloudflare repos still use `middleware.ts`. Default to `middleware.ts`; verify with `ls node_modules/next/dist/esm/lib/constants.js` or the Next.js changelog before renaming. |
| AP13 | Committing `pk_test_*` Paystack / Stripe **test** publishable keys into `wrangler.toml [vars]` (observed in Repo C) | Test publishable keys are not secrets, but mixing them with production config leads to "works locally, fails in prod" confusion. Keep test keys in `.dev.vars` and set production keys via `wrangler secret put`. |
| AP14 | Stale `compatibility_date` in `wrangler.toml` (Repo C has `2024-09-23` — 19 months old at audit time) | Phase 11 must bump `compatibility_date` to within the last 6 months and verify the build still works. Old dates silently miss Workers runtime bug fixes. |
| AP15 | Retrofitting `product_variants` after launch | Only Repo B shipped variants; the other two shops can't add them without a data migration. If Q19b is yes, create the table in Phase 3, even if the UI ships later. |
| AP16 | Storing `orders.items` as a JSON blob (Repo B) vs a normalized `order_items` table (Repo A) | JSON is faster to ship but blocks per-item reporting, refunds, and stock reconciliation. Default to `order_items`. Only use JSON if Q19 is "digital only, no fulfilment". |

### L3 · Common omissions (every shipped repo missed these — add to the plan up-front)

For e-commerce projects, the plan template MUST include or explicitly defer these — never silently omit:

- **Search** (D1 FTS5 or client-side) — every shop has only category filtering
- **Inventory quantities** (not just boolean stock)
- **Email order receipts** (Resend integration takes 1 hour, big UX win)
- **Customer accounts / order history lookup** (even just "look up my order by email + order number")
- **Discount codes**
- **`next/image` with R2 loader** (perf + bandwidth)
- **JSON-LD Product / BlogPosting schema** (only `Repo A` shipped this; SEO impact is significant)
- **Blog draft/scheduled state** (every repo blocked itself with boolean `published`)
- **Related blog posts** on detail pages
- **RSS feed for blog**
- **Audit log** for admin actions (mandatory once >1 admin)

For each, ask: "ship in v1?" / "v2?" / "never?" — log the answer in the plan so the omission is intentional, not accidental.

### L4 · Updated schema templates

Use these instead of the simpler versions in the original Repo A plan. They cost nothing extra at v1 but prevent painful migrations later.

```sql
-- products: full inventory, variants, soft delete, audit
CREATE TABLE products (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  name            TEXT NOT NULL,
  slug            TEXT NOT NULL UNIQUE,
  description     TEXT,
  short_desc      TEXT,
  price           INTEGER NOT NULL,                  -- minor units (cents)
  compare_at_price INTEGER,                          -- for "was X" pricing
  category_id     INTEGER REFERENCES categories(id),
  stock_quantity  INTEGER NOT NULL DEFAULT 0,        -- numeric, not boolean
  reserved_qty    INTEGER NOT NULL DEFAULT 0,        -- in pending orders
  low_stock_threshold INTEGER NOT NULL DEFAULT 5,
  status          TEXT NOT NULL DEFAULT 'active',    -- active|draft|archived
  featured        INTEGER NOT NULL DEFAULT 0,
  weight_grams    INTEGER,                           -- for shipping calc
  sku             TEXT UNIQUE,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at      TEXT NOT NULL DEFAULT (datetime('now')),
  deleted_at      TEXT                               -- soft delete
);

CREATE TABLE product_variants (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  product_id      INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  label           TEXT NOT NULL,                     -- e.g. "Large / Red"
  sku             TEXT UNIQUE,
  price_override  INTEGER,                           -- NULL = use product.price
  stock_quantity  INTEGER NOT NULL DEFAULT 0,
  position        INTEGER NOT NULL DEFAULT 0
);

-- blog_posts: 4-state status, scheduled publishing, multi-author ready
CREATE TABLE blog_posts (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  title           TEXT NOT NULL,
  slug            TEXT NOT NULL UNIQUE,
  excerpt         TEXT,
  content         TEXT NOT NULL,
  cover_image     TEXT,
  status          TEXT NOT NULL DEFAULT 'draft',     -- draft|scheduled|published|archived
  published_at    TEXT,                              -- when status went to 'published'
  scheduled_for   TEXT,                              -- if status='scheduled'
  author_id       INTEGER REFERENCES users(id),
  reading_minutes INTEGER,                           -- precompute on save
  featured        INTEGER NOT NULL DEFAULT 0,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- orders: link to customer (if account), shipping address, refund tracking
CREATE TABLE orders (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  order_number    TEXT NOT NULL UNIQUE,              -- e.g. SHOP-20260418-001
  customer_id     INTEGER REFERENCES users(id),      -- NULL for guest orders
  customer_name   TEXT NOT NULL,
  customer_email  TEXT NOT NULL,
  customer_phone  TEXT,
  subtotal        INTEGER NOT NULL,
  shipping_cost   INTEGER NOT NULL DEFAULT 0,
  discount_amount INTEGER NOT NULL DEFAULT 0,
  discount_code   TEXT,
  tax             INTEGER NOT NULL DEFAULT 0,
  total           INTEGER NOT NULL,
  currency        TEXT NOT NULL DEFAULT 'KES',
  status          TEXT NOT NULL DEFAULT 'pending',   -- pending|paid|processing|shipped|delivered|cancelled|refunded
  payment_method  TEXT NOT NULL,                     -- paystack|stripe|whatsapp|mpesa|cod|bank
  payment_ref     TEXT,
  shipping_address TEXT,                             -- JSON blob
  notes           TEXT,
  refunded_amount INTEGER NOT NULL DEFAULT 0,
  refunded_at     TEXT,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- audit_log: required if >1 admin
CREATE TABLE audit_log (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id      INTEGER REFERENCES users(id),
  action       TEXT NOT NULL,                        -- e.g. "product.delete"
  target_type  TEXT,                                 -- e.g. "product"
  target_id    INTEGER,
  metadata     TEXT,                                 -- JSON
  ip_address   TEXT,
  created_at   TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### L5 · Ready-made R2 image loader

Drop-in `next/image` loader so projects don't fall back to raw `<img>` tags. Add this in Phase 6 when shop/blog images first appear.

```ts
// src/lib/r2-image-loader.ts
import type { ImageLoaderProps } from "next/image";

export default function r2Loader({ src, width, quality }: ImageLoaderProps) {
  // src is the R2 object key OR a full URL. We support both.
  const base = process.env.NEXT_PUBLIC_R2_BASE_URL ?? "";
  const url = src.startsWith("http") ? src : `${base}/${src}`;
  // Cloudflare Image Resizing (paid add-on) — falls back to raw URL if disabled.
  return `/cdn-cgi/image/width=${width},quality=${quality ?? 75},format=auto/${url}`;
}
```

```ts
// next.config.ts
const nextConfig: NextConfig = {
  images: {
    loader: "custom",
    loaderFile: "./src/lib/r2-image-loader.ts",
    remotePatterns: [{ protocol: "https", hostname: "*.r2.dev" }],
  },
};
```

If the user does NOT have Cloudflare Image Resizing enabled, simplify the loader to `return url;` — they still get lazy loading + responsive `srcset` from `next/image`, just no on-the-fly resizing.

### L6 · Engineering-level → defaults decision matrix

Use Q5 to choose defaults across the plan:

| Decision | Novice | Intermediate | Expert |
|---|---|---|---|
| Auth | NextAuth.js v5 \u2605 (provider plugins, more docs) | jose \u2605 (no beta dep, minimal surface) | jose \u2605 (or NextAuth v5 if OAuth providers required) |
| Package manager | npm | npm or pnpm | pnpm or bun |
| Commit cadence | Agent commits + pushes after each phase | Agent prints commands, user runs them | Same as intermediate |
| Branch strategy | Single `main` branch | `main` + feature branches | Trunk-based with PRs + branch protection |
| Tests | None in v1 (defer to v2) | Vitest unit tests on `lib/` only | Vitest + Playwright E2E on critical flows (checkout) |
| TypeScript strictness | `strict: true` but tolerate `any` | `strict: true`, no `any` | `strict: true` + `noUncheckedIndexedAccess` |
| Error handling | Try/catch + toast | Same + Sentry | Sentry + structured logging + circuit breakers |
| Schema migrations | Edit single SQL file | Numbered migration files | Migrations + a rollback plan |
| Deployment | Manual `npm run deploy` | Workers Builds CI on push to main | CI with preview env per PR + smoke tests |
| Secrets | `.dev.vars` + dashboard | Same | + secret rotation policy |

When the plan is generated, mention which row was applied and why so the user can override.

### L7 · What to defer vs. what NOT to defer

**Safe to defer to v2 (always offer this):**
- PWA manifest, service worker
- Dynamic OG image generation
- Cookie banner (only needed if non-essential cookies are set)
- Multi-language support (if not in Q34)
- Customer accounts (guest checkout is fine for v1)
- Discount codes
- Reviews / ratings
- Wishlist / favourites
- 3D / AR product viewers
- Blog scheduled publishing UI (schema can support it; UI can wait)
- CSV import/export
- Audit log UI (table can exist; viewer can wait)

**NEVER defer (will block launch or require painful refactor):**
- `next/image` with R2 loader — bandwidth + Lighthouse cost compounds
- Numeric stock quantities (changing from boolean later breaks order logic)
- 4-state blog post status (changing from boolean later breaks every query)
- Order persistence BEFORE payment redirect (lost orders are real money)
- Webhook signature verification (Paystack/Stripe webhooks without HMAC = security hole)
- HTTPS-only cookies (`secure: true, sameSite: 'lax'`) — fixing later means logging everyone out
- `metadataBase` set in `layout.tsx` — broken OG previews are hard to debug after launch
- Robots disallow on `/admin` — admin pages indexed = embarrassment
- Changing admin seed password before launch
- Hard-coded placeholder phone/till numbers — grep for these in Final Review

### L8 · Hosting alternatives matrix (when Q8 ≠ Cloudflare Workers)

| Stack | Free tier | Paid tier starts | When to choose | Caveats |
|---|---|---|---|---|
| **Cloudflare Workers + D1 + R2** \u2605 | 100k req/day Workers, 5GB D1, 10GB R2 + zero egress | $5/mo Workers Paid (10M req incl.) | Default for media-heavy sites, African markets, edge perf. | D1 not at build time; bcrypt N/A; smaller ecosystem |
| **Vercel Hobby + Neon** | Unlimited static, Neon 0.5GB free | Vercel Pro $20/mo, Neon $19/mo | Best Next.js DX; standard Postgres | Egress charges on big images; 100GB-hours serverless |
| **Vercel Hobby + Supabase** | Same | Supabase Pro $25/mo | Want Postgres + auth + storage in one | Supabase storage egress not free |
| **Netlify free + Supabase** | 100GB bandwidth, 300 build min | Netlify Pro $19/mo | Static-heavy sites with light backend | Functions slower cold-start than Vercel |
| **Render free** | 750 hrs/mo Web Service, free Postgres (90-day expiry) | $7/mo Web Service | Long-running Node servers, websockets | Free DB expires; cold starts |
| **Railway** | $5 one-time trial credit | $5/mo + usage | Postgres + cron + workers in one place | Not generous after trial |
| **Fly.io free** | 3 shared-cpu VMs, 3GB persistent vol, Postgres | Pay-as-you-go | Need always-on Node, Docker, regions | More ops knowledge needed |
| **Deno Deploy** | 1M req/mo, KV included | Pay-as-you-go | Edge JS, simpler than Workers | Smaller community, no R2-equivalent |
| **GitHub Pages + form services** | Unlimited static | $0 | Pure static brochure / portfolio | No backend at all |

**Truly free combinations for a polished launch:**
- Cloudflare Workers free + D1 free + R2 free + Cloudflare Web Analytics free + Resend 3k/mo free + GitHub Actions for CI = **$0/mo total**, supports a real shop up to ~3 req/sec sustained.
- Same with custom domain: domain registration is the only cost (~$10/year on Cloudflare Registrar at-cost pricing).

If user wants **completely zero-cost** including domain: use `*.workers.dev` URL or `*.pages.dev` (Workers/Pages free subdomains).

### L9 · Sample content generation policy

If Q21a / Q23c / Q23d are "yes":

**Products (8–20 items):**
- Use the user's category names from Q4/Q11
- Generate realistic prices in the user's currency (Q2)
- 1–3 sentence descriptions written in the brand voice (warm/professional/playful per Q26)
- For each product, suggest 1–3 image keywords for the user to source from Unsplash, OR use coloured gradient placeholders
- Include 2–3 "out of stock" items so admin UI is testable
- Include 1–2 "featured" items for landing page slots

**Blog posts (3–5):**
- Topics relevant to Q1 purpose
- ~400–600 words each
- Real headings, real paragraphs (not lorem ipsum) — written in brand voice
- Cover-image keyword suggestion per post
- 1 post in `draft` state, 1 in `scheduled`, 3 in `published` — exercises the full status flow
- Include at least one post with code block, one with image, one with list — exercises typography styles

**Categories:**
- 3–6 product categories matching Q4
- 2–4 blog categories if Q20 = yes

The seed file should be re-runnable (use `INSERT OR IGNORE` and predictable IDs).

### L10 · Final Review additions

Add these to the Final Review checklist:

- [ ] No `<img>` tag remains where R2/external image is used (grep `src/`).
- [ ] No `bcrypt` or Node `crypto.createHash` imports (grep — these break on Workers).
- [ ] No `generateStaticParams` on routes that read from D1.
- [ ] No `console.log` of secrets or session tokens.
- [ ] Order rows persist with `status='pending'` BEFORE Paystack redirect.
- [ ] Webhook routes verify HMAC signatures (grep for `crypto.subtle.verify` or equivalent).
- [ ] Newsletter form is wired OR removed (no dead inputs).
- [ ] `metadataBase` set in root `layout.tsx` to production URL.
- [ ] R2 bucket has public dev URL enabled OR custom subdomain configured.
- [ ] Admin seed password changed (grep for `admin123`, `password123`, `changeme`).
- [ ] Placeholder phone numbers replaced (grep `254700000000`, `XXXXXXX`, `+1234567890`).
- [ ] `.dev.vars` is gitignored and not committed.
- [ ] Two R2 buckets exist if using OpenNext incremental cache.
- [ ] `nodejs_compat` flag in `wrangler.toml` (or `wrangler.jsonc`) if any Node-specific lib is imported.

After running the checklist, propose **next-phase candidates** specifically drawn from the omissions in §L3 the user deferred:
- "Inventory & low-stock alerts (Phase 13)?"
- "Customer accounts + order history (Phase 14)?"
- "Search via D1 FTS5 (Phase 15)?"
- "Email receipts via Resend (Phase 16)?"
- "Discount codes (Phase 17)?"
- "Migrate `<img>` to `next/image` with R2 loader (Phase 18)?"

Ask which to plan next and offer to draft the v2 plan.

---

## L11 · Package manager choice (from audit)

| Repo | Manager | Lockfile | Notes |
|---|---|---|---|
| Repo A | npm | `package-lock.json` | Works everywhere, slower install |
| Repo B | pnpm | `pnpm-lock.yaml` | Fastest + disk-efficient; needs `pnpm-workspace.yaml` + `onlyBuiltDependencies` for Workers Builds CI |
| Repo C | npm | `package-lock.json` | No `packageManager` field in `package.json` |

**Recommendation:** default to **npm** for novices (fewer CI surprises). Offer **pnpm** if user says "fast install" or "monorepo". Avoid **bun** until proven on Workers Builds CI (no reference repo uses it yet).

## L12 · Auth library tradeoffs (from audit, as of 18 April 2026)

| Choice | Used by | Version observed | Pros | Cons |
|---|---|---|---|---|
| `jose` custom JWT | Repo A | 6.2.2 | Minimal deps, edge-native, debuggable cookies, no `AUTH_TRUST_HOST` surprises | ~150 lines of boilerplate per project (session, login, logout, middleware) |
| NextAuth v5 beta | Repo B, Repo C | `5.0.0-beta.30` | Provider plugins (Google/GitHub/etc.), D1 adapter available | **Still beta** 18 months after first beta tag; API has churned; requires `AUTH_TRUST_HOST=true` or silent redirect-loops on Workers |
| NextAuth v4 stable | — | `4.x` | Mature, many StackOverflow answers | Edge runtime support is rough; D1 adapter is community-maintained |
| Clerk | — | — | Fastest UX, no code needed | Paid above 10k MAU; adds an external dependency |

**Default recommendation (April 2026):** **NextAuth v5 beta for novices and simple sites** (Scope A/B/C/F or Q5 = novice) — the provider plugins and ecosystem docs save hours for first-time builders, and the beta has shipped in two production reference repos. **`jose` for everything else** — larger shops, intermediate/expert users, anyone who wants to avoid a beta dep on the auth path. Revisit once NextAuth 5.0.0 GA ships.

## L13 · Growth-phase feature roadmap (what Repo B shipped that others deferred)

When the user is ready for v2, these are the features Repo B layered on top of the MVP in the order of visible ROI. Use them as the Phase 13–18 sequence:

1. **Product variants** (`product_variants` table with per-variant price + stock).
2. **Coupons / discount codes** (`coupons` table with percent/fixed + usage tracking).
3. **Shipping zones** (`shipping_zones` with base rate + per-kg + free-above threshold).
4. **Customer accounts + wishlist** (NextAuth users + `wishlists` junction + `user_addresses`).
5. **Reviews + ratings** (`reviews` table, admin-moderated with `is_approved` flag).
6. **Email receipts via Resend** (`resend` npm dep, React Email templates).
7. **Inventory alerts** (`inventory_alerts` table + low-stock threshold).

## L14 · Network-check and self-update duty (agent responsibility)

The skill is dated — packages and Cloudflare APIs change. Before Phase 1 the agent **must**:

1. **Check package currency.** Run `npm view next version`, `npm view @opennextjs/cloudflare version`, `npm view wrangler version`, `npm view next-auth@beta version`. If any observed in the audit (Next 16.2.4, @opennextjs/cloudflare 1.19.1, wrangler 4.83.0, next-auth 5.0.0-beta.30) is now **>2 minor versions behind**, pin to the latest stable and note the bump in the plan.
2. **Check Cloudflare compatibility_date.** Suggest today's date (or within the last 30 days) for new projects; for existing repos flag any `compatibility_date` older than 6 months.
3. **Verify the `middleware.ts` vs `proxy.ts` convention.** Grep `node_modules/next/dist/` for the current filename Next.js expects. Do not rely on this SKILL.md's cached claim.
4. **Check Paystack / Stripe API versions.** If the user picked a payment provider, `fetch_webpage` the provider's "API changelog" page and surface any breaking changes since the reference implementation.
5. **Self-update when corrections are found.** After the project ships, if the agent discovers a claim in this SKILL.md is now wrong (e.g., new Next.js rename, `@opennextjs/cloudflare` API change), it should:
   - Edit the local `.github/skills/cloudflare-website-builder/SKILL.md` in the project to record the correction in a `## L15 · Agent notes` section with date + evidence.
   - Suggest the user upstream the fix to any "master" copy of the skill they maintain.

## L15 · Agent notes (append new findings here with date + repo evidence)

*(Agents: append new corrections to this section in chronological order. Keep entries short: `YYYY-MM-DD — finding — evidence path/URL`.)*

- `2026-04-18` — `middleware.ts` is still the correct Next.js 16 filename; no `proxy.ts` rename observed across 3 Cloudflare repos. Evidence: `Repo A/src/middleware.ts`, `Repo C/middleware.ts`, Repo B (confirmed by file listing).
- `2026-04-18` — Only 1/3 Cloudflare repos uses a second R2 bucket for OpenNext cache. Evidence: `Repo A/wrangler.jsonc` has `NEXT_INC_CACHE_R2_BUCKET`; `Repo B/app/wrangler.toml` and `Repo C/wrangler.toml` have only `MEDIA`.
- `2026-04-18` — NextAuth v5 is still on `5.0.0-beta.30` in two shipped production sites. Recommend jose for new novice-level projects until GA.
- `2026-04-18` — `Repo C/wrangler.toml` has `compatibility_date = "2024-09-23"` (19 months stale). Confirms AP14.
- `2026-04-18` — Added [B6 Plan Versioning Rule](#b6--plan-versioning-rule-never-overwrite-old-plans) after `Repo A` audit produced `IMPLEMENTATION_V2.md`. Old `Implementation.md` retained with banner pointing at v2; pattern to repeat for every future vN. Evidence: `Repo A/Implementation.md` (lines 1-13 banner) + `Repo A/IMPLEMENTATION_V2.md` (front-matter `Replaces:` line).

