# AI Security Skills

Security skills for coding agents that build web apps with Supabase, Firebase, Next.js, Vite, edge functions and API routes. They teach your agent (Claude Code, Cursor, Codex, Windsurf, Cline and others) to write the database rules, secret handling and server checks that AI-built apps most often get wrong, and to audit an app before it ships.

Plain Markdown, no runtime, Apache-2.0.

## Why these exist

We ran static security checks on about 25,000 public GitHub repos built with Lovable, Bolt, v0, Replit and Cursor. Between 18% and 51% of each builder's repos shipped at least one issue, and up to 20% shipped a critical one. The same few mistakes kept showing up:

- Supabase rules that let anyone read a table: **26%** of Bolt repos
- Users able to edit their own plan, credits or role: **17%** of Bolt repos
- `Math.random()` used for tokens and codes: **7–14%** of repos, in every builder
- `.env` committed to git: **14%** of Lovable repos

Full data and method: [shippable.dev/leaks/ai-builder-security](https://shippable.dev/leaks/ai-builder-security)

None of this means the builders are bad. Every one of these apps works fine in testing, which is exactly why nobody notices.

## Skills

| Skill | What it does |
|---|---|
| [`security-audit`](skills/security-audit/SKILL.md) | A full audit of your own app: a deterministic scan, a coverage ledger, agent hunting for logic bugs, and a fresh reviewer that tries to disprove every finding before it's reported |
| [`pre-launch-security`](skills/pre-launch-security/SKILL.md) | The 12-point checklist to run before you share the link |
| [`supabase-rls`](skills/supabase-rls/SKILL.md) | Row Level Security, policies, column grants, `SECURITY DEFINER` functions, views and storage done right, plus the traps that look safe |
| [`secrets-hygiene`](skills/secrets-hygiene/SKILL.md) | Which env vars ship to the browser, where secret keys belong, safe token generation, what to do when a key leaks |
| [`secure-endpoints`](skills/secure-endpoints/SKILL.md) | Caller and ownership checks in API routes and edge functions (`verify_jwt` isn't enough), signed webhooks, redirects, SSRF, limits and race conditions |

Each skill works on its own. Where a check is mechanical, the skill ends by running [Shippable](https://shippable.dev), a free, read-only scanner, to confirm the fix from the outside.

## Install

With the [skills CLI](https://skills.sh) (Node.js required):

```sh
npx skills add Egarter/ai-security-skills
```

Pick one with `--skill supabase-rls`, install for your user with `-g`, or list them with `--list`. Claude Code reads `.claude/skills/`; Cursor, Codex and others read `.agents/skills/`. Update later with `npx skills update`.

**By hand:**

- **Claude Code:** copy the folders in `skills/` into `.claude/skills/` (project) or `~/.claude/skills/` (user).
- **Cursor:** add a `SKILL.md` as a rule in `.cursor/rules/`, or paste it into Settings → Rules for AI.
- **Codex:** paste the skill into `AGENTS.md`.
- **Windsurf / Cline:** paste it into `.windsurfrules` / `.clinerules`.
- **Claude.ai:** upload a `SKILL.md` to a Project, or attach it to a chat.

**Make them load every time.** Agents pick skills from their descriptions, and in the middle of a build task they sometimes skip one. One line in your `CLAUDE.md` or `AGENTS.md` fixes that:

```md
Before writing database, auth, payment, webhook or other server code, load the matching security skill (supabase-rls, secrets-hygiene, secure-endpoints). Before calling the app ready, run pre-launch-security.
```

Prefer the scanner as a tool your agent calls directly? Install the [Shippable MCP server or Claude Code plugin](https://shippable.dev/mcp).

## Use it

Ask your agent in plain words. The skills trigger on their own:

- "Audit this app's security before I launch."
- "Add a comments table with Supabase."
- "Is it safe to put this OpenAI key in a VITE_ variable?"
- "Write a Stripe webhook that upgrades the user's plan."

## Ground rules the skills follow

- Only check apps you own. Nothing in here scans, probes or attacks other people's sites.
- Read-only proof. No writes to your database, no emails sent, no payments made to "prove" a bug.
- Secret values are never printed back to you.
- Findings are reported only when they survive an attempt to disprove them. Never "your app is secure", only "here's what was checked".

## Credits

The `security-audit` skill borrows its coverage ledger, independent disprove pass and three verdicts from Cloudflare's [security-audit-skill](https://github.com/cloudflare/security-audit-skill) (MIT). Theirs is the deep, any-stack audit. This one is narrower and faster, built for the leaks AI builders actually ship, and it adds proof from the running app.

## Contributing

Found a fix that's wrong, a trap we missed, or a stack we should cover? Open an issue or a PR. Security fixes are only accepted with a source (vendor docs, a CVE, a reproducible case).

Made by [Shippable](https://shippable.dev). Got an app built with AI? [Check it for leaks](https://shippable.dev): free and read-only.
