---
name: craaft-api
description: Use when interacting with the Craaft Kanban JSON API - creating projects (optionally from a board template such as sprint / bug-tracker / sales-pipeline), columns, cards and comments, reading or updating a single card or its full detail view, archiving and restoring cards or listing a board's archived cards, bulk-importing or batch-updating/moving cards, managing checklists and milestones, tags, board members, board access and workspace invitations, uploading card attachments or board backgrounds, following cards, reading the focus/hygiene views, searching across boards, drag-dropping cards via PATCH, configuring outbound webhooks (craaft signed JSON, Slack or Discord delivery, verifying `X-Craaft-Signature`) or email-to-card intake addresses, or running any automation that authenticates with a `cra_*` bearer token. Triggers when the user mentions "Craaft API", "Craaft personal access token", a `cra_…` token, "Craaft webhook", "Craaft email to card", the Craaft Python / JavaScript / PHP SDK, the Craaft MCP server, or a Craaft host (`craaft.io` or self-hosted). Does NOT apply to browser/SPA flows - those use cookie-based session auth and CSRF.
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
| 402 | Plan limit - Free tier project cap (3), attachment uploads, member invites, or integrations (webhooks / email-to-card, `limit: "integrations"`) on a Free workspace; body has `{error, limit, currentPlan, ...}` | Surface upgrade path |
| 403 | Authenticated but not authorized - e.g. `requires board admin` on webhook / email-to-card routes when the caller can see the board but isn't a board admin | Stop, surface to user |
| 404 | Resource missing OR caller lacks board access - intentionally indistinguishable | Don't probe; report |
| 409 | Conflict (email/username taken, column still holds live cards, email-to-card already enabled, etc.) | Adjust input |
| 413 | Upload exceeds 25 MiB attachment cap | Shrink file or split workflow |
| 429 | Rate limit; carries `Retry-After: <seconds>` | Sleep, retry |
| 5xx | Server bug | Retry once with backoff; report after 2 |

## Rate limits

Per-token bucket: **60 burst + 1 req/sec sustained.** In-memory per Cloud Run replica - the effective ceiling scales with horizontal replicas.

