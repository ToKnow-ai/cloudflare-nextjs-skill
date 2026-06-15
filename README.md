# Cloudflare Next.js Website Builder — Agent Skill

A reusable **agent skill** that takes you from an empty folder to a **production-grade**
website on **Cloudflare Workers** with **Next.js 16**, Cloudflare **D1** (database), and
**R2** (file storage). An online store, a blog, a checkout flow, or all three, built for
real traffic and running **entirely on the free tier**.

> **Free and production-ready are the whole point.** This is not a throwaway demo that
> falls over under load. The same site can scale to **millions of database reads a day**
> with no hosting bill, because Cloudflare's free tier is genuinely generous and the skill
> is built to use it correctly, with security in mind from the first commit.

It is written for novices: the agent **asks plenty of questions first**, proposes a phased
implementation plan, then builds one verifiable slice at a time with commit checkpoints. It
was distilled from several real, shipped production websites, so the defaults reflect what
actually worked in production rather than generic tutorial advice.

## Production-ready, and free

Everything below runs at **$0/month** on Cloudflare's free tier:

| What you get | On the free tier |
|---|---|
| **Hosting** (Workers) | 100,000 requests/day, served from Cloudflare's global edge |
| **Database** (D1, SQLite) | 5 GB storage, **5 million reads/day**, 100k writes/day |
| **File storage** (R2) | 10 GB storage, **zero egress fees** (no bandwidth charges) |
| **Blog** | D1-backed posts or in-repo MDX, free |
| **Analytics** | Cloudflare Web Analytics, free and privacy-friendly |
| **Email** | Resend free tier (3,000/month) for receipts and replies |
| **TLS / CDN / DDoS protection** | Included, free |

A complete shop with a blog and an admin dashboard launches at **$0/month** and scales as
you grow. The only optional cost is registering a custom domain when you are ready.

## Built for production, with security in mind

The skill bakes in the things that separate a real, production-grade site from a tutorial:

- **Edge-correct auth** — Workers-compatible password hashing (PBKDF2 via Web Crypto; the
  common `bcrypt` library does not run on Workers), HTTP-only secure cookies.
- **Safe payments** — verified webhook signatures, and orders persisted *before* the payment
  redirect so none are ever lost.
- **Solid data modelling** — numeric stock, proper order tables, and draft/scheduled/published
  blog states chosen up front, so you never hit a painful migration later.
- **SEO and accessibility** — sitemap, robots, structured data, semantic HTML, WCAG AA.
- **Secrets done right** — never committed; local `.dev.vars` plus `wrangler secret put` in
  production.

## It asks questions first

Instead of generating code blindly, the skill **interviews you before it builds**:

- A **3-question fast path** for beginners (project name, country/currency, inspiration sites),
  then smart, production-ready defaults.
- Or a **full interview** (up to 45 questions) for a complete shop: payments, auth, blog,
  shipping, inventory, SEO, and more.

It then writes a **phased plan**, builds **one tested slice at a time**, and commits after each
phase, so you always have a working site rather than a half-finished pile of code.

## What the skill does

- **Bootstraps** an empty workspace (git init, identity, first commit, optional GitHub remote).
- Runs a **scope picker** (one-pager, blog, shop, booking, directory, SaaS, etc.) so you
  are not asked 40 questions for a landing page.
- Offers a **Novice Fast-Path** (3 questions + smart defaults) or an **Advanced** full interview.
- Reads any **inspiration websites** you supply and turns them into theme and content decisions.
- Writes a **phased plan** (`IMPLEMENTATION.md`) and executes it under a strict, phase-gated
  protocol with local verification and commit checkpoints.
- Surfaces every **manual step the agent cannot do for you** (Cloudflare sign-up, DNS, custom
  domain, GitHub↔Cloudflare CI, Paystack live keys, secrets) as explicit checklists.
- Generates **placeholder assets** (SVG/PNG/video) so the site looks complete, and tracks them
  for replacement before launch.

## How to use it

The skill is a single self-contained [`SKILL.md`](SKILL.md). Use it in whichever agent you prefer:

1. **VS Code (GitHub Copilot agent skills):** copy the file to your project at
   `.github/skills/cloudflare-website-builder/SKILL.md`. The agent discovers it automatically.
2. **Claude Code / Claude.ai:** place the folder under your skills directory, or upload it.
   See the [Agent Skills standard](https://agentskills.io/).
3. **Any chat agent:** paste the contents of `SKILL.md` into the conversation and say
   "build me a website".
4. **As a git submodule** (keep it updated across projects):
   ```bash
   git submodule add https://github.com/ToKnow-ai/cloudflare-nextjs-skill.git tools/cloudflare-nextjs-skill
   # then copy or symlink tools/cloudflare-nextjs-skill/SKILL.md into .github/skills/cloudflare-website-builder/
   ```

Then just tell the agent what you want: *"I want a small online shop for my bakery."*
The skill takes over from there.

## Keeping it current

Cloudflare and Next.js move fast. Before scaffolding, ask your agent to confirm the latest
versions and any breaking changes — the skill includes a "Research the latest changes online"
section with the exact pages to check (`@opennextjs/cloudflare` on npm, the OpenNext Cloudflare
docs, the Cloudflare Workers Next.js guide, and the Next.js release notes).

## License

MIT © ToKnow.ai. The skill is provided as-is for educational and practical use; always test in
your own environment before relying on it for production.
