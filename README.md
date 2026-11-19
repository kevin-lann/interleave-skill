# Interleave agent skill

An [agent skill](https://agentskills.io) that onboards a repo to [Interleave](https://interleaveqa.com):
sign in, install the GitHub App, run `interleave init`, commit the recipe, connect the MCP server and
trigger a first run, all driven by your coding agent while you approve the browser steps.

Works with Claude Code, Cursor, Codex and any agent that reads `SKILL.md` skills.

## Install

```bash
npx skills add kevin-lann/interleave-skill
```

Pick one agent with `-a`:

```bash
npx skills add kevin-lann/interleave-skill -a claude-code
npx skills add kevin-lann/interleave-skill -a cursor
npx skills add kevin-lann/interleave-skill -a codex
```

Or copy `skills/interleave-onboarding` into your agent's skills folder by hand (for Claude Code,
`~/.claude/skills/` or `.claude/skills/` in the repo).

## Use

In the repo you want to test, ask your agent:

> Use the interleave-onboarding skill to set up Interleave in this repo.

The agent checks your environment, walks you through login and the GitHub App, runs
`npx @interleaveqa/cli init` while your app is running locally, and reports the first verdict.

It never sees your secrets: the CLI encrypts them on your machine, and the skill tells the agent not
to ask for, print or commit keys. It asks before committing or pushing.

## What's in here

```text
skills/interleave-onboarding/SKILL.md   the skill
```

## Docs

https://docs.interleaveqa.com

## License

MIT