When automating: don't fan out > 30 parallel requests per token. For bulk imports, prefer the bulk endpoints (since 2026-07-18) - one request spends one rate-limit token for up to 100 cards, vs 100 tokens for 100 single creates. Honor `Retry-After` strictly; it's the next-token-available delay.

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
| POST | `/projects` | Create (`name`, `description?`, `template?` - a key from `/board-templates`; omitted = `kanban`) |
| GET | `/board-templates` | Template catalogue: `[{key, name, description, columns: [{title, color, isDone}]}]` |
| GET | `/projects/{id}` | Single project + columns + counts |
| PATCH | `/projects/{id}` | Partial update (incl. `visibility`, `backgroundColor`) |
| DELETE | `/projects/{id}` | Cascading delete |
| GET | `/projects/{id}/export?format=json\|csv` | Export the board (project, columns, cards, comments, attachment metadata). Defaults to JSON |
| POST | `/projects/{id}/share` | Enable public read-only sharing → `{publicToken}` |
| DELETE | `/projects/{id}/share` | Revoke sharing |
| POST | `/projects/{id}/columns` | Add column (`title`) |
| GET | `/projects/{id}/cards` | All live cards in a project (**omits `description`** - use `GET /cards/{id}` or `/detail`) |
| GET | `/projects/{id}/cards/archived` | Archived cards (same lean shape + `archivedAt`), newest first, max 200 |
| POST | `/projects/{id}/cards` | Create card (`title`, `column`, `position`, optional `description` only) |
| POST | `/projects/{id}/cards/bulk` | Create up to 100 cards in one transaction - items DO take full metadata |
| POST | `/projects/{id}/cards/rebalance` | Renumber a column's cards to 1, 2, 3, … in request order (up to 10 000 ids) |
| GET | `/projects/{id}/tags` | Distinct tags on the board's cards (includes archived cards) |
| POST | `/projects/{id}/background-image` | Upload a board background (`multipart/form-data`, field `file`, max 10 MiB). Board-admin only |
| GET | `/projects/{id}/background-image` | Download the board background bytes |
| DELETE | `/projects/{id}/background-image` | Remove the background. Board-admin only |
| GET | `/projects/{id}/events` | **SSE stream** (realtime), NOT an activity log - see pitfall 19 |
| GET | `/cards/{id}/attachments` | List attachments on a card |
| POST | `/cards/{id}/attachments` | Upload file (`multipart/form-data`, field `file`, max 25 MiB; Pro workspace) |
| PATCH | `/columns/{id}` | Update (title/color/position/isDone/cardLimit) |
| POST | `/columns/{id}/archive` | Archive every live card in a **done** column → `{archived, ids}` |
| DELETE | `/columns/{id}` | 409 if it holds live cards; its archived cards move to the first remaining column |
| GET | `/cards/{id}` | Fetch one card (same shape PATCH returns, including `following` and `description`) |
| GET | `/cards/{id}/detail` | Card (with description) + comments + events (newest 100, oldest-first) + checklist + attachments |
| POST | `/cards/{id}/archive` | Archive one card → `{archived: true, id}` |
| POST | `/cards/{id}/restore` | Un-archive → full card, back in its original column + position |
| PATCH | `/cards/{id}` | Update card metadata + same-board drag-drop |
| PATCH | `/cards/bulk` | Up to 100 partial updates in one transaction (items are `{id, ...patch}`) |
| DELETE | `/cards/{id}` | Delete |
| POST | `/cards/{id}/move` | Move card to another board (`targetProjectId`, `column`) |
| POST | `/cards/bulk/move` | Sweep/move up to 100 cards (`ids`, `column`, optional `targetProjectId`) |
| GET | `/cards/{id}/events` | Activity log (moves, priority, assignee), oldest-first |
| GET | `/cards/{id}/comments` | List, oldest-first |
| POST | `/cards/{id}/comments` | Add comment (`body`, max 5000 chars) |
| POST | `/cards/{id}/follow` | Follow a card. Idempotent, `204`, no body |
| DELETE | `/cards/{id}/follow` | Unfollow. Idempotent, `204`, no body |
| GET | `/cards/{id}/checklist` | List checklist items, ordered by position |
| POST | `/cards/{id}/checklist` | Add item (`text`, max 1000 chars); appends to the end |
| PATCH | `/checklist/{id}` | Update item (`text` and/or `done`) |
| DELETE | `/checklist/{id}` | Delete item |
| GET | `/projects/{id}/milestones` | List milestones (date asc) |
| POST | `/projects/{id}/milestones` | Add (`name`, `dueOn` as `YYYY-MM-DD`); board-admin only |
| PATCH | `/milestones/{id}` | Update (`name`, `dueOn`, `achieved` bool); board-admin only |
| DELETE | `/milestones/{id}` | Delete; board-admin only |
| GET | `/attachments/{id}` | Download attachment bytes |
| DELETE | `/attachments/{id}` | Delete attachment |
| GET | `/cards/upcoming` | Cross-project due-dated cards |
| GET | `/cards/focus` | Focus envelope: `due` / `attention` / `hygiene` counts |
| GET | `/cards/hygiene?type=ghosts\|stuck\|mine_no_date` | Hygiene drill-down |
| PATCH | `/comments/{id}` | Edit (author only) |
| DELETE | `/comments/{id}` | Delete (author or workspace owner/admin) |
| GET | `/search?q=…&limit=…` | Cross-project ILIKE search (`description` is a ≤180-char snippet; hits carry `archived`). `limit` defaults to 20, values over 50 clamp to 50 |
| GET | `/members` | Workspace members (`joinedAt`, optional `boardAccess`) |
| PATCH | `/members/{userId}` | Change a member's workspace role. Owner/admin only |
| DELETE | `/members/{userId}` | Remove from the workspace (also closes their SSE streams). Owner/admin only |
| GET | `/invitations` | Pending invitations |
| POST | `/invitations` | Invite by email + role (+ optional `boardGrants` on `member` invites) → `{invitation, consumed}` wrapper. Pro only |
| DELETE | `/invitations/{id}` | Revoke a pending invite so its accept link stops working. Owner/admin only |
| GET | `/projects/{id}/members` | Board members (explicit + implicit; each row has `source`) |
| POST | `/projects/{id}/members` | Grant board access `{userId, role}` (`admin` \| `contributor`). Board-admin only |
| PATCH | `/projects/{id}/members/{userId}` | Change explicit grant role |
| DELETE | `/projects/{id}/members/{userId}` | Remove grant (board-admin, or self-remove) |
| GET | `/projects/{id}/webhooks` | `{webhooks: [...], eventCatalogue: [...]}`. Board admin, Pro |
| POST | `/projects/{id}/webhooks` | Create `{url, description?, format?, events?}` → 201 subscription. Board admin, Pro |
| PATCH | `/webhooks/{id}` | Partial update `{url?, description?, format?, events?, active?}`. `{id}` is the subscription id. Board admin, Pro |
| DELETE | `/webhooks/{id}` | `204`. Board admin, **not** plan-gated |
| GET | `/projects/{id}/inbound-email` | `{enabled: false, address: null}` or `{enabled: true, address}`. Board admin, Pro |
| POST | `/projects/{id}/inbound-email` | Enable email-to-card `{targetColumn?}` → 201 address; 409 if already enabled. Board admin, Pro |
| PATCH | `/projects/{id}/inbound-email` | `{active?, targetColumn?, rotate?}` → address. Board admin, Pro |
| DELETE | `/projects/{id}/inbound-email` | `204`. Board admin, **not** plan-gated |

