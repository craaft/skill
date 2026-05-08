# craaft-api skill

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches Claude how to talk to the [Craaft](https://craaft.io) Kanban JSON API: auth, error shapes, rate limits, the full v1 surface, and the bugs Claude reliably introduces if left to guess.

## What it does

When you ask Claude Code to script against Craaft — create cards, move them, list focus items, comment, search — and a `cra_*` personal access token is in scope, this skill loads automatically. Claude then knows:

- The bearer auth shape and why `cra_` is mandatory.
- Every error code Craaft returns and how to recover.
- The full `/api/v1/...` surface and which endpoints are intentionally token-inaccessible (`/auth/*`, `/api-keys`, `/billing/*`).
- The traps: `position` is a float (midpoint reordering), `column` in card payloads is the *key* not the id, drag-drop is just `PATCH /cards/{id}`, etc.
- Working `curl` and Python idioms with `Retry-After` handling.

The intent is that you can say *"add a 'review docs' card to my Inbox column with a Friday due date"* and get a correct API call instead of a plausible-looking one.

## Install

One-liner — installs into your user-level Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/craaft-api && \
  curl -fsSL https://raw.githubusercontent.com/craaft/skill/main/SKILL.md \
    -o ~/.claude/skills/craaft-api/SKILL.md
```

Or, to scope it to a single project, run from that project's root:

```bash
mkdir -p .claude/skills/craaft-api && \
  curl -fsSL https://raw.githubusercontent.com/craaft/skill/main/SKILL.md \
    -o .claude/skills/craaft-api/SKILL.md
```

Then export a token (mint one in Craaft → Settings → API keys; it's shown once):

```bash
export CRAAFT_TOKEN=cra_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

To update later, re-run the same `curl`. To remove, delete the `craaft-api` directory.

## Token hygiene

Tokens never expire and are stored only as `sha256(token)` server-side — there is no recovery flow. If a token leaks, revoke it in Settings and mint a new one. Don't paste tokens into scripts you commit; use env vars or a secret manager.

## Reporting drift

`SKILL.md` is an interpretation of the Craaft OpenAPI spec at the time of writing. If a request returns an unexpected field or a `400` mentioning an unknown property, the spec has moved — open an issue or PR with the discrepancy.

## License

MIT — see [`LICENSE`](./LICENSE).
