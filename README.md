# Cloudflare Next.js Website Builder — Agent Skill

A reusable **agent skill** that walks a person from an empty folder to a live,
production website on **Cloudflare Workers** with **Next.js 16**, Cloudflare **D1**
(database), and **R2** (file storage). It is written for novices: the agent asks
plenty of questions first, proposes a phased implementation plan, then builds one
verifiable slice at a time with commit checkpoints.

It was distilled from several real, shipped production websites and their iterated
implementation plans, so the defaults reflect what actually worked in production rather
than generic tutorial advice.

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