These four need no `Authorization` header at all:

| Method | Path | Purpose |
|---|---|---|
| GET | `/public/projects/{token}` | Read-only board snapshot for a share token |
| GET | `/public/projects/{token}/background-image` | That board's background bytes |
| GET | `/users/{id}/avatar` | A user's uploaded avatar bytes (`404` when they have none) |
| GET | `/version` | Build/version info, useful as a liveness probe |

`GET /me` returns `id, email, name, username, avatarUrl, hasPassword, emailVerified` plus `newsletterSubscribed` / `newsletterAvailable`.

**Card responses** carry `id, projectId, column, title, description, position, dueDate, assignedUserId, assignedUserName, size, priority, tags, createdBy, createdByName, updatedBy, updatedByName, attachmentCount, checklistDone, checklistTotal, following, createdAt, updatedAt`. `checklistDone` / `checklistTotal` are denormalized counts, so you get checklist progress without a second call; `following` is scoped to the authenticated caller. **`GET /projects/{id}/cards` omits `description`** (the board list is lean); fetch `GET /cards/{id}` or `/detail` when you need the body. Search hits return a ≤180-character `description` snippet.

**Project responses** include `myRole`, `myBoardRole`, `visibility`, `canUploadAttachments` (reflects the **board's workspace plan**, not the caller's), plus `isFavorite`, `publicToken` (non-empty only while sharing is on), `backgroundImage` / `backgroundColor` (mutually exclusive), `colorScheme`, `textColor`, `totalCards`, `columnCounts`, `workspaceName`, `members` and `columns`.

Endpoints intentionally **not** exposed via token auth (session + CSRF only): `/auth/*`, `/api-keys`, `/billing/*`, `/admin/*`, `/me/avatar` upload+delete, `/me/newsletter`, `/workspace`, `/support`.

## Archive and restore (documented 2026-09-28)

- `POST /cards/{id}/archive` hides a card from the board without deleting it
  → `{"archived": true, "id": "..."}`. `404` if it is already archived.
