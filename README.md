# Craaft: LLM Skill

A [Claude Code Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches Claude how to talk to the [Craaft](https://craaft.io) Kanban JSON API: auth, error shapes, rate limits, the full v1 surface, and the bugs Claude reliably introduces if left to guess.

## Install

### Claude Code plugin (recommended)

The `craaft` plugin bundles this skill and the craaft MCP server, so your agent can read boards, add and move cards, and search, as well as script against the REST API. Set your token first (mint one in craaft, Settings, then API keys; it is shown once):

```bash
export CRAAFT_API_TOKEN=cra_...
claude plugin marketplace add craaft/skill
claude plugin install craaft@craaft
```

Or from inside Claude Code: `/plugin marketplace add craaft/skill`, then `/plugin install craaft@craaft`.

The MCP server reads `CRAAFT_API_TOKEN` from your environment (Claude Code 2.1.238 or later), so the token never goes into a config file. Updates arrive through `/plugin`; turn on auto-update for the marketplace there, or run `claude plugin update craaft@craaft`.

### Skill only

One-liner - installs into your user-level Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/craaft-api && \
  curl -fsSL https://raw.githubusercontent.com/craaft/skill/main/plugins/craaft/skills/craaft-api/SKILL.md \
    -o ~/.claude/skills/craaft-api/SKILL.md
```

Or, to scope it to a single project, run from that project's root:

```bash
mkdir -p .claude/skills/craaft-api && \
  curl -fsSL https://raw.githubusercontent.com/craaft/skill/main/plugins/craaft/skills/craaft-api/SKILL.md \
    -o .claude/skills/craaft-api/SKILL.md
```

The skill used to live at the repo root; installs made from the old `main/SKILL.md` URL should re-run the command above.

## Layout

```
.claude-plugin/marketplace.json            # this repo is a plugin marketplace named "craaft"
plugins/craaft/.claude-plugin/plugin.json  # the "craaft" plugin
plugins/craaft/.mcp.json                   # craaft MCP server, bearer token from $CRAAFT_API_TOKEN
plugins/craaft/skills/craaft-api/SKILL.md  # the skill
```

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
