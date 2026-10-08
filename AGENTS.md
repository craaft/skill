# AGENTS.md

Instructions for AI coding agents (Claude, Codex, Cursor, Copilot, Gemini and others) working in this repository: the craaft Claude Code plugin, which ships the `craaft-api` skill and the craaft MCP server config.

## Copy: read VOICE.md first

craaft's voice guide lives in the platform repo: `../platform/VOICE.md` when the repos sit side by side, or https://github.com/craaft/platform/blob/main/VOICE.md. **Read it before writing or editing any text a person or an agent will read, and follow it.** Here that means `SKILL.md` (its description and body), the plugin and marketplace manifests' descriptions, the README, commit messages and PR descriptions.

The rules agents break most often:

- Always lowercase "craaft", even at the start of a sentence.
- British spelling: colour, organise, licence (noun).
- No em dashes or en dashes, anywhere. Use a full stop, a comma, a colon or two sentences.
- Numerals for anything countable: "3 projects", "60 requests".
- No hype: no "AI-powered", "supercharge", "seamless", "10x".
- Only claim what the API actually does. `../platform/docs/openapi.yaml` is the source of truth.

Check your text against VOICE.md before you hand it back.

## Working rules

- Bump `version` in both manifests when you ship a change plugin users should pick up, and check them with `claude plugin validate .` and `claude plugin validate ./plugins/craaft`.
- Don't commit, push or open pull requests unless the person you're working for asks you to.
- Never add AI attribution (co-author lines, "generated with" footers) to commits, PRs, code comments or files.