- `POST /cards/{id}/restore` un-archives it → the full card (same shape as
  `GET /cards/{id}`), back in the column and position it was archived from.
  `404` if the card is not archived.
- `GET /projects/{id}/cards/archived` lists archived cards, most recently
  archived first, capped at 200 (older ones stay reachable via search).
- `POST /columns/{id}/archive` archives every live card in a column marked
  `isDone` and returns `{"archived": <count>, "ids": [...]}`; the `ids` let
  you undo by restoring each one.
- Realtime / webhook events: `card.archived` (payload `{id, column}`, same as
  `card.deleted`) and `card.restored` (payload = full card).

## Integrations: webhooks and email-to-card (Pro, board admin)

Both are gated the same way, checked in this order: board not visible →
`404`; visible but caller isn't a board admin → `403 requires board admin`;
Free workspace → `402` with `limit: "integrations"`. `DELETE` skips the plan
check so a downgraded workspace can still clean up.

**Outbound webhooks.** Subscription shape:
`{id, endpointId, url, secret, description, format, events, active, createdAt, recentDeliveries: [{event, status, statusCode, attempts, error, createdAt}]}`.

- `format`: `craaft` (default, signed JSON envelope `{event, occurredAt, projectId, data}`),
  `slack` or `discord` (a chat message POSTed to their incoming-webhook URL,
  **unsigned** - those services authenticate by the secret URL itself).
- `events`: filter of names from `eventCatalogue`; omitted or `[]` = all.
  Unknown names, unknown formats and bad URLs are `400`.
- Event catalogue (2026-09-28): `card.created`, `card.updated`,
  `card.deleted`, `card.archived`, `card.restored`,
  `card.comment.{created,updated,deleted}`,
  `card.checklist.item.{created,updated,deleted}`,
  `column.{created,updated,deleted}`.
- `craaft` deliveries carry `X-Craaft-Signature: t=<unix>,v1=<hex>` where
  `v1 = HEX(HMAC-SHA256(secret, "<t>." + raw_body))`. Verify against the raw
  bytes, never re-serialised JSON.
- URL must be absolute `http(s)`; private / loopback / link-local targets are
  rejected (and re-checked at delivery), so `localhost` receivers won't work.
- Delivery is best-effort: up to 3 attempts with backoff, then logged in
  `recentDeliveries`. No replay endpoint.

**Email-to-card.** One intake address per board. Address shape:
`{email, token, targetColumn, active, createdAt}` where `email` is
`<token>@<deployment inbound domain>`. Mail becomes a card (subject → title,
body → description) only when the sender is a workspace member and SPF + DKIM
pass; everything else is dropped silently. `targetColumn` is a column **key**;
`""` on PATCH clears it (first column), and a key that no longer exists also
falls back to the first column. `rotate: true` mints a new token - the old
address stops accepting mail immediately.

## Bulk card operations (since 2026-07-18)

Three endpoints batch card work: `POST /projects/{id}/cards/bulk`
(create), `PATCH /cards/bulk` (update), `POST /cards/bulk/move` (move).
Shared rules:

- **Max 100 items**; request body limit 1 MiB (vs 64 KiB elsewhere).
- **All-or-nothing transactions.** One bad item rolls back the whole
  batch; the error names the offending index: `{"error":"cards[3]: title is required"}`.
  Never assume a failed batch was partially applied - it wasn't.
- **No notification emails** (mentions / moves / assignments stay
  silent). Activity events and realtime SSE fire normally.
- Responses are `{"cards":[...]}` in request order (create returns 201,
  the others 200).
- Bulk create items take **full metadata** (`dueDate`, `assignedUserId`,
  `size`, `priority`, `tags`) - unlike the single create. Omitted
  `position` appends to the end of the column in request order. The
  assignee is NOT defaulted to the caller (single create does that).
- Bulk update items are `{id, ...}` plus any single-PATCH field, same
  semantics (present = apply, `null` = clear, absent = leave).
