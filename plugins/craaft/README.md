# craaft for Claude Code

craaft is a kanban board shared by people and their AI agents. This plugin puts Claude Code on your craaft boards.

## What's in it

- **The craaft MCP server** at `https://ai.craaft.io/mcp`. Claude can list your boards, read cards, add and move cards, comment, tick off checklist items, manage milestones and search across every board you can see. It runs with your account, so Claude sees exactly the boards you do and nothing else.
- **The `craaft-api` skill.** When you ask Claude to script against the craaft REST API (curl, the Python SDK, a cron job), the skill loads the API's rules: auth, error shapes, rate limits, every endpoint, and the mistakes Claude makes when it guesses, such as sending a column's id where the API wants its key.

## Signing in

The first time Claude Code connects, run `/mcp`, pick `craaft` and sign in with your craaft account in the browser. craaft uses OAuth, so there is no token to copy, and you can revoke Claude Code's access from craaft at any time. If you don't have an account, craaft is free for 3 projects, with no credit card: https://craaft.io

## What it sends where

The plugin runs no local code. It connects to 1 server, `https://ai.craaft.io/mcp`, run by craaft, which calls the craaft API at `https://craaft.io/api/v1` on your behalf. Board data you ask Claude to read or change goes between Claude Code and craaft only. The skill is documentation: it runs nothing itself, and the API calls Claude writes from it go to `https://craaft.io/api/v1` with a personal access token you supply.

## Links

- craaft and Claude: https://craaft.io/mcp
- API reference: https://craaft.io/docs
- Source for this plugin: https://github.com/craaft/skill
- Source for the MCP server: https://github.com/craaft/mcp

MIT licensed.
