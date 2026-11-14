# ERP ↔ Permission API Integration Guide (HD Document)

> Living document. Last updated: 2026-10-05, after phase 5. **Must be updated again after phase 7** (see section 9).
> Source of truth is the code in `apps/outline-permission-api/src`; this document describes the behavior actually built.

## 1. Big picture

ERP calls the **permission API** (Koa, internal port 4100) using a **service key** to:
1. Push ERP users into Outline (`/users/...`).
2. Create a project (= 1 Outline collection) and assign a role to the user (`/projects/...`).
3. Assign node/doc-level permissions (`/documents/{id}/members/...`).
4. **Create a doc node under the real user's name** (`POST /documents`, phase 5) → ERP receives the Outline link to hand to the user.

Users log in to Outline via SSO (bridge → IdP `idp.hdwebsoft.co`). ERP **does not sign any token/SSO** — it just hands over the plain URL from the response.

```
ERP ──service key──> permission API ──admin token──> Outline (user/group/collection)
                          │
                          └─ user's own OAuth token ──> Outline documents.create  (author = the real user)
```

Dev base URL (host): `http://localhost:4100`. Inside the compose network: `http://outline-permission-api:4100`. **No `/api/v1` prefix** — routes are at the root (the plan says `/api/v1/...` but the code doesn't use a prefix yet; if a prefix is added later it will be noted in section 9).

## 2. Authentication (service key)

- Header: `Authorization: Bearer <service key>`. Error: `401` (missing/wrong key).
- Each key has an allowed **scope** + **project-keys**. Missing scope → `403 SCOPE_FORBIDDEN`; wrong project → `403 PROJECT_KEY_FORBIDDEN`.

| Scope | Allows |
|---|---|
| `users:write` | `/users/*` |
| `permissions:write` | `/projects/*`, `/documents/:id/members/*` |
| `documents:create` | `POST /documents` (phase 5) |
| `tree:read` | `GET /projects/{projectKey}/document-tree` (phase 7) |
| `shared-tree:read` | `GET /shared-collection-tree`, `GET /shared-documents/:documentId/content` (phase 8) |
| `analytics:read` | `POST /documents/:id/view-stats`, `POST /shared-documents/:documentId/view-stats` |
| `analytics:notify` | `POST /documents/:documentId/view-reminders` (sends mail — kept separate from `analytics:read`) |
| `docs:sync` | `POST /sync-docs` (git repo docs sync) |
| `docs-sync-tokens:write` | `/collections/:collectionId/sync-tokens/*`, `/projects/:projectKey/sync-tokens/*` (token issuance & rotation) |

Key management (the key is only printed **once**):
```sh
pnpm --filter @hd-document/outline-permission-api manage-service-client create erp-main \
  --scopes users:write,permissions:write,documents:create --project-keys "*"
# rotate | revoke work the same way: manage-service-client rotate erp-main
```
This script connects to the DB from the host: requires `PERMISSION_API_DATABASE_URL` (see `infra/README.md`), e.g.
`postgres://permission_api_app:<PERMISSION_API_DB_PASSWORD>@localhost:55432/hd_document_apps`.

Common to all routes: errors are returned as `{ "error": { "code", "message" }, "requestId" }`; rate limiting applies (429 + `Retry-After`) and audit is logged per service client.

## 3. Endpoint list

The ERP user ID (`erpUserId`/`actingErpUserId`) **must be a UUID** = the IdP's `sub`.

### 3.1 Users — scope `users:write`
| Method | Path | Body | Result |
|---|---|---|---|
| PUT | `/users/:erpUserId` | `{ email, name }` | upsert user + invite into Outline |
| POST | `/users/batch-upsert` | `{ users: [{erpUserId,email,name}] }` (1–20) | `{ results }` |
| POST | `/users/:erpUserId/deactivate` | – | `{erpUserId,status:"deactivated"}`; **new (phase 5):** removes OAuth grant + revokes user token |
| POST | `/users/:erpUserId/activate` | – | `{erpUserId,status:"active"}` |

Errors: `409 EMAIL_ALREADY_IN_USE`, `403 SYSTEM_ADMIN_EMAIL_RESERVED` (the `SYSTEM_ADMIN_EMAIL` address is off-limits).

### 3.2 Projects — scope `permissions:write`
| Method | Path | Body | Result |
|---|---|---|---|
| PUT | `/projects/:projectKey` | `{ name }` | `{ projectKey, collectionId, url }` (idempotent) |
| GET | `/projects/:projectKey/members` | – | `{ projectKey, members:[{erpUserId,email,name,role}] }` — who currently has **collection-level** permission; `erpUserId: null` = someone outside ERP invited by email |
| PUT | `/projects/:projectKey/members/:erpUserId` | `{ role: viewer\|editor\|manager }` | `{projectKey,erpUserId,role}` |
| DELETE | `/projects/:projectKey/members/:erpUserId` | – | `204` (removed from collection; node-level permissions are kept) |
| PUT | `/projects/:projectKey/invitations` | `{ email, role: viewer\|editor\|manager }` | `{projectKey,email,role}` — any email; creates an Outline account if none exists, sends an email with the collection link |
| DELETE | `/projects/:projectKey/invitations?email=` | – | `204` (`404 USER_NOT_IN_OUTLINE` if the email has no account) |

`projectKey` follows the regex in `projects-routes.ts` (`INVALID_PROJECT_KEY` if invalid).

> **Collection permission ≠ node permission.** Project role (this section) = permission on the **entire collection**: the user sees EVERY doc in the project, regardless of node-level permission. Only assign it to people who need the whole project (managers). A user who only needs a few nodes → do **not** assign a project role, use section 3.3 instead. Audit for PUT/DELETE logs `erpUserId` + `role`.

### 3.3 Node-level permissions — scope `permissions:write`
| Method | Path | Body | Result |
|---|---|---|---|
| GET | `/documents/:documentId/members` | – | `{ documentId, members:[{erpUserId,email,name,permission}] }` (`erpUserId: null` = someone outside ERP) |
| PUT | `/documents/:documentId/members/:erpUserId` | `{ permission: read\|read_write\|admin }` | `{documentId,erpUserId,permission}` — `admin` = "Manage" (edit + share + delete that doc) |
| DELETE | `/documents/:documentId/members/:erpUserId` | – | `204` |
| PUT | `/documents/:documentId/invitations` | `{ email, permission }` | `{documentId,email,permission}` — any email, creates an Outline account if none exists, sends an email |
| DELETE | `/documents/:documentId/invitations?email=` | – | `204` |

### 3.4 Create node — scope `documents:create` (NEW, phase 5)

```
POST /documents
Authorization: Bearer <key>
Idempotency-Key: <1–200 printable ASCII characters>     ← required
Content-Type: application/json

{
  "projectKey": "PRJ-001",
  "actingErpUserId": "<uuid of the user who will be the author>",
  "title": "Meeting minutes",          // 1–255
  "text": "# Markdown ...",            // ≤ 200,000 characters
  "parentDocumentId": "<uuid>",        // optional, must belong to the projectKey's collection
  "publish": true                      // default true
}
```

| Status | Meaning | Body |
|---|---|---|
| `201` | Created, author = the real user | `{ documentId, url }` |
| `202` | User hasn't consented to OAuth yet (or the grant is broken/revoked) | `{ requestId, pendingUrl, expiresAt }` |
| `400` | Invalid body/header; `PARENT_DOCUMENT_WRONG_PROJECT`; `IDEMPOTENCY_KEY_REQUIRED` | |
| `403` | `USER_DEACTIVATED`, `ACTING_USER_FORBIDDEN` (user has no write permission), `PROJECT_KEY_FORBIDDEN` | |
| `404` | `USER_NOT_FOUND`, `PROJECT_NOT_FOUND`, `PARENT_DOCUMENT_NOT_FOUND` | |
| `409` | `IDEMPOTENCY_KEY_REUSED` (same key, different body) | |
| `410` | `PENDING_REQUEST_EXPIRED` (the pending request expired; use a new key) | |
| `429` / `502` | Outline/OAuth rate limit / upstream error → retry | |

**How ERP should handle it:**
1. Generate an `Idempotency-Key` (e.g. a UUID tied to the business record), call `POST /documents`.
2. `201` → store `documentId`+`url`; hand the `url` to the user (plain link, not wrapped).
3. `202` → store `requestId`; hand the user the **`pendingUrl`** to open. The user clicks "Accept" once in Outline → the doc is created automatically → the browser navigates into the doc.
4. To check the outcome after a 202: **call again with the exact same `Idempotency-Key` + the same body** → `202` (still pending) or `201 {documentId,url}` (done). No webhook.
5. Retry on timeout/5xx: reuse the **same key** — never creates a second doc.
6. Next time, once the user has already consented → always an immediate `201`.
7. First time when the user **has no Outline session yet**: actually verified (2026-10-05) — after the Outline SSO login step it returns to the **consent screen**, no need to reopen the link. Not yet tested with the real IdP when the user has no IdP session (expected to just add one extra IdP login screen).

### 3.5 Document tree — scope `tree:read` (NEW, phase 7)

```
GET /projects/{projectKey}/document-tree?actingErpUserId=<uuid>&parentDocumentId=<uuid>?&depth=<1-5>
Authorization: Bearer <key>
```

| Query | Type | Description |
|-------|------|-------------|
| `actingErpUserId` | UUID | Required; the ERP user whose permissions will be read |
| `parentDocumentId` | UUID | Optional; the parent document. If omitted → collection root |
| `depth` | number 1–5 | Optional, default 2; number of levels returned below the starting point |

| Status | Meaning | Body |
|---|---|---|
| `200` | Success | `{ projectKey, parentDocumentId, truncated, nodes:[{id,title,url,parentDocumentId,hasMoreChildren,children[]}] }` |
| `400` | Invalid query; `INVALID_PROJECT_KEY` | |
| `403` | `ACTING_USER_FORBIDDEN` (user has no collection permission AND is not shared on any node in the project) | |
| `404` | `USER_NOT_FOUND`, `PROJECT_NOT_FOUND`, `PARENT_DOCUMENT_NOT_FOUND` | |
| `409` | `OUTLINE_GRANT_REQUIRED` (user hasn't granted the `read` OAuth scope yet) | `{ error: {...}, grantUrl: "<pending-url>" }` |

**Notes:**
- Max 1000 nodes per response; if exceeded → `truncated: true`, call again with `parentDocumentId` = one of the child nodes to continue.
- Nodes carry no content; only `id`, `title`, `url`, `parentDocumentId`, `hasMoreChildren`, `children[]` (the child nodes if `depth > 1`).
- Permissions are checked on the Outline side. A user with a project role (collection permission) → gets the full tree (`collections.documents`). A user **without** collection permission but shared on a node (section 3.3) → gets a tree containing only those nodes + their descendants (Outline cascades permission down to children automatically; built from `userMemberships.list` + `documents.list`). Nothing at all → `403 ACTING_USER_FORBIDDEN`.
- If a collection is deleted directly in Outline: the `projectKey → collectionId` mapping still exists → every tree call returns `403`. There's currently no API to clean up the mapping; create the project with a new key or delete the row in `project_collection_map`.
- 409 Grant: ERP hands the user the `grantUrl` to open, the user clicks Accept → grants the `read` scope → retry the API.

**How ERP should handle it:**
1. Call `GET /projects/PRJ-001/document-tree?actingErpUserId=<uuid>&depth=2`.
2. `200` → process `nodes`, if `truncated=true` → call again with `parentDocumentId` = each child node to get details.
3. `409` → store `grantUrl`, hand the user the link to open (reuses the `/pending/:id` flow), then retry the API.

### 3.6 Shared collection tree & documents — scope `shared-tree:read` (NEW, phase 8)

#### 3.6.1 Shared collection tree

```
GET /shared-collection-tree?actingErpUserId=<uuid>&depth=<1-5>
Authorization: Bearer <key>
```

| Query | Type | Description |
|-------|------|-------------|
| `actingErpUserId` | UUID | Required; the ERP user to look up (must be active) |
| `depth` | number 1–5 | Optional, default 2; tree depth below each collection root |

| Status | Meaning | Body |
|---|---|---|
| `200` | Success | Array of `{ collectionId, collectionName, permission, truncated, nodes:[{id,title,url,parentDocumentId,hasMoreChildren,children[]}] }` (possibly `[]`) |
| `400` | Invalid query (bad UUID, depth out of range) | |
| `403` | `USER_DEACTIVATED` | |
| `404` | `USER_NOT_FOUND` | |

**Notes:**
- Uses the **admin token** (not the user's OAuth token); no grant flow, no `409`.
- Only returns collections that are **All-Members** (workspace-shared): `permission: "read"` or `"read_write"` in Outline. Private collections and group-restricted collections are excluded.
- Per-collection cap: 1000 nodes; if exceeded → `truncated: true` for that entry.
- The user's existence and active status are verified; beyond that, permissions are not checked (admin token sees everything).

**How ERP should handle it:**
1. `GET /shared-collection-tree?actingErpUserId=<uuid>&depth=2` → `200` array (may be `[]` if no workspace-shared collections exist).
2. For any entry with `truncated: true`, query `GET /projects/{projectKey}/document-tree?parentDocumentId=<nodeId>` (section 3.5) to page through deeper levels — note that section 3.5 requires `tree:read` scope and uses the user's OAuth token (may hit 409 grant flow).

#### 3.6.2 Shared document content

```
GET /shared-documents/:documentId/content?actingErpUserId={erpUserId}
Authorization: Bearer <key>
```

| Param | Type | Description |
|---|---|---|
| `documentId` | UUID | Required; ID of the document in Outline |
| `actingErpUserId` | UUID | Optional; ERP user UUID — records a view in Outline for that user |

| Status | Meaning | Body |
|---|---|---|
| `200` | Success | `{ documentId, title, text, url }` |
| `400` | Invalid UUID format (`DOCUMENT_ID_NOT_UUID`) | |
| `403` | Document is not in a workspace-shared collection (`DOCUMENT_NOT_IN_SHARED_COLLECTION`) | |
| `404` | Document not found in Outline (`DOCUMENT_NOT_FOUND`) | |

**Notes:**
- Uses the **admin token**; scope `shared-tree:read` required.
- Verifies that the document belongs to an **All-Members** (workspace-shared) collection (`permission: "read"` or `"read_write"`).
- If the document belongs to a private, restricted, or admin collection (or the collection cannot be accessed), returns `403 DOCUMENT_NOT_IN_SHARED_COLLECTION`.

#### Optional view tracking

Pass `actingErpUserId` (ERP user UUID) to record the view in Outline for that user.
The user must have previously granted OAuth consent (via the create-document flow).
If the grant is missing or expired, content is returned without tracking — no error.

### 3.7 Document View Stats — scope `analytics:read` (NEW, phase 9)

Returns who has viewed a document and which node members haven't yet.
Use this to show view status in your UI and identify users to send reminders to.

```
POST /documents/{documentId}/view-stats
Authorization: Bearer <key>
Content-Type: application/json

{
  "memberIds": ["uuid1", "uuid2"]   // optional; up to 500 ERP user UUIDs
}
```

- **Omit `memberIds`** (recommended): the API auto-fetches the document's node members from Outline and uses them as the member list. The "haven't viewed" result will always match who was actually invited to the node.
- **Pass `memberIds` explicitly**: overrides the auto-fetched list — useful when you want to check a custom subset (e.g. a project team).

**Response:**
```json
{
  "totalViews": 37,
  "uniqueViewers": 2,
  "viewers": [
    {
      "erpUserId": "...",
      "displayName": "Nguyễn Văn A",
      "viewCount": 8,
      "firstViewedAt": "2026-10-20T09:15:00Z",
      "lastViewedAt": "2026-10-29T16:42:00Z"
    }
  ],
  "unviewedMembers": ["uuid3", "uuid4"]
}
```

| Status | Meaning | Body |
|---|---|---|
| `200` | Success | see above |
| `400` | Invalid body / `memberIds` exceeds 500 | |
| `403` | `SCOPE_FORBIDDEN` | |
| `404` | `DOCUMENT_NOT_FOUND` | |

**Notes:**
- `totalViews`: sum of all view counts, including users outside the member list.
- `uniqueViewers`: count of members (from the effective list) who have viewed.
- `unviewedMembers`: member IDs with no view record.
- Only ERP users (with Outline grant) appear in `viewers`; direct Outline UI views are not counted.

### 3.7b Shared node view stats — scope `analytics:read`

Same purpose as 3.7, but for a node inside a **workspace-shared** collection. A shared node has
no invitation list — the whole workspace can read it — so the member set is **implicit**: every
active ERP user. A freshly created shared node therefore reports *all* of them as unviewed, and
`unviewedMembers` shrinks each time a view is recorded.

```
POST /shared-documents/{documentId}/view-stats
Authorization: Bearer <key>
Content-Type: application/json

{
  "excludeErpUserIds": ["uuid1", "uuid2"]   // optional; up to 500 ERP user UUIDs
}
```

The body may be omitted entirely (`{}` or no body at all) — the member set is derived from
the DB, not supplied by the caller. There is no `memberIds` override: unlike 3.7 you cannot
*define* the member list, only **narrow** it.

- **`excludeErpUserIds`** (optional): drops these users from the member set completely — they
  appear in neither `viewers` nor `unviewedMembers` and are not counted in `totalMembers`.
  Use it for people a given node does not concern. IDs that aren't members are ignored.
- Always excluded, without being asked for: deactivated users and the reserved
  `SYSTEM_ADMIN_EMAIL` account.

**Response:**
```json
{
  "documentId": "...",
  "totalMembers": 42,
  "totalViews": 37,
  "uniqueViewers": 2,
  "viewers": [
    {
      "erpUserId": "...",
      "displayName": "Nguyễn Văn A",
      "viewCount": 8,
      "firstViewedAt": "2026-10-20T09:15:00Z",
      "lastViewedAt": "2026-10-29T16:42:00Z"
    }
  ],
  "unviewedMembers": [
    { "erpUserId": "...", "name": "Trần Thị B", "email": "b@company.com" }
  ]
}
```

| Status | Meaning | Body |
|---|---|---|
| `200` | Success | see above |
| `400` | `DOCUMENT_ID_NOT_UUID`, `INVALID_JSON`, `excludeErpUserIds` exceeds 500 | |
| `403` | `SCOPE_FORBIDDEN`, `DOCUMENT_NOT_IN_SHARED_COLLECTION` | |
| `404` | `DOCUMENT_NOT_FOUND` | |

**Notes:**
- `totalMembers`: size of the implicit member set (active ERP users, minus the reserved account, minus `excludeErpUserIds`).
- `totalViews`: sum of every view count on the node — including viewers with no ERP mapping **and excluded users**, so it is unaffected by `excludeErpUserIds`.
- `uniqueViewers` = `viewers.length`: members with at least one view. `totalMembers - uniqueViewers` = `unviewedMembers.length`.
- Members not yet synced into Outline (no `outline_user_id`) stay in `unviewedMembers`.
- Views are recorded by `GET /shared-documents/:documentId/content` when `actingErpUserId` is passed (see 3.6) — without it nothing is tracked.
- The node must live in a workspace-shared collection; otherwise `403 DOCUMENT_NOT_IN_SHARED_COLLECTION` (same guard as 3.6).

### 3.7c Document view reminders — scope `analytics:notify`

Sends a gentle "you haven't opened this document yet" email (document title + link) to the
addresses you supply. A **reminder, not an invitation**: nothing is granted, nothing in Outline
is mutated. Because of that it performs **no collection check** and therefore works the same for
a node in a **private** collection and for one in a **workspace-shared** collection.

```
POST /documents/{documentId}/view-reminders
Authorization: Bearer <key>
Content-Type: application/json

{
  "emails": ["b@company.com", "c@company.com"]   // required, 1–200 valid emails
}
```

**Usual flow:** call 3.7 (`/documents/:id/view-stats`) or 3.7b (`/shared-documents/:id/view-stats`)
first → take the `email` of each entry in `unviewedMembers` → post them here. The API does not
re-check who has viewed: it mails exactly who you list, so compute the list right before sending.

**Response (200):**
```json
{
  "documentId": "...",
  "documentTitle": "Q4 safety policy",
  "documentUrl": "https://docs.company.com/doc/q4-safety-policy-abc",
  "reminded": ["b@company.com"],
  "skipped": ["B@company.com"]
}
```

| Status | Meaning | Body |
|---|---|---|
| `200` | Mails handed to SMTP | see above |
| `400` | `DOCUMENT_ID_NOT_UUID`, `INVALID_JSON`, empty list, >200 emails, invalid email | |
| `403` | `SCOPE_FORBIDDEN` (key lacks `analytics:notify`) | |
| `404` | `DOCUMENT_NOT_FOUND` | |

**Notes:**
- `skipped` = addresses intentionally not mailed: case-insensitive duplicates and the reserved
  `SYSTEM_ADMIN_EMAIL`. `reminded.length + skipped.length` always equals the number of emails sent in.
- Any address is accepted (same as the invite endpoints) — it does **not** have to be an ERP user.
  The mail only exposes the title + link; whoever cannot read the doc in Outline still cannot open it.
- SMTP failures are logged server-side and do **not** fail the request (fire-and-forget, same as
  invite mails). With `SMTP_HOST`/`SMTP_PORT` unset, nothing is sent and the call still returns 200.
- No reminder history / throttling: calling twice sends twice. ERP decides the cadence.
- The service key must be issued with the new scope:
  `manage-service-client create erp-main --scopes analytics:read,analytics:notify --project-keys "*"`
  (or `rotate` an existing key with the scope added).

### 3.9 Docs sync (Git repository -> Outline) — scope `docs:sync`

Synchronizes Markdown documentation and image assets from consumer Git repositories directly into Outline collections. All parsing, validation, image uploads, Outline diffing, and DB mapping run server-side.

- **Method / Path:** `POST /sync-docs`
- **Scope required:** `docs:sync`
- **Headers:**
  - `Authorization: Bearer <sync_token>`
  - `Content-Type: application/gzip` (or `application/x-gzip`)
- **Query parameters:**
  - `dryRun` (`1` or `true`, optional): Runs full extraction, validation, markdown/image parsing, and Outline diff planning without acquiring locks or making any mutations in Outline or DB.

#### Limits & Safety (enforced server-side)
Referenced from `apps/outline-permission-api/src/docs-sync/docs-sync-limits.ts`:
- **Max archive request body size:** 50 MB (`MAX_ARCHIVE_REQUEST_BYTES`).
- **Max archive entries:** 5,000 files (`MAX_ARCHIVE_ENTRIES`).
- **Max single file uncompressed size:** 20 MB (`MAX_FILE_BYTES`).
- **Max total uncompressed size:** 200 MB (`MAX_UNCOMPRESSED_BYTES` — zip-bomb guard).
- **Allowed file types:** `.erp-docs.config.json` (at archive root), `.md`, and image formats (`.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp`).
- **Path safety:** POSIX paths only; backslashes (`\`), null bytes (`\0`), absolute paths (`/foo` or `C:`), and directory traversal (`..`) are strictly rejected. Symlinks, hardlinks, FIFOs, and devices are rejected (`UNSUPPORTED_ARCHIVE_ENTRY`).

#### Configuration file (`.erp-docs.config.json`)
Must be present at the archive root:
```json
{
  "collectionId": "11111111-1111-1111-1111-111111111111",
  "root": "docs/",
  "include": ["docs/**"],
  "exclude": ["docs/internal/**"],
  "apiUrl": "${PERMISSION_API_PUBLIC_URL}",
  "token": "${ERP_DOCS_SYNC_TOKEN}",
  "envFile": ".env.erp-docs"
}
```
> **Security Rule:** `apiUrl` and `token` must use `${VAR_NAME}` placeholders. Hardcoded secrets in this file are rejected with `INVALID_SYNC_CONFIG`.

#### Processing & Synchronization Flow
1. **Extraction:** Stream gzipped tar in memory under strict entry and byte caps.
2. **Access resolution:** Verifies that caller token has access to the project mapped to `collectionId` (throws `403 COLLECTION_FORBIDDEN` on mismatch or non-existent collection without enumeration leak).
3. **Glob filtering:** Matches markdown paths against `include` / `exclude` patterns via `picomatch`.
4. **Tree & Titles:** Builds hierarchical node tree. First `# H1 Heading` in markdown becomes document title (falls back to filename). Directory nodes use `README.md` or `index.md` as content.
5. **Image deduplication:** Image references (`![alt](path)`) are extracted and uploaded to Outline as attachments (`attachments.create`), deduplicated by SHA-256 in `permission_api.doc_sync_attachments`. Markdown image links are rewritten to Outline URLs.
6. **Advisory lock:** Non-dry-run sync acquires `pg_try_advisory_lock` on `collectionId`. If locked, returns `409 SYNC_IN_PROGRESS`.
7. **Action planning & apply:** Compares desired state against live Outline state (`documents.list`) and DB mapping (`permission_api.doc_sync_documents`). Performs `create`, `update`, `move`, `archive`, `restore`, or `skip`. Removed Git files are archived in Outline and mapping rows deleted.

#### Response (`200 OK`)
```json
{
  "ok": true,
  "dryRun": false,
  "collectionId": "11111111-1111-1111-1111-111111111111",
  "created": 5,
  "updated": 2,
  "moved": 1,
  "restored": 0,
  "skipped": 10,
  "archived": 1,
  "failed": [],
  "warnings": []
}
```

If document operations fail, `"ok": false` with details:
```json
{
  "ok": false,
  "dryRun": false,
  "collectionId": "11111111-1111-1111-1111-111111111111",
  "created": 4,
  "updated": 2,
  "moved": 1,
  "restored": 0,
  "skipped": 10,
  "archived": 1,
  "failed": [
    {
      "repoPath": "docs/architecture/spec.md",
      "action": "create",
      "reason": "Outline API network error: timeout"
    }
  ],
  "warnings": [
    "Image 'assets/missing.png' referenced in 'docs/guide.md' was not found in archive; link left intact."
  ]
}
```

#### Error codes
| Status | Code | Meaning / Trigger |
|---|---|---|
| 400 | `EMPTY_ARCHIVE` | Request body is empty. |
| 400 | `UNSAFE_ARCHIVE_PATH` | Path contains null byte, backslash, absolute path, or `..` traversal. |
| 400 | `UNSUPPORTED_ARCHIVE_ENTRY` | Symlink, FIFO, socket, or unsupported tar entry. |
| 400 | `UNSUPPORTED_ARCHIVE_FILE` | Disallowed file extension. |
| 400 | `INVALID_ARCHIVE` | Corrupt tar/gzip stream. |
| 400 | `MISSING_SYNC_CONFIG` | `.erp-docs.config.json` not found in archive root. |
| 400 | `INVALID_SYNC_CONFIG` | Config JSON invalid, invalid collection UUID, or contains literal secrets. |
| 401 | `UNAUTHORIZED` | Missing or invalid Bearer token. |
| 403 | `SCOPE_FORBIDDEN` | Token lacks `docs:sync` scope. |
| 403 | `COLLECTION_FORBIDDEN` | Token not authorized for collection, or collection not found. |
| 409 | `SYNC_IN_PROGRESS` | Another sync is running on this collection (advisory lock held). |
| 413 | `PAYLOAD_TOO_LARGE` | Raw gzipped body exceeds 50 MB. |
| 413 | `ARCHIVE_TOO_MANY_ENTRIES` | Archive contains > 5,000 files. |
| 413 | `ARCHIVE_FILE_TOO_LARGE` | Single file uncompressed size exceeds 20 MB. |
| 413 | `ARCHIVE_UNCOMPRESSED_TOO_LARGE` | Total uncompressed archive size exceeds 200 MB. |
| 415 | `UNSUPPORTED_MEDIA_TYPE` | Content-Type is not `application/gzip` or `application/x-gzip`. |

---

### 3.10 Sync token management & Planner contract — scope `docs-sync-tokens:write`

Provides token lifecycle management for Git repository documentation synchronization. Designed for HD ERP Project Planner / Admin Portal to provision, rotate, and revoke sync tokens.

- **Scope required:** `docs-sync-tokens:write`
- **Dual routing:** Every endpoint supports BOTH:
  - Collection-centric path: `/collections/:collectionId/sync-tokens[/*]` (where `:collectionId` is a UUID)
  - Project-centric alias: `/projects/:projectKey/sync-tokens[/*]` (where `:projectKey` matches project regex e.g. `PRJ-001`)

#### 1. List active sync tokens
- **Path:** `GET /collections/:collectionId/sync-tokens` or `GET /projects/:projectKey/sync-tokens`
- **Response (`200 OK`):**
  ```json
  {
    "collectionId": "11111111-1111-1111-1111-111111111111",
    "projectKey": "PRJ-001",
    "maxAllowed": 1,
    "items": [
      {
        "id": "a1b2c3d4e5f60718",
        "name": "docs-sync:PRJ-001:a1b2c3d4e5f60718",
        "createdAt": "2026-11-10T08:00:00.000Z",
        "lastUsedAt": "2026-11-10T08:30:00.000Z"
      }
    ]
  }
  ```
  *(Note: `lastUsedAt` is queried efficiently from `permission_api.api_audit_log`; returns `null` if the token has never synced).*

#### 2. Create sync token
- **Path:** `POST /collections/:collectionId/sync-tokens` or `POST /projects/:projectKey/sync-tokens`
- **Response (`201 Created`, `Cache-Control: no-store`):**
  ```json
  {
    "id": "a1b2c3d4e5f60718",
    "token": "hdk_a1b2c3d4e5f60718_e8f23a54b9d047c6a1e389b21f07",
    "collectionId": "11111111-1111-1111-1111-111111111111",
    "projectKey": "PRJ-001",
    "createdAt": "2026-11-10T08:00:00.000Z"
  }
  ```
- **Error:** `409 SYNC_TOKEN_LIMIT_REACHED` if active token count equals or exceeds `maxAllowed` (default 1, configurable via `DOCS_SYNC_MAX_ACTIVE_TOKENS` environment variable). Use rotate or revoke first.

#### 3. Rotate sync token (Atomic replacement)
Replaces existing token(s) and creates a new one in a single atomic database operation. Prevents deadlock when onboarding new engineers or setting up new CI secrets.
- **Path:** `POST /collections/:collectionId/sync-tokens/rotate` or `POST /projects/:projectKey/sync-tokens/rotate`
- **Body (optional JSON):**
  ```json
  {
    "previousTokenId": "a1b2c3d4e5f60718"
  }
  ```
  *(If `previousTokenId` is omitted, all currently active sync tokens for this project are revoked while issuing the new one).*
- **Response (`200 OK`, `Cache-Control: no-store`):**
  ```json
  {
    "id": "b2c3d4e5f6071829",
    "token": "hdk_b2c3d4e5f6071829_9c8b7a6d5e4f3021a8b7c6d5e4f3",
    "collectionId": "11111111-1111-1111-1111-111111111111",
    "projectKey": "PRJ-001",
    "createdAt": "2026-11-10T08:35:00.000Z",
    "revokedCount": 1
  }
  ```

#### 4. Revoke single sync token
- **Path:** `DELETE /collections/:collectionId/sync-tokens/:id` or `DELETE /projects/:projectKey/sync-tokens/:id`
- **Parameter `:id`:** 16-character hex string token ID.
- **Response:** `204 No Content`
- **Error:** `404 SYNC_TOKEN_NOT_FOUND` if token does not exist or is already revoked.

#### 5. Revoke all sync tokens (Emergency revoke)
- **Path:** `DELETE /collections/:collectionId/sync-tokens?all=true` or `DELETE /projects/:projectKey/sync-tokens?all=true`
- **Query param:** `?all=true` is required (safety guard against accidental deletion without `:id`).
- **Response:** `204 No Content`
- **Error:** `400 MISSING_TOKEN_ID_OR_ALL` if `?all=true` is missing.

#### Planner UI Integration Contract (Hợp đồng giao diện cho team Planner)
For developers integrating the sync token management into the **HD ERP Project Planner** UI:

1. **Route Flexibility:** Planner can call either `/collections/:collectionId/sync-tokens` or `/projects/:projectKey/sync-tokens` interchangeably based on current view model context.
2. **Token List Display:**
   - Display active token list with `id`, `name`, `createdAt`, and `lastUsedAt`.
   - If `lastUsedAt` is `null`, display "Chưa sử dụng (Never used)".
3. **Button "Tạo sync token" (Create Sync Token):**
   - Enabled only when active token count < `maxAllowed` (default 1).
   - If limit reached (1/1), disable button or show hint: "Đã đạt giới hạn 1 token. Vui lòng sử dụng Đổi token (Rotate) hoặc Thu hồi (Revoke)".
4. **Button "Đổi token (Rotate)":**
   - Atomically revokes existing active token and immediately generates a new token.
   - Recommended action when onboarding new team members or updating CI secrets, preventing lockout.
5. **Button "Thu hồi (Revoke)":**
   - In-row button: `DELETE .../sync-tokens/:id` with confirmation modal.
   - Top-level emergency button: "Thu hồi tất cả (Revoke All)" calling `DELETE .../sync-tokens?all=true`.
6. **Secret Reveal Modal (Hiển thị 1 lần duy nhất):**
   - When creating or rotating, the response contains `token: "hdk_..."`.
   - Show modal dialog with:
     - Plaintext token value in a masked / toggle-visible input.
     - "Copy" button.
     - Critical warning alert: *"Token này chỉ hiển thị một lần duy nhất. Hãy lưu vào kho bí mật CI hoặc file .env.erp-docs ngay. Hệ thống sẽ không hiển thị lại token này."*
     - **Planner backend requirement:** Planner database MUST NOT store plaintext sync tokens.
7. **Developer Onboarding Assistance:**
   - Display `collectionId` (UUID) for easy reference.
   - Provide prefilled `.erp-docs.config.json` snippet ready to copy into repo root.
   - Provide prefilled `.env.erp-docs` template snippet.
   - Provide download button or direct link to `run.sh` (see consumer onboarding guide: [scripts/docs-sync/README.md](../scripts/docs-sync/README.md)).

---

### 3.11 Browser routes (not meant to be called by ERP)
- `GET /pending/:id` → redirects to `{OUTLINE_URL}/oauth/authorize` (state + PKCE S256, httpOnly cookie) or directly to the doc if already done. Supports both the doc-creation flow (phase 5) and the read-grant flow (phase 7).
- `GET /oauth/outline/callback?code&state` → exchanges the token, verifies identity (`auth.info` must match the user), stores the grant, creates the doc (if there's a pending request), redirects to the doc or a confirmation page. Errors are shown as a single line of text + an `ERP_PORTAL_URL` link.
- `GET /healthz` — no key required.

## 4. Configuration (infra/.env)

| Variable | Meaning |
|---|---|
| `OUTLINE_OAUTH_CLIENT_ID/SECRET` | Created with `pnpm --filter @hd-document/outline-workspace-setup register-oauth-client` |
| `PERMISSION_API_PUBLIC_URL` | The **user's browser**-reachable URL for the service; the redirect URI = `<URL>/oauth/outline/callback` must match the OAuth client |
| `TOKEN_SEAL_PASSWORD` | ≥ 32 characters; seals tokens. Changing it = users must consent again |
| `OUTLINE_OAUTH_SCOPE` | default `documents:create auth:read read` (the `read` scope is auto-added if missing, for the phase 7 document-tree API) |
| `PENDING_REQUEST_TTL_DAYS` | default 7 |
| `DOCS_SYNC_MAX_ACTIVE_TOKENS` | default 1; maximum number of active docs-sync tokens allowed per project |
| `ERP_PORTAL_URL` | the "back to ERP" link shown on the error page |

Missing any one of `PERMISSION_API_PUBLIC_URL`, `OUTLINE_OAUTH_CLIENT_ID/SECRET`, `TOKEN_SEAL_PASSWORD` → the phase 5 route group is **disabled** (`POST /documents` 404).

> Note on your current `.env` state: `PERMISSION_API_PUBLIC_URL=http://localhost:4002` but the service is published on host port `PERMISSION_API_HOST_PORT=4100`. Without a proxy/tunnel from 4002→4100, `pendingUrl` won't open and the OAuth redirect URI will be wrong. Fix this mismatch (and update the OAuth client's redirect URI) before testing the 202 flow.

## 5. Build & migrate (dev)

```powershell
# migration 0004 (grant/pending/idempotency tables) — owner role
$env:APP_DATABASE_URL = "postgres://hd_document_apps:<APP_DB_PASSWORD>@localhost:55432/hd_document_apps"
pnpm --filter @hd-document/app-database migrate
# rebuild
docker compose -f infra/docker-compose.yml --env-file infra/.env up -d --build outline-permission-api oidc-bridge
curl.exe http://localhost:4100/healthz
```
Ran on 2026-10-05: migration 0004 applied successfully, 5 containers healthy, `/pending/<unknown id>` → 404 (phase 5 route is active).

## 6. How to test the APIs

One-time setup (PowerShell; get the key from the `manage-service-client create` command):
```powershell
$B = "http://localhost:4100"; $H = @{ Authorization = "Bearer $KEY" }
$U = "<uuid = sub of a real IdP user>"
```

1. **User:** `curl.exe -X PUT $B/users/$U -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" -d '{\"email\":\"a@x.com\",\"name\":\"A\"}'`
2. **Project:** `PUT /projects/PRJ-001` body `{"name":"Project 1"}` → note the `collectionId`.
3. **Role:** `PUT /projects/PRJ-001/members/$U` body `{"role":"editor"}` (needs `editor` or above to create docs).
4. **Create node for the first time (expect 202):**
   `POST /documents` + `Idempotency-Key: t1` + body from section 3.4 → `202 {pendingUrl}`.
5. Open `pendingUrl` in a browser, log in as that user → click **Accept** → land in the new doc. Verify the author in Outline = that user.
6. **Ask for status again:** POST the exact same request again (`Idempotency-Key: t1`) → `201` with the same `documentId`.
7. **Create a second time with a new key (`t2`):** → immediate `201` (grant already exists).
8. **Idempotency:** same `t1` with a different `title` → `409 IDEMPOTENCY_KEY_REUSED`.
9. **Permission:** downgrade the user to `viewer` → `403 ACTING_USER_FORBIDDEN`.
10. **Deactivate:** `POST /users/$U/deactivate` then create a node → `403 USER_DEACTIVATED`; `activate` then create a node → `202` (grant was removed, must consent again).
11. **Document tree (phase 7):**
    - Create a new key with scope `tree:read`: `pnpm --filter @hd-document/outline-permission-api manage-service-client create erp-tree --scopes tree:read --project-keys "*"`.
    - `GET /projects/PRJ-001/document-tree?actingErpUserId=$U&depth=2` (scope `tree:read` required) → `200 {projectKey,truncated,nodes}` if the user has already granted the `read` OAuth scope.
    - First-time user (hasn't granted `read` yet) → `409 {grantUrl}` → open grantUrl → "Accept" → browser redirects to the Outline homepage → retry → `200`.
    - Downgrade the user to `viewer` → `403 ACTING_USER_FORBIDDEN`.
    - Invalid query (wrong UUID, depth > 5) → `400`.

12. **Shared collection tree & documents (phase 8):**
    - Create a key with scope `shared-tree:read`: `pnpm --filter @hd-document/outline-permission-api manage-service-client create erp-shared-tree --scopes shared-tree:read --project-keys "*"`.
    - `GET /shared-collection-tree?actingErpUserId=$U&depth=2` → `200` array (possibly `[]`).
    - Unknown user UUID → `404`; invalid UUID → `400`; use `tree:read`-only key → `403 SCOPE_FORBIDDEN`.
    - `GET /shared-documents/:documentId/content` → `200 {documentId,title,text,url}` if in a shared collection; private/admin/inaccessible doc → `403 DOCUMENT_NOT_IN_SHARED_COLLECTION`; non-existent doc → `404 DOCUMENT_NOT_FOUND`.

13. **Git repo docs sync & sync tokens (docs-sync):**
    - Create an admin service key with scope `docs-sync-tokens:write`:
      `pnpm --filter @hd-document/outline-permission-api manage-service-client create erp-sync-mgr --scopes docs-sync-tokens:write --project-keys "*"`
    - List tokens: `curl.exe -H "Authorization: Bearer $KEY" $B/projects/PRJ-001/sync-tokens` → `200 { collectionId, projectKey, items, maxAllowed }`.
    - Create sync token: `curl.exe -X POST -H "Authorization: Bearer $KEY" $B/projects/PRJ-001/sync-tokens` → `201 { id, token: "hdk_...", collectionId }`. (Note: save token as `$SYNC_TOKEN`).
    - Rotate token: `curl.exe -X POST -H "Authorization: Bearer $KEY" $B/projects/PRJ-001/sync-tokens/rotate` → `200 { id, token, revokedCount }`.
    - Revoke single token: `curl.exe -X DELETE -H "Authorization: Bearer $KEY" $B/projects/PRJ-001/sync-tokens/$TOKEN_ID` → `204`.
    - Emergency revoke all: `curl.exe -X DELETE -H "Authorization: Bearer $KEY" "$B/projects/PRJ-001/sync-tokens?all=true"` → `204`.
    - Run dry-run sync with client script:
      `sh scripts/docs-sync/run.sh --dry-run`
      or manually via curl:
      `tar -czf - .erp-docs.config.json docs/ | curl.exe -X POST "$B/sync-docs?dryRun=1" -H "Authorization: Bearer $SYNC_TOKEN" -H "Content-Type: application/gzip" --data-binary @-`

Automated tests: `pnpm -r typecheck`, `pnpm --filter @hd-document/outline-permission-api test` (includes unit and integration tests for docs sync and sync tokens).

## 7. Data model (schema `permission_api`)

Core schema objects:
- `erp_users`: User identity map from ERP (`erp_user_id` UUID, email, name, status).
- `project_collection_map`: Map between ERP `project_key` and Outline `collection_id`.
- `service_clients`: API keys with hashed secrets, allowed scopes, and project restrictions.
- `api_audit_log`: Detailed audit logs of all API calls.
- `user_outline_grants`: Sealed OAuth user tokens (phase 5).
- `pending_document_requests`: Transient state for user consent flow (phase 5).
- `idempotency_keys`: Idempotent mutation tracking (phase 5).

Docs sync tables (migration 0007):
- `doc_sync_documents`:
  - `(collection_id, repo_path)` (PRIMARY KEY): Maps repository file/folder path to Outline document.
  - `document_id`: Outline document UUID (`UNIQUE`).
  - `content_hash`: SHA-256 hash of (title + body + parentId) used to detect content changes.
  - `is_folder`: Boolean indicating whether the node is a directory/folder.
  - `outline_updated_at`: Last known `updatedAt` from Outline.
  - `updated_at`: Timestamp of last sync mapping update.
- `doc_sync_attachments`:
  - `(collection_id, sha256)` (PRIMARY KEY): Cache for image attachments uploaded to Outline.
  - `attachment_url`: CDN/Outline URL returned by `attachments.create`.
  - `created_at`: Upload timestamp.
- Index `api_audit_log_service_client_occurred_at_idx`:
  - On `permission_api.api_audit_log (service_client_id, occurred_at DESC)` for efficient retrieval of sync token `lastUsedAt`.

> **Consumer Onboarding Guide:** Complete setup guide for Git repository maintainers: [scripts/docs-sync/README.md](../scripts/docs-sync/README.md).

## 8. Known status & limitations (updated 2026-11-10)

**Verified against real Outline 1.10.1** (report: `plans/reports/fullstack-developer-261005-GH-5-phase-05-real-outline-verification.md`):
- authorize → consent → token → `documents.create` using the user's token → author is correctly that user. Options B/C from the plan are **not needed**.
- Scope `documents:create auth:read` works (`create read` also works). `auth:read_write` is **invalid** (the Authorize button hangs).
- Access token lives 3600s; refresh token lives 30 days, **sliding** on each refresh (the user only needs to consent again after 30 days of non-use); each refresh rotates the token, reusing the old token → 400.
- `OUTLINE_OAUTH_CLIENT_ID` must be the **public `clientId`** (not the UUID `id`); the `register-oauth-client` CLI was fixed to print the right one.
- Tests: Playwright `tests/e2e/create-node-pending-consent-flow.spec.ts` + `create-node-real-outline-integration.spec.ts` (8 tests pass). Run with: `cd tests/e2e; npx playwright test create-node` (requires `E2E_PERMISSION_API_SERVICE_KEY` in `tests/e2e/.env`).

**Done in phase 5:** audit for `POST /documents` logs `actingErpUserId`/`projectKey`/`documentId`/`resultStatus`; the 2 browser routes are rate limited to 30 req/min/IP (needs `PERMISSION_API_TRUST_PROXY=true` behind a reverse proxy) + logging; a cleanup job removes expired pending/idempotency rows (hourly, migration 0005); `parentDocumentId` is validated before returning 202; 4xx validation errors delete the idempotency row so retrying with the same key after fixing the body doesn't hit a 409; network errors on `auth.info` → `503 OUTLINE_UNAVAILABLE` (token is not revoked, the user reopens pendingUrl).

**Done in phase 8:** shared collection tree API `GET /shared-collection-tree` and shared document content `GET /shared-documents/:documentId/content` + scope `shared-tree:read`; uses admin token (no OAuth grant needed); returns only All-Members collections; per-collection 1000-node cap with `truncated` flag; depth 1–5 default 2; user existence + active status validated; shared document content guarded by workspace collection permission check; audit logs `actingErpUserId`.

**Done in phase 7:** document-tree API `GET /projects/{projectKey}/document-tree` + scope `tree:read` (service key); uses the user's OAuth token so Outline enforces permissions automatically; user hasn't granted the `read` scope yet → `409 OUTLINE_GRANT_REQUIRED` + grantUrl (reuses the `/pending/:id` consent flow, migration 0006); max 1000 nodes/response, `truncated` flag if exceeded; each node only carries `id`/`title`/`url`/`parentDocumentId`/`hasMoreChildren`/`children` (no content); depth 1–5, default 2; audit logs `actingErpUserId`/`projectKey`/`resultStatus`.

**Done in docs-sync (server-side, 2026-11-10):**
- Replaced reverted client-side sync (PR #7) with server-side sync service (`POST /sync-docs`, scope `docs:sync`).
- Safe streaming tar extraction in memory (`tar-stream`), path traversal protection, zip-bomb size guards (50MB request, 200MB uncompressed, 5,000 files, 20MB/file max).
- Outline diffing engine plans create/update/move/archive/restore/skip actions against live Outline state and `permission_api.doc_sync_documents`.
- Image attachment deduplication in `permission_api.doc_sync_attachments` (SHA-256) and markdown URL rewrite.
- PostgreSQL session advisory locking (`pg_try_advisory_lock`) per collection to prevent concurrent sync races.
- Sync token management endpoints under `docs-sync-tokens:write` with dual routing (`/collections/:id/...` and `/projects/:key/...`) supporting list, create, rotate (atomic replacement), and revoke (single or all).
- Lightweight consumer shell script `scripts/docs-sync/run.sh` (POSIX git/tar/curl, zero node dependencies).

**Remaining / notes:**
- Concurrent requests with the same `Idempotency-Key`: fixed, both return `201` for the same doc (confirmed by e2e test). ERP should still retry on `5xx`.
- Replaying a `202` still returns `202` if the user already has a grant but the previous creation attempt failed → use a new key.
- `/oauth/token` is limited to 100 req/hour → may return `429`.
- Not yet tested against the real IdP when the user has no IdP session.
- The old `sso-handoff-link-opens-document-as-erp-user` spec is currently failing at `beforeAll` (the Outline login screen now has 2 buttons) — the fixture needs updating, part of phase 6.

## 9. Updates needed after phase 7+

- Phase 6: finalize the main integration document, security review; remove `/sso` + dead `/pending` allow-list code in the bridge; may add an `/api/v1` prefix — update the Base URL in section 1.
- Every time the API contract changes: update this file in the same PR and record it in `docs/project-changelog.md`.