- Bulk move without `targetProjectId` requires every id on the SAME
  board (400 otherwise); with it, moves the batch to that board (same
  workspace only). Cards append to the end of the target column.

## Pitfalls Claude commonly gets wrong

1. **`position` is a `float64`, not an integer.** Midpoint reordering: between siblings at 2 and 3, send `2.5`. Head insert above position 1 → `0.5`.

2. **`column` in card payloads is the column `key`, NOT its `id`.** Use `columns[].key` from the project response.

3. **Same-board moves use `PATCH /cards/{id}` with `{column, position}`.** Cross-board moves use **`POST /cards/{id}/move`** with `{targetProjectId, column}`.

4. **`POST /projects/{id}/cards` only accepts `title`, `column`, `position`, optional `description`.** Set `dueDate`, `assignedUserId`, `size`, `priority`, `tags` via **`PATCH /cards/{id}`** after create - or create via **`POST /projects/{id}/cards/bulk`**, whose items accept full metadata directly (works fine with a single item).

5. **Assignee field is `assignedUserId`, not `assigneeId`.** The server JSON key is `assignedUserId`.

6. **`size` is an optional integer** (estimate), not `xs`/`s`/`m`/`l`/`xl`.

7. **`priority` values are `low`, `medium`, `high`, `urgent`** - there is no `normal`.

8. **`/api/v1/` is mandatory.** `GET /api/me` returns 404.

9. **Don't send CSRF headers on bearer-authed calls.**

10. **List endpoints return `[]` not `null` when empty.**

11. **`dueDate` on PATCH accepts RFC3339 datetimes** (and date strings parse). `createdAt`/`updatedAt` are always RFC3339.

12. **Patches are partial.** Send only changed fields. Use `null` on nullable fields to clear (`dueDate`, `assignedUserId`, `size`, `priority`).

13. **404 means "no access OR doesn't exist".** Per-board access: workspace membership alone doesn't guarantee a board. Check `myBoardRole` / `visibility` on the project.

14. **Attachments upload via `multipart/form-data`** with a single `file` part - not JSON. Requires Pro/Team workspace (`402` on Free). Check `canUploadAttachments` on the project first.

15. **Optimistic state is the SPA's job, not yours.** Call the API and trust the response.

16. **Don't loop single creates for imports.** `POST /projects/{id}/cards/bulk` does up to 100 cards in one transaction and one rate-limit token. Looping 100 `POST /cards` calls burns the whole rate budget and leaves a half-imported board if one fails mid-way; the bulk endpoint leaves either everything or nothing.

17. **Bulk create doesn't self-assign; single create does.** `POST /projects/{id}/cards` sets `assignedUserId` to the caller automatically; bulk items leave it `null` unless you send it.

18. **`POST /cards/bulk/move` ids must share a board unless `targetProjectId` is set.** Mixed-board ids without a target return 400. Cross-workspace targets return 404 (not 403) - same non-probing rule as everything else.

19. **`GET /projects/{id}/events` is the realtime SSE stream, not a board activity log.** It opens a long-lived `text/event-stream` that never completes, so a normal request/response client will hang on it. Per-card history is **`GET /cards/{id}/events`** - a plain JSON array, oldest-first. The two are unrelated despite the matching path segment.

20. **To read one card, use `GET /cards/{id}`.** Don't fetch `/projects/{id}/cards` and filter client-side - that list omits `description`. For the modal envelope (card + comments + events + checklist + attachments) use **`GET /cards/{id}/detail`**. `404` when it doesn't exist or you have no board access, same as everywhere.

21. **`tags` on PATCH replaces the whole set.** It isn't a merge: send the full array you want, or `[]` to clear. Max 12 tags per card, 32 chars each.

22. **Follow / unfollow return `204` with no body.** Both are idempotent, so don't read a response object or treat a repeat call as an error.

