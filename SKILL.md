---
name: craaft-api
description: Use when interacting with the Craaft Kanban JSON API - creating projects/columns/cards/comments, listing data across boards, drag-dropping cards via PATCH, or running any automation that authenticates with a `cra_*` bearer token. Triggers when the user mentions "Craaft API", "Craaft personal access token", a `cra_…` token, or a Craaft host (`craaft.io` or self-hosted). Does NOT apply to browser/SPA flows - those use cookie-based session auth and CSRF.
---

# Craaft API skill

## When to use

This skill applies when:
- The user asks to call the Craaft API directly (curl, fetch, requests, SDK).
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
| 402 | Plan limit (Free tier project cap = 3); body has `{limit, currentPlan, max, current}` | Surface upgrade path |
| 403 | Authenticated but not authorized; almost never via token | Stop, surface to user |
| 404 | Resource missing OR caller isn't a workspace member - intentionally indistinguishable | Don't probe; report |
| 409 | Duplicate (email/username already taken, column non-empty, etc.) | Adjust input |
| 429 | Rate limit; carries `Retry-After: <seconds>` | Sleep, retry |
| 5xx | Server bug | Retry once with backoff; report after 2 |

## Rate limits

Per-token bucket: **60 burst + 1 req/sec sustained.** In-memory per Cloud Run replica - the effective ceiling scales with horizontal replicas.

When automating: don't fan out > 30 parallel requests per token. For bulk imports, throttle to ~1/sec or batch. Honor `Retry-After` strictly; it's the next-token-available delay.

## Surface (versioned: `/api/v1/...`)

Authoritative spec is served at **`<HOST>/openapi.yaml`** (no auth required). Fetch the live URL when you need the current shape - the running binary serves it directly, so it's always exactly the version that handler is responding to:

```bash
curl -s "$HOST/openapi.yaml" > /tmp/craaft-openapi.yaml
```

Highlights:

| Method | Path | Purpose |
|---|---|---|
| GET    | `/me` | Current user (id, email, name, username, avatarUrl, hasPassword) |
| PATCH  | `/me` | Partial update of name / email / username |
| GET    | `/projects` | List your projects |
| POST   | `/projects` | Create (`name`, `description?`) |
| GET    | `/projects/{id}` | Single project + columns + counts |
| PATCH  | `/projects/{id}` | Partial update |
| DELETE | `/projects/{id}` | Cascading delete |
| POST   | `/projects/{id}/columns` | Add column (`title`) |
| PATCH  | `/columns/{id}` | Update (title/color/position/isDone/cardLimit) |
| DELETE | `/columns/{id}` | 409 if non-empty |
| GET    | `/projects/{id}/cards` | All cards in a project |
| POST   | `/projects/{id}/cards` | Create card |
| PATCH  | `/cards/{id}` | Update card (also drag-drop via column+position) |
| DELETE | `/cards/{id}` | Delete |
| GET    | `/cards/upcoming` | Cross-project due-dated cards |
| GET    | `/cards/focus` | Three-section Focus envelope: due / attention / hygiene |
| GET    | `/cards/hygiene?type=ghosts\|stuck\|mine_no_date` | Drill into a hygiene category |
| GET    | `/cards/{id}/comments` | List, oldest-first |
| POST   | `/cards/{id}/comments` | Add comment (`body`) |
| PATCH  | `/comments/{id}` | Edit (author only) |
| DELETE | `/comments/{id}` | Delete (author or workspace owner/admin) |
| GET    | `/search?q=…&limit=…` | Cross-project ILIKE search |
| GET    | `/members` | Workspace members |
| GET    | `/invitations` | Pending invitations |
| POST   | `/invitations` | Invite by email + role |

Endpoints intentionally NOT exposed via token auth: `/auth/*` (browser-bound), `/api-keys` (recursion), `/billing/*` (Paddle flows), `/me/avatar` upload.

## Pitfalls Claude commonly gets wrong

These are the bugs Claude reliably introduces. Re-read this list before generating Craaft API code.

1. **`position` is a `float64`, not an integer.** Cards (and columns) reorder by midpoint - to drop a card between two siblings at positions 2 and 3, send `position: 2.5`. To insert at the head with a head card at position 1, send `0.5`. Don't send integer indexes - they collide and silently land in the wrong slot.

2. **In card payloads, `column` is the column's `key`, NOT its `id`.** Keys (`"todo"`, `"doing"`, `"done"`, or per-project custom strings) are stable across moves; ids are internal UUIDs. The board GET response gives you `columns[].key` - use that.

3. **Drag-drop is a card PATCH, not a separate endpoint.** Send `{column, position}` to `PATCH /cards/{id}`. The same handler covers metadata edits (title/description/dueDate/...).

4. **`/api/v1/` is mandatory.** The legacy unversioned mount was retired. `GET /api/me` returns 404; `GET /api/v1/me` is correct.

