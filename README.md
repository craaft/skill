# Craaft: LLM Skill

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches Claude how to talk to the [Craaft](https://craaft.io) Kanban JSON API: auth, error shapes, rate limits, the full v1 surface, and the bugs Claude reliably introduces if left to guess.

## Install

### Claude Code plugin (recommended)

The `craaft` plugin bundles this skill and the craaft MCP server, so your agent can read boards, add and move cards, comment and search, as well as script against the REST API.

```bash
claude plugin marketplace add craaft/skill
claude plugin install craaft@craaft
```

Or from inside Claude Code: `/plugin marketplace add craaft/skill`, then `/plugin install craaft@craaft`.

The first time Claude Code connects, run `/mcp`, pick `craaft` and sign in with your craaft account in the browser. There is no token to copy: the MCP server uses OAuth, and Claude Code keeps the session refreshed. Updates arrive through `/plugin`; turn on auto-update for the marketplace there, or run `claude plugin update craaft@craaft`.

Version 1.0.0 of the plugin read a `CRAAFT_API_TOKEN` personal access token from your environment instead. You can unset it once you have signed in; the skill's curl and SDK examples still use one.

### Skill only

One-liner - installs into your user-level Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/craaft-api && \
  curl -fsSL https://raw.githubusercontent.com/craaft/skill/main/skills/craaft-api/SKILL.md \
    -o ~/.claude/skills/craaft-api/SKILL.md
```

Or, to scope it to a single project, run from that project's root:

```bash
mkdir -p .claude/skills/craaft-api && \
  curl -fsSL https://raw.githubusercontent.com/craaft/skill/main/skills/craaft-api/SKILL.md \
    -o .claude/skills/craaft-api/SKILL.md
```

The skill has moved twice, from `main/SKILL.md` and then from `main/plugins/craaft/skills/craaft-api/SKILL.md`. Installs made from either old URL should re-run the command above.

## Layout

The repo root is the plugin, so directories that scan a repo for plugin files (such as cursor.directory) find them at the top level.

```
.claude-plugin/marketplace.json   # this repo is a plugin marketplace named "craaft", whose 1 plugin is the repo root
.claude-plugin/plugin.json        # the "craaft" plugin
.mcp.json                         # craaft MCP server at ai.craaft.io/mcp, OAuth sign-in
skills/craaft-api/SKILL.md        # the skill
```

## What it sends where

The plugin runs no local code. It connects to 1 server, `https://ai.craaft.io/mcp`, run by craaft, which calls the craaft API at `https://craaft.io/api/v1` on your behalf. Board data you ask Claude to read or change goes between Claude Code and craaft only. The skill is documentation: it runs nothing itself, and the API calls Claude writes from it go to `https://craaft.io/api/v1` with a personal access token you supply.

## What it does

When you ask Claude Code to script against Craaft - create cards, move them, list focus items, comment, search, manage board access - and a `cra_*` personal access token is in scope, this skill loads automatically. Claude then knows:

- The bearer auth shape and why `cra_` is mandatory.
- Every error code Craaft returns and how to recover (including `402` for projects and attachment uploads, `413` for oversized files).
- The full `/api/v1/...` surface - projects, cards (including the transactional bulk create/update/move endpoints), comments, checklists, milestones, attachments, focus/hygiene, board access, export/share - and which endpoints are session-only.
- The per-board access model: visibility is granted board-by-board (`/projects/{id}/members`), not by plain workspace membership, so a board you weren't granted reads back as 404, not 403.
- The traps: `position` is a float, `column` is the *key* not the id, single-card create takes no metadata (bulk create does), `assignedUserId` not `assigneeId`, bulk batches are all-or-nothing, attachments are multipart, etc.
- Working `curl` examples and the official Python SDK (`pip install craaft`).

The intent is that you can say *"add a 'review docs' card to my Inbox column with a Friday due date"* and get a correct API call instead of a plausible-looking one.

**Then export a token (mint one in Craaft > Settings > API keys; it's shown once):**

```bash
export CRAAFT_API_TOKEN=cra_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

(`CRAAFT_TOKEN` also works in curl examples; the Python SDK reads `CRAAFT_API_TOKEN`.)

To update a skill-only install, re-run the same `curl`; to remove it, delete the `craaft-api` directory. Plugin installs update and uninstall through `/plugin`.

## Token hygiene

Tokens never expire and are stored only as `sha256(token)` server-side - there is no recovery flow. If a token leaks, revoke it in Settings and mint a new one. Don't paste tokens into scripts you commit; use env vars or a secret manager.

## Reporting drift

`SKILL.md` is an interpretation of the Craaft OpenAPI spec at the time of writing. If a request returns an unexpected field or a `400` mentioning an unknown property, the spec has moved - open an issue or PR with the discrepancy.

## License

MIT - see [`LICENSE`](./LICENSE).
