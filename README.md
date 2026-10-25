# Cockpitify skills for AI coding assistants

[Agent Skills](https://agentskills.io) that teach Claude Code, Cursor, Codex and other coding agents
how to connect an app to [Cockpitify](https://cockpitify.app): usage tracking, AI cost per user,
AdMob ad revenue per user, and running Cockpitify next to your own Postgres. Each skill uses only the public SDK APIs, follows the same rules as the
SDKs' docs, ends with a check in the Cockpitify app, and reports what it changed.

| Skill | Use it to |
|---|---|
| [cockpitify-usage-tracking](cockpitify-usage-tracking/SKILL.md) | record screens and feature events (React Native / Expo, web, Flutter, HTTP) |
| [cockpitify-ai-cost](cockpitify-ai-cost/SKILL.md) | record each user's AI cost on the server (`@cockpitify/node`) |
| [cockpitify-ad-revenue](cockpitify-ad-revenue/SKILL.md) | record AdMob revenue per impression and user (React Native / Expo) |
| [cockpitify-own-postgres](cockpitify-own-postgres/SKILL.md) | run the ops-runner next to your own Postgres (self-managed, RDS, Neon, Cloud SQL) and pair it |

## Install

The skills are plain Markdown, the same for every language your app is written in (Expo, web,
Flutter, a Python or Node server).

```sh
npx skills add cockpitify/cockpitify-skills
```

`npx skills` only copies the folders into your agent's skills directory, so it works for Flutter or
Python projects too (it needs Node on your machine, not in your project). Without Node, copy a
skill's folder into `.claude/skills/` (Claude Code) or your agent's skills directory, or `git clone`
this repository there.

Then ask your assistant, for example: "Add Cockpitify usage tracking to this app", "Connect AI cost
to users", "Connect AdMob revenue to Cockpitify", or "Connect our Postgres to Cockpitify, the code is K7QM-4TXP".

Every Cockpitify package also ships an `AI.md` built from these skills (`@cockpitify/react-native`,
`@cockpitify/js`, `@cockpitify/node`, `ops_flutter`), so an assistant without the skills finds the
same rules in the installed package.


## Before you start

Install Cockpitify into your app's Supabase project once, from the Cockpitify app (Connect → one-tap
install). On your own Postgres, `cockpitify-own-postgres` sets up the ops-runner instead (Connect → Your
own Postgres gives the code). The other skills add code to your app and server; they don't touch your
database.

## Updating

```sh
npx skills update
```

## License

MIT. Cockpitify: https://cockpitify.app · Help: https://cockpitify.app/help/