5. **Don't send CSRF headers on bearer-authed calls.** The server's CSRF middleware exempts requests with `Authorization: Bearer …`. Sending an `X-CSRF-Token` is harmless but unnecessary.

6. **List endpoints return `[]` not `null` when empty.** Don't null-guard with `?? []` on top of code that does it; you'll add a redundant fallback.

7. **`dueDate` is `YYYY-MM-DD` (date), `createdAt`/`updatedAt` are RFC3339 timestamps.** Don't pass a full timestamp where a date is expected; the server will error with a parse hint.

8. **Patches are partial via `COALESCE`.** Send only the fields you want changed. Don't echo unmodified fields back - it works, but masks bugs (e.g. accidentally clearing a field by sending `null`).

9. **Many endpoints 404 instead of 403 for non-membership.** Authorization runs at the SQL layer through `workspace_members`. A card you can't see returns 404, indistinguishable from "doesn't exist". Don't infer existence from 404.

10. **Optimistic state is the SPA's job, not yours.** When automating, just call PATCH and trust the response. Don't try to mimic the SPA's optimistic + SSE-deduped flow.

## Common workflows

Each example assumes:
```bash
HOST=https://craaft.io
TOKEN=cra_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Use `$CRAAFT_TOKEN` from env in real code; do not paste tokens in scripts you save.

### Get the current user (smoke test)

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/me"
```

### List projects

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/projects" | jq '.[].name'
```

### Add a card to the first column of the first project

```bash
PROJECT_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  "$HOST/api/v1/projects" | jq -r '.[0].id')

# Pull columns + existing cards so we know the column key + a sane position.
PROJECT=$(curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/projects/$PROJECT_ID")
COLUMN_KEY=$(echo "$PROJECT" | jq -r '.columns[0].key')

# Position 1 puts the card at the head if no card already has position <= 1.
# For a robust insert at head, fetch cards in that column and pick min - 1.
curl -s -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d "$(jq -n --arg c "$COLUMN_KEY" --arg t "Triaged from API" \
          '{title:$t, column:$c, position:1}')" \
     "$HOST/api/v1/projects/$PROJECT_ID/cards"
```

### Move a card (drag-drop equivalent)

```bash
# Move card $CARD_ID to "doing" column, between cards at positions 4 and 5.
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"column":"doing","position":4.5}' \
     "$HOST/api/v1/cards/$CARD_ID"
```

### Set a due date + assignee

```bash
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"dueDate":"2026-06-15","assigneeId":"<user-uuid>","priority":"high"}' \
     "$HOST/api/v1/cards/$CARD_ID"
```

### Comment on a card

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"body":"Please review by Friday"}' \
     "$HOST/api/v1/cards/$CARD_ID/comments"
```

### Cross-project search

```bash
curl -sG -H "Authorization: Bearer $TOKEN" \
     --data-urlencode "q=migration" \
     --data-urlencode "limit=10" \
     "$HOST/api/v1/search" | jq '.cards[] | {title, projectName: .projectId}'
```

### "What should I focus on?" snapshot

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$HOST/api/v1/cards/focus" | jq '
  {due_count: (.due | length),
   attention_count: (.attention | length),
   hygiene}
'
```

## Python idiom

```python
import os, requests

BASE  = "https://craaft.io/api/v1"
TOKEN = os.environ["CRAAFT_TOKEN"]
S = requests.Session()
S.headers["Authorization"] = f"Bearer {TOKEN}"

def craaft(method, path, **kw):
    r = S.request(method, BASE + path, timeout=15, **kw)
    if r.status_code == 429:
        # Honor Retry-After strictly; sleeping a fraction less re-429s instantly.
        import time
        time.sleep(int(r.headers.get("Retry-After", "1")))
        return craaft(method, path, **kw)
    r.raise_for_status()
    return r.json() if r.content else None

projects = craaft("GET", "/projects")
project  = craaft("GET", f"/projects/{projects[0]['id']}")
column   = project["columns"][0]["key"]
craaft("POST", f"/projects/{projects[0]['id']}/cards",
       json={"title": "From Python", "column": column, "position": 1})
```

## When the spec drifts

This skill is an interpretation of the OpenAPI spec at the time of writing. If a request returns an unexpected field or 400 with a message about an unknown property:

1. Fetch the live spec: `curl -s "$HOST/openapi.yaml"`. The running binary serves it directly, so it's always exactly the version that handler is responding to.
2. If the spec doesn't match the response either, the Go handler in `internal/cards/handlers.go` (etc.) is the actual implementation.
3. Tell the user the discrepancy. Don't silently adapt - the docs may need an update.

## Don't

- Don't probe for resource existence (404 is intentionally ambiguous about membership vs nonexistence).
- Don't paginate manually - endpoints either return capped results or have explicit limits in the spec.
- Don't try to refresh tokens - tokens don't expire; revocation is the only lifecycle event. If 401, ask the user for a new token.
- Don't store tokens in plain text. Env vars or a secret manager only.