23. **`POST /projects/{id}/cards/rebalance` is not an import tool.** It renumbers cards already on the board onto one column as 1, 2, 3, … in request order, for when a drop can't find a representable midpoint. Every id must already be on that board (`404` otherwise). To create cards, use the bulk create endpoint.

24. **Comment bodies cap at 5000 characters.** Longer bodies are rejected, so split or truncate before sending.

25. **`POST /invitations` returns a wrapper, not an Invitation.** The body is `{"invitation": {...}, "consumed": bool}` - read `.invitation.id`, not `.id`. `consumed: true` means the email already belonged to a verified account, which was added to the workspace on the spot (no accept link needed). `GET /invitations` still returns bare Invitation objects.

26. **Archive is not delete, and restore puts the card back where it was.** `POST /cards/{id}/restore` returns it to its original column and position - don't follow it with a PATCH to "put it back". Archived cards vanish from `GET /projects/{id}/cards`; list them with `GET /projects/{id}/cards/archived`. To undo a column sweep, restore each id from the `POST /columns/{id}/archive` response.

27. **`POST /columns/{id}/archive` only works on a done column.** On a column without `isDone` it succeeds with `{"archived": 0, "ids": []}` rather than erroring - check the count. To archive arbitrary cards, archive them one by one.

28. **The webhook `{id}` is the subscription id.** `PATCH` / `DELETE /webhooks/{id}` take the `id` field from the list or create response, never `endpointId`.

29. **`PATCH /webhooks/{id}` needs Pro; `DELETE` doesn't.** On a downgraded (Free) workspace you can't pause a webhook with `{"active": false}` (402) - delete it instead. Same split for `/projects/{id}/inbound-email`.

30. **Webhook `secret` is returned on every read.** It isn't redacted after creation, so treat list / create / update responses as sensitive: don't print or log them whole. Only `craaft`-format deliveries are signed; a Slack / Discord receiver has nothing to verify.

31. **Templated boards don't have `todo` / `doing` / `done`.** Only the default `kanban` template seeds those keys. After `POST /projects` with a `template`, `GET /projects/{id}` and read `columns[].key` before creating cards. The create response carries no `columns` at all, and `/board-templates` doesn't expose keys. An unknown template key is `400 unknown board template`.

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

### Bulk import (one transaction, full metadata)

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"cards":[
           {"title":"Spec the flow","column":"todo","priority":"high","tags":["import"]},
           {"title":"Build it","column":"todo","size":5},
           {"title":"Ship it","column":"doing","dueDate":"2026-08-01T12:00:00Z"}
         ]}' \
     "$HOST/api/v1/projects/$PROJECT_ID/cards/bulk"
```

### Bulk update

```bash
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"cards":[
           {"id":"<card-1>","priority":"urgent"},
           {"id":"<card-2>","dueDate":null}
         ]}' \
     "$HOST/api/v1/cards/bulk"
```

### Sweep several cards to a column

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"ids":["<card-1>","<card-2>","<card-3>"],"column":"done"}' \
     "$HOST/api/v1/cards/bulk/move"
```

Add `"targetProjectId":"<other-project-uuid>"` to move the batch to
another board in the same workspace instead.

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

### Read one card

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/cards/$CARD_ID"
```

### Follow a card (204, no body)

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
     -X POST -H "Authorization: Bearer $TOKEN" \
     "$HOST/api/v1/cards/$CARD_ID/follow"
```

### List the tags in use on a board

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
     "$HOST/api/v1/projects/$PROJECT_ID/tags"
```

### Export a board as CSV

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
     -o board.csv \
     "$HOST/api/v1/projects/$PROJECT_ID/export?format=csv"
```

### Create a board from a template

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/board-templates" | jq -r '.[].key'

curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"name":"Q4 bugs","template":"bug-tracker"}' \
     "$HOST/api/v1/projects" | jq -r '.id' > /tmp/pid

# The create response has no columns; fetch the board for its column keys.
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/projects/$(cat /tmp/pid)" \
     | jq '.columns[] | {key, title}'
