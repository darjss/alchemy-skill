# alchemy-skill

An agent skill for [Alchemy v2](https://alchemy.run), the Infrastructure-as-Effects framework on Effect 4. With it, a coding agent writes Stacks, Workers, Lambdas and their bindings the way the installed Alchemy version expects. Without it, agents tend to write v1's `await alchemy(...)` or invent binding names.

It targets `alchemy@2.0.0-beta.84`, where each provider is its own `@alchemy.run/*` package. It also tells the agent what changed for projects still on beta.83 or earlier. Every API it names was checked against the Alchemy source on main at `cb4ec94`. As of 2026-10-23, the beta.84 notes are on main but npm `latest` is still beta.83.

## Install

Claude Code:

```
/plugin marketplace add darjss/alchemy-skill
/plugin install alchemy@alchemy-skill
```

Any agent the [skills CLI](https://github.com/vercel-labs/skills) supports (Claude Code, Codex, Cursor, OpenCode and others):

```sh
npx skills add darjss/alchemy-skill
```

Add `-g` to install for your user instead of the current project, or `-a claude-code` to pick one agent.

By hand: copy `skills/alchemy/` into your agent's skills directory, such as `.claude/skills/alchemy/` or `.agents/skills/alchemy/`.

## What it covers

The skill loads when a project has `alchemy.run.ts`, imports `alchemy` or `@alchemy.run/*`, or when you ask for Alchemy work.

- `SKILL.md` checks the installed version first. Then it covers the model (Stack, Resource, Output, Platform, Binding, the Construction and Runtime phases), Config and secrets, stages and physical names, state stores and credentials, `alchemy dev`, cross-stage references, the CLI, and custom providers. It ends with a table that maps each question to the source file and docs page.
- `references/cloudflare.md` covers Workers, the D1, R2, KV, Queues, rate limit, email, cron and RPC bindings, Website.Vite, D1 migrations, Durable Objects and local emulation.
- `references/aws.md` covers Lambda Functions and Web Functions, `...Http` binding layers, event sources, websites, floci local dev and credentials.

For other providers, the skill points the agent at the matching upstream example and the package source.

It does not cover `@alchemy.run/domain`. Upstream ships its own skill for that: `npx skills add alchemy-run/alchemy --skill alchemy-domain`. For anything else, Alchemy's docs index is at [alchemy.run/llms.txt](https://alchemy.run/llms.txt).

## Layout

```
.claude-plugin/marketplace.json   Claude Code marketplace, one plugin
skills/alchemy/SKILL.md           always read when the skill loads
skills/alchemy/references/        read only for the provider in use
```

The skill follows the [Agent Skills spec](https://agentskills.io/specification). Its frontmatter uses only the spec's fields, so it loads in claude.ai and the Skills API too.

## Updating

```sh
npx skills update
```

In Claude Code, run `/plugin marketplace update alchemy-skill`.

When Alchemy ships a release that breaks something here, the skill's `metadata.alchemy-version` moves with the fix.

## License

MIT
