---
name: craaft-api
description: Use when interacting with the Craaft Kanban JSON API - creating projects/columns/cards/comments, uploading card attachments, listing data across boards, drag-dropping cards via PATCH, or running any automation that authenticates with a `cra_*` bearer token. Triggers when the user mentions "Craaft API", "Craaft personal access token", a `cra_…` token, the Craaft Python SDK, or a Craaft host (`craaft.io` or self-hosted). Does NOT apply to browser/SPA flows - those use cookie-based session auth and CSRF.
---

# Craaft API skill

## When to use

This skill applies when:
- The user asks to call the Craaft API directly (curl, fetch, requests, Python SDK).
- A `cra_*` bearer token appears in the conversation, env, or instructions.
- The user wants to automate Craaft - cron jobs, scripts, integrations.
- The user asks "how do I create / move / list cards via the API".

**Do NOT use** when:
- The user is logged into the SPA in a browser - that path uses cookies + CSRF, not bearer auth.
- The user wants to manage API keys themselves (the `/api-keys` endpoint requires session auth, not key auth, to avoid recursion).

## Base URL + auth

```
BASE = <PUBLIC_BASE_URL>/api/v1     # NEVER /api/ - unversioned routes 404 since 2026-05-06
TOKEN = cra_<24 base64url chars>    # 36 chars total, 192 bits entropy
```

Header on every request:

```
Authorization: Bearer cra_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Scheme is case-insensitive (`Bearer` / `bearer` both work). The literal `cra_` prefix is required - tokens without it 401 fast (Resolver short-circuits before hitting the DB).

**Tokens are minted once, never recoverable.** The user creates one in Settings → API keys. Only sha256(token) is stored. Never echo a full token back to the user; never log it; never embed it in shell history (use env vars or stdin).

## Error shapes

All errors are `{"error": "<human readable>"}`. Status codes:

| Status | Meaning | Recovery |
|---|---|---|
| 400 | Malformed body / missing field / invalid `?type` | Fix payload, retry |
| 401 | Missing / bad / revoked token; sets `WWW-Authenticate: Bearer realm="craaft API"` | Ask user for a fresh token |
| 402 | Plan limit — Free tier project cap (3) **or** attachment uploads on a Free workspace; body has `{error, limit, currentPlan, ...}` | Surface upgrade path |
| 403 | Authenticated but not authorized; almost never via token | Stop, surface to user |
| 404 | Resource missing OR caller lacks board access - intentionally indistinguishable | Don't probe; report |
| 409 | Duplicate (email/username taken, column non-empty, etc.) | Adjust input |
| 413 | Upload exceeds 25 MiB attachment cap | Shrink file or split workflow |
| 429 | Rate limit; carries `Retry-After: <seconds>` | Sleep, retry |
| 5xx | Server bug | Retry once with backoff; report after 2 |

## Rate limits

Per-token bucket: **60 burst + 1 req/sec sustained.** In-memory per Cloud Run replica - the effective ceiling scales with horizontal replicas.

When automating: don't fan out > 30 parallel requests per token. For bulk imports, throttle to ~1/sec or batch. Honor `Retry-After` strictly; it's the next-token-available delay.

## Surface (versioned: `/api/v1/...`)

Authoritative spec is served at **`<HOST>/openapi.yaml`** (no auth required). Fetch the live URL when you need the current shape - the running binary serves it directly:

```bash
curl -s "$HOST/openapi.yaml" > /tmp/craaft-openapi.yaml
```

Highlights:

| Method | Path | Purpose |
|---|---|---|
| GET | `/me` | Current user |
| PATCH | `/me` | Partial update of name / email / username |
| GET | `/projects` | List your projects |
| POST | `/projects` | Create (`name`, `description?`) |
| GET | `/projects/{id}` | Single project + columns + counts |
| PATCH | `/projects/{id}` | Partial update (incl. `visibility`, `backgroundColor`) |
| DELETE | `/projects/{id}` | Cascading delete |
| GET | `/projects/{id}/export` | JSON export (project, columns, cards, comments, attachment metadata) |
| POST | `/projects/{id}/share` | Enable public read-only sharing → `{publicToken}` |
| DELETE | `/projects/{id}/share` | Revoke sharing |
| POST | `/projects/{id}/columns` | Add column (`title`) |
| GET | `/projects/{id}/cards` | All cards in a project |
| POST | `/projects/{id}/cards` | Create card (`title`, `column`, `position`, optional `description` only) |
| GET | `/projects/{id}/tags` | Distinct tags on the board's cards |
| GET | `/cards/{id}/attachments` | List attachments on a card |
| POST | `/cards/{id}/attachments` | Upload file (`multipart/form-data`, field `file`, max 25 MiB; Pro workspace) |
| PATCH | `/columns/{id}` | Update (title/color/position/isDone/cardLimit) |
| POST | `/columns/{id}/archive` | Archive every card in the column |
| DELETE | `/columns/{id}` | 409 if non-empty |
| PATCH | `/cards/{id}` | Update card metadata + same-board drag-drop |
| DELETE | `/cards/{id}` | Delete |
| POST | `/cards/{id}/move` | Move card to another board (`targetProjectId`, `column`) |
| GET | `/cards/{id}/events` | Activity log (moves, priority, assignee), oldest-first |
| GET | `/cards/{id}/comments` | List, oldest-first |
| POST | `/cards/{id}/comments` | Add comment (`body`) |
| GET | `/attachments/{id}` | Download attachment bytes |
| DELETE | `/attachments/{id}` | Delete attachment |
| GET | `/cards/upcoming` | Cross-project due-dated cards |
| GET | `/cards/focus` | Focus envelope: `due` / `attention` / `hygiene` counts |
| GET | `/cards/hygiene?type=ghosts\|stuck\|mine_no_date` | Hygiene drill-down |
| PATCH | `/comments/{id}` | Edit (author only) |
| DELETE | `/comments/{id}` | Delete (author or workspace owner/admin) |
| GET | `/search?q=…&limit=…` | Cross-project ILIKE search |
| GET | `/members` | Workspace members (`joinedAt`, optional `boardAccess`) |
| GET | `/invitations` | Pending invitations |
| POST | `/invitations` | Invite by email + role (+ optional `boardGrants` on `member` invites) |
| GET | `/projects/{id}/members` | Board members (explicit + implicit; each row has `source`) |
| POST | `/projects/{id}/members` | Grant board access `{userId, role}` (`admin` \| `contributor`). Board-admin only |
| PATCH | `/projects/{id}/members/{userId}` | Change explicit grant role |
| DELETE | `/projects/{id}/members/{userId}` | Remove grant (board-admin, or self-remove) |

`GET /me` returns `id, email, name, username, avatarUrl, hasPassword, emailVerified` plus `newsletterSubscribed` / `newsletterAvailable`.

Project responses include `myRole`, `myBoardRole`, `visibility`, and `canUploadAttachments` (reflects the **board's workspace plan**, not the caller's).

Endpoints intentionally **not** exposed via token auth (session + CSRF only): `/auth/*`, `/api-keys`, `/billing/*`, `/me/avatar` upload, `/me/newsletter`, `/workspace`, `/support`.

## Pitfalls Claude commonly gets wrong

1. **`position` is a `float64`, not an integer.** Midpoint reordering: between siblings at 2 and 3, send `2.5`. Head insert above position 1 → `0.5`.

2. **`column` in card payloads is the column `key`, NOT its `id`.** Use `columns[].key` from the project response.

3. **Same-board moves use `PATCH /cards/{id}` with `{column, position}`.** Cross-board moves use **`POST /cards/{id}/move`** with `{targetProjectId, column}`.

4. **`POST /projects/{id}/cards` only accepts `title`, `column`, `position`, optional `description`.** Set `dueDate`, `assignedUserId`, `size`, `priority`, `tags` via **`PATCH /cards/{id}`** after create.

5. **Assignee field is `assignedUserId`, not `assigneeId`.** The server JSON key is `assignedUserId`.

6. **`size` is an optional integer** (estimate), not `xs`/`s`/`m`/`l`/`xl`.

7. **`priority` values are `low`, `medium`, `high`, `urgent`** — there is no `normal`.

8. **`/api/v1/` is mandatory.** `GET /api/me` returns 404.

9. **Don't send CSRF headers on bearer-authed calls.**

10. **List endpoints return `[]` not `null` when empty.**

11. **`dueDate` on PATCH accepts RFC3339 datetimes** (and date strings parse). `createdAt`/`updatedAt` are always RFC3339.

12. **Patches are partial.** Send only changed fields. Use `null` on nullable fields to clear (`dueDate`, `assignedUserId`, `size`, `priority`).

13. **404 means "no access OR doesn't exist".** Per-board access: workspace membership alone doesn't guarantee a board. Check `myBoardRole` / `visibility` on the project.

14. **Attachments upload via `multipart/form-data`** with a single `file` part — not JSON. Requires Pro/Team workspace (`402` on Free). Check `canUploadAttachments` on the project first.

15. **Optimistic state is the SPA's job, not yours.** Call the API and trust the response.

## Common workflows

```bash
HOST=https://craaft.io
TOKEN=cra_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx   # or $CRAAFT_API_TOKEN
```

Never commit real tokens.

### Smoke test

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/me"
```

### Create a card, then set metadata

```bash
CARD=$(curl -s -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"title":"From API","column":"todo","position":1}' \
     "$HOST/api/v1/projects/$PROJECT_ID/cards")

CARD_ID=$(echo "$CARD" | jq -r '.id')

curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"dueDate":"2026-06-15T00:00:00Z","assignedUserId":"<user-uuid>","priority":"high","size":3,"tags":["api"]}' \
     "$HOST/api/v1/cards/$CARD_ID"
```

### Same-board drag-drop

```bash
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"column":"doing","position":4.5}' \
     "$HOST/api/v1/cards/$CARD_ID"
```

### Cross-board move

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"targetProjectId":"<other-project-uuid>","column":"todo"}' \
     "$HOST/api/v1/cards/$CARD_ID/move"
```

### Upload an attachment

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     -F "file=@screenshot.png" \
     "$HOST/api/v1/cards/$CARD_ID/attachments"
```

### Download an attachment

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
     -o screenshot.png \
     "$HOST/api/v1/attachments/$ATTACHMENT_ID"
```

### Focus snapshot

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/cards/focus" | jq '
  {due_count: (.due | length),
   attention_count: (.attention | length),
   hygiene}
'
```

## Python

Official SDK (`pip install craaft`, v1.2+):

```python
import os
from craaft import CraaftClient

with CraaftClient(api_key=os.environ["CRAAFT_API_TOKEN"]) as client:
    project = client.projects.get("<project-id>")
    card = client.projects.create_card(
        project.id, title="From Python", column="todo", position=1.0
    )
    client.cards.update(card.id, priority="high", size=3, tags=["sdk"])
    if project.can_upload_attachments:
        att = client.attachments.upload(card.id, file="screenshot.png")
        data = client.attachments.download(att.id)
```

Env vars: `CRAAFT_API_TOKEN` (required), `CRAAFT_BASE_URL` (optional, default `https://craaft.io/api/v1`).

Raw `requests` works too — same bearer header, honour `Retry-After` on 429.

## When the spec drifts

1. Fetch live spec: `curl -s "$HOST/openapi.yaml"`.
2. If the spec doesn't match the response, read the Go handler in `internal/<resource>/handlers.go` in the main repo.
3. Tell the user the discrepancy; update this skill if the API changed intentionally.

## Don't

- Don't probe for resource existence via 404.
- Don't paginate manually — endpoints return capped lists or explicit limits.
- Don't refresh tokens — they don't expire; revocation is the only lifecycle event.
- Don't store tokens in plain text.