```

### Archive, list archived, restore

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/cards/$CARD_ID/archive"

curl -s -H "Authorization: Bearer $TOKEN" \
     "$HOST/api/v1/projects/$PROJECT_ID/cards/archived" | jq '.[] | {id, title, archivedAt}'

curl -s -X POST -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/cards/$CARD_ID/restore"
```

### Invite a teammate (response is a wrapper)

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"email":"sam@example.com","role":"member"}' \
     "$HOST/api/v1/invitations" | jq '{id: .invitation.id, consumed}'
```

### Add a Slack webhook for card moves and new cards

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"url":"https://hooks.slack.com/services/T000/B000/XXXX","format":"slack","description":"#eng board feed","events":["card.created","card.updated"]}' \
     "$HOST/api/v1/projects/$PROJECT_ID/webhooks" | jq '{id, format, events, active}'
```

Pause it later with `PATCH /webhooks/<id>` and `{"active":false}`.

### Verify a `craaft`-format delivery (receiver side)

```python
import hashlib, hmac

def verify(secret: str, raw_body: bytes, header: str) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    mac = hmac.new(secret.encode(), f"{parts['t']}.".encode() + raw_body, hashlib.sha256)
    return hmac.compare_digest(mac.hexdigest(), parts["v1"])
```

### Turn on email-to-card

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"targetColumn":"todo"}' \
     "$HOST/api/v1/projects/$PROJECT_ID/inbound-email" | jq -r '.email'
```

### Focus snapshot

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/cards/focus" | jq '
  {due_count: (.due | length),
   attention_count: (.attention | length),
   hygiene}
'
```

## Clients

There are three first-party SDKs and an MCP server. All of them speak the
same bearer-token API described above, so anything missing from a client
can still be reached with a raw request.

| Client | Install | Notes |
|---|---|---|
| Python | `pip install craaft` (1.3+) | Example below |
| JavaScript / TypeScript | `craaft` (1.0) | `src/resources/*.ts`, same resource layout |
| PHP | `composer require craaft/craaft` | Requires PHP 8.2+ and ext-curl |
| MCP server | remote HTTP endpoint | Exposes the API as tools for agents (`get_card`, `get_card_detail`, …) |

All three SDKs wrap the single-card read (`cards.get`) and the one-call card view `GET /cards/{id}/detail` (`cards.detail`: card + comments + events + checklist + attachments); the MCP server exposes them as `get_card` and `get_card_detail`. Avatar and public background-image bytes are `public.avatar` / `public.boardBackground` in all three SDKs.

Audited against the OpenAPI spec on 2026-09-28: the SDKs and the MCP server cover the whole token-accessible surface, including board templates, card archive / restore / archived list, webhooks and email-to-card, and they unwrap the `POST /invitations` `{invitation, consumed}` response. The only operation none of them wraps is the SSE stream (`GET /projects/{id}/events`), which is not a request/response call. MCP tool names for the newer surface: `list_board_templates` (plus a `template` argument on `create_project`), `archive_card`, `restore_card`, `list_archived_cards`, `list_webhooks` / `create_webhook` / `update_webhook` / `delete_webhook`, and `get_inbound_email` / `enable_inbound_email` / `update_inbound_email` / `disable_inbound_email`. If a client you have installed predates that audit, fall back to a raw request.

### Python

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

Raw `requests` works too - same bearer header, honour `Retry-After` on 429.

## When the spec drifts

1. Fetch live spec: `curl -s "$HOST/openapi.yaml"`.
2. If the spec doesn't match the response, read the Go handler in `internal/<resource>/handlers.go` in the main repo.
3. Tell the user the discrepancy; update this skill if the API changed intentionally.

## Don't

- Don't probe for resource existence via 404.
- Don't paginate manually - endpoints return capped lists or explicit limits.
- Don't refresh tokens - they don't expire; revocation is the only lifecycle event.
- Don't store tokens in plain text.
