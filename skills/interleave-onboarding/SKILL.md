---
name: interleave-onboarding
description: Sets up Interleave (cloud QA for pull requests) in the user's repo and connects it to their coding agent. Use when the user asks to onboard, set up or install Interleave, run `interleave init`, connect the Interleave MCP server, or get their first Interleave run going.
---

# Onboard a repo to Interleave

Interleave reads each pull request, writes end-to-end tests, runs them in a cloud sandbox and judges
each failure. This skill takes a repo from nothing to a first verdict, and leaves the agent able to
start runs itself over MCP.

Docs: https://docs.interleaveqa.com (quickstart, CLI reference, `interleave.yml`, MCP server).

## Ground rules

- **Never handle secrets.** Don't ask for, print, paste or commit passwords, tokens or API keys. The
  CLI encrypts secrets on the user's machine; let it. The key it saves at login is all the MCP server needs.
- **The user approves browser steps.** Login approval and GitHub App installation happen in their
  browser. Tell them what to click, then wait. Never try to approve on their behalf.
- **Ask before git writes.** Committing or pushing `interleave.yml` needs a yes. Never force-push or
  touch branches the user didn't name.
- **Don't guess what only the user knows.** How the app starts, which services it needs, test-user
  details. Ask. If `init` asks a question (a line starting with `?`), relay it and pass the answer back.
- **Report honestly.** If a step fails, show the error and what you tried. Don't claim setup worked until
  `init` says "Environment ready".

## Steps

Work through these in order. Skip any that's already done, and say so.

### 1. Check the ground

```bash
git rev-parse --show-toplevel && git remote get-url origin
node --version
npx -y @interleaveqa/cli whoami
```

- Must be a git repo with a GitHub `origin`. Interleave tests what's on GitHub. If there's no remote,
  stop and tell the user.
- Node 20 or later. If older, stop and say so.
- `whoami` failing means not signed in yet: go to step 2. Succeeding: skip to step 3.

Run everything from the repo root. If the user's in a monorepo subfolder, `cd` to the root first.

### 2. Sign in

```bash
npx -y @interleaveqa/cli login
```

It prints a one-time code and opens the dashboard's device page. Tell the user to check the code
matches and click **Approve**. Over SSH or in a container, add `--no-browser` and give them the URL.
Confirm with `whoami`.

### 3. Install the GitHub App

If `init` later says the app isn't installed on this repo, it prints an install link. Pass that link to
the user: they pick the repos Interleave may test ("Only select repositories" is fine). The app reads
code and writes PR comments and checks; it never pushes. They can also do it from **Home** in the
dashboard.

### 4. Bring the app up locally

`init` learns the setup by watching the app run, so it must be running first: database and other
services (Docker Compose, Redis…), then the dev server. Ask the user how they normally start it, start
it in the background, and confirm it responds before continuing. Keep it running until `init` finishes.

If the user would rather not run it, `init` still works from the repo's files, but may miss apps it
can't see. Mention that trade-off and let them choose.

### 5. Run init

```bash
npx -y @interleaveqa/cli init
```

It scans, drafts a recipe, asks the user to confirm a summary, classifies env vars (test keys sealed
and uploaded, live keys blocked), then builds the cloud environment and streams progress. This takes a
few minutes.

- Relay prompts and confirmations to the user. Don't answer confirmations for them.
- Lines starting `repair:` are the repair agent retrying a failed step; let it run.
- A line starting `?` is a question for the user.
- Success ends with `✓ Environment ready`. On failure, read the log it prints, fix what's in the
  user's code or config if it's clearly theirs to fix, and re-run. After two failed attempts, stop and
  show the user the failing step.

To change the recipe afterwards, edit `interleave.yml` and run `init --keep-recipe`. Field reference:
https://docs.interleaveqa.com/interleave-yml

### 6. Commit the recipe

`init` writes `interleave.yml` (and sometimes files under `.interleave/`). It holds `secret://NAME`
references, never values. Show the user `git status`, then **ask** before:

```bash
git add interleave.yml .interleave && git commit -m "Add Interleave recipe" && git push
```

Skip `.interleave` in the `git add` if it doesn't exist.

### 7. Connect the MCP server

So the agent can start runs and read verdicts itself. Skip if the agent already has Interleave tools
(`run_tests`, `get_report`).

The login from step 2 already saved a key, so no key goes in the command. Use the one for the agent
you're running in:

| Agent | Command |
|---|---|
| Claude Code | `claude mcp add interleave -- npx -y @interleaveqa/mcp` |
| Codex | `codex mcp add interleave -- npx -y @interleaveqa/mcp` |
| Cursor | Add the server to `.cursor/mcp.json` (below) |
| Anything else | A stdio MCP server named `interleave`: command `npx`, args `-y @interleaveqa/mcp` |

```json
{ "mcpServers": { "interleave": { "command": "npx", "args": ["-y", "@interleaveqa/mcp"] } } }
```

The tools appear after the client restarts or reloads. Tell the user if they need to do that.

### 8. First run

Push the branch (ask first if it isn't pushed), then:

- With MCP connected: call `run_tests` with `wait: true`.
- Otherwise: `npx -y @interleaveqa/cli run` follows the run live and prints the report.
- Or the user comments `@interleave test` on an open PR.

Read the report. The verdict is one of `passed`, `failed`, `needs_review`, `run_failed`, `cancelled`.
A failed case marked as a regression is a bug in the change. `needs_review` means a person should
look. Summarize it for the user in a few lines, with the report link.

### 9. Wrap up

Tell the user, briefly: what's now set up, the report link, and how to use it from here:

- `@interleave test` on any PR, or ask you to "push, run the Interleave tests and fix regressions".
- Behind a login? A test user lets cases start signed in:
  https://docs.interleaveqa.com/interleave-yml#where-the-test-user-comes-from
- Runs on every push: https://docs.interleaveqa.com/github#automatic-runs

## When something goes wrong

| Symptom | Do this |
|---|---|
| `whoami` says the token was rejected | `npx -y @interleaveqa/cli logout`, then step 2 again. |
| `init` can't find the app | The app isn't running, or is on an unexpected port. Start it, re-run. |
| `init` says the GitHub App isn't installed on the repo | Step 3, then re-run. |
| Setup keeps failing on one step | Show the user the step's log lines. Don't loop. |
| MCP tools don't appear | The client needs a restart or reload. Check `claude mcp list` or the client's MCP panel. |
| Anything else | https://docs.interleaveqa.com/troubleshooting |
