# System Architecture

## Overview

The system has 3 main components + 2 supporting packages:
- **Outline** (v1.10.1): document platform, web UI
- **OIDC Bridge** (`apps/oidc-bridge`): SSO login gateway + IdP relay
- **Permission API** (`apps/outline-permission-api`): user permission management, OAuth document creation & trees, analytics, and Git repository docs sync
- **outline-api-client** (`packages/outline-api-client`): HTTP client wrapper for the Outline API, shared by the setup scripts
- **outline-workspace-setup** (`packages/outline-workspace-setup`): 2 idempotent CLI scripts: apply Outline configuration + branding, register the OAuth client

Flow: ERP → calls the API to manage permissions, create nodes, or sync repository docs → SSO link / CI script → Bridge / Permission API → Outline permissions and documents applied

---

## Packages & shared libraries

### `@hd-document/app-database`
- Database schema migrations (node-pg-migrate)
- Sets up the Postgres connection pool
- Defines the `bridge` + `permission_api` schemas (roles separated per service)

### `@hd-document/outline-api-client`
- HTTP client wrapper for the Outline JSON-RPC API (`POST {OUTLINE_URL}/api/{method}`)
- Typed error classes: `OutlineApiError`, `OutlineBadRequestError`, `OutlineUnauthorizedError`, `OutlineNotFoundError`, `OutlineRateLimitedError`, `OutlineTimeoutError`, `OutlineNetworkError`
- Retry logic: automatically retries 429 + 5xx (bounded by `Retry-After` and attempt count), per-call timeout
- API methods: `getCurrentTeam()`, `updateTeam()`, `listOAuthClients()`, `findOAuthClientByName()`, `createOAuthClient()`, `updateOAuthClient()`
- Non-idempotent calls (e.g. `createOAuthClient`) have a `retry: false` option to avoid duplicates

### `@hd-document/outline-workspace-setup`
- CLI: `apply-outline-workspace-settings` — reads env, changes team settings via `team.update` (branding, security, invitation policy). Idempotent: re-running changes nothing if settings already match.
- CLI: `register-outline-oauth-client` — registers the OAuth client for the permission API, idempotent lookup by name
- Env validation (zod): `WORKSPACE_NAME`, `WORKSPACE_LOGO_URL`, `WORKSPACE_ACCENT_COLOR`, `OUTLINE_ADMIN_API_TOKEN`, `OUTLINE_OAUTH_CLIENT_ID/SECRET` (if already created)

---

## Authentication & authorization (Auth & Bridge)

### User login

The bridge supports **2 paths** for authenticating users:

#### 1. Real IdP (upstream OIDC) — **recommended, current**

- **Issuer:** https://idp.dev.hdwebsoft.co (OpenIddict)
- **Flow:** Authorization Code + PKCE S256
- **Detailed flow:**
  ```
  Browser → GET /interaction/:uid (no handoff cookie)
           → Bridge redirect → IdP /connect/authorize?code_challenge&state&nonce
           → User logs in at the IdP
           → IdP redirect → Bridge /upstream/callback?code&state
           → Bridge verifies (verify id_token RS256 via jwks_uri)
           → Lookup erp_users by sub → sso_handoffs → finishLogin
  ```

- **Env vars:**
  | Variable | Meaning | Example |
  |------|---------|--------|
  | `UPSTREAM_OIDC_ISSUER_URL` | IdP issuer URL | `https://idp.dev.hdwebsoft.co` |
  | `UPSTREAM_OIDC_CLIENT_ID` | Client ID at the IdP | `hd-dochub` |
  | `UPSTREAM_OIDC_CLIENT_SECRET` | Secret (optional; empty = public client) | — |
  | `UPSTREAM_OIDC_SCOPES` | Requested scope (default) | `openid profile email` |
  | `BRIDGE_PUBLIC_URL` | Bridge's public URL (redirect URI registered at the IdP) | `https://bridge.example.com` |

- **Keying:** The transaction token (PKCE verifier, state, nonce) is stored in the `hd_upstream_login` cookie (signed, httpOnly, lax, path `/upstream`, 10 minutes, read-once).

#### 2. ERP handoff (JWT token) — **optional, legacy**

- **Issuer:** ERP self-signs a short-lived JWT
- **Flow:** Browser GET `/sso?token=<jwt>` → Bridge verifies via JWKS/PEM → finishLogin
- **Env vars:**
  | Variable | Meaning |
  |------|---------|
  | `ERP_SSO_ISSUER` | JWT issuer claim (e.g., `urn:erp:sso`) |
  | `ERP_SSO_JWKS_URL` | ERP's JWKS endpoint URL |
  | `ERP_SSO_PUBLIC_KEY_PEM` | Public key PEM (if not using JWKS) |

- **Requirement:** The JWT must have a `jti` claim (lives ≤ 120s, verified once).

- **Status:** When `UPSTREAM_OIDC_ISSUER_URL` is set, the `/sso` route is **optional**. Configuration requires at least 1 path (upstream **or** ERP key).

### Break-glass admin

- **Login page** `GET /interaction/:uid`: "Login with SSO" button (→ `GET /interaction/:uid/upstream` → IdP) + a `system_admin` form below it. Does not auto-redirect to the IdP.
- **Fallback:** if the IdP is down → the SSO button returns 503, the `system_admin` form still works.

### Gating & permissions

- **All users (upstream + ERP)** must exist in `permission_api.erp_users` (status `active`).
- **Email/name:** Comes from `erp_users`, not from IdP claims (for now; may be relaxed later).
- **Audit:** Event `upstream_login` → success / rejected (reason: `transaction_missing`, `idp_denied`, `callback_invalid`, `idp_unavailable`, `unknown_user`, `deactivated`, `email_reserved_for_system_admin`).

---

## Docs Sync (Git Repository -> Outline)

Server-side documentation synchronization replaces client-heavy scripts (PR #7) with a secure, centralized architecture:
- **Consumer repo stays minimal:** Contains only `.erp-docs.config.json` and a lightweight POSIX shell script (`run.sh`, using only standard `git`, `tar`, and `curl`).
- **Zero secrets stored in git:** The repo config uses `${VAR_NAME}` placeholders resolved from `.env.erp-docs` (gitignored) or CI environment variables at runtime.
- **Server owns Outline state and tokens:** All Outline Admin API tokens, diffing algorithms, image uploads, advisory locking, and state mapping live inside `outline-permission-api`.

### Data Flow

```
CI / Local: git ls-files | tar -cz | curl POST /sync-docs[?dryRun=1]  (Bearer hdk_ sync token)
API: auth(docs:sync) -> rate limit -> read gzip (<=50MB) -> safe extract -> config(zod)
  -> collection check (403) -> filter globs (picomatch) -> build tree (folders, README) -> try lock
  -> documents.list(collection) + mapping rows -> plan actions -> apply (upload imgs,
     create/update/move/restore/archive) -> upsert mapping -> unlock -> JSON result + audit
```

### Key Architectural Pillars

1. **Safe Tar Extraction & Zip-Bomb Protection:**
   - In-memory streaming extraction using `tar-stream` and `node:zlib` gunzip with strict bounds.
   - Request body cap: 50 MB (`MAX_ARCHIVE_REQUEST_BYTES`).
   - Archive entry limit: 5,000 files (`MAX_ARCHIVE_ENTRIES`).
   - Single file uncompressed limit: 20 MB (`MAX_FILE_BYTES`).
   - Total uncompressed limit: 200 MB (`MAX_UNCOMPRESSED_BYTES`).
   - Path security sanitization: Rejects null bytes, backslashes, absolute paths, and `..` directory traversal outside archive root. Only regular files and directories permitted (symlinks, fifos, and sockets strictly forbidden).

2. **Attachment Deduplication & Markdown Processing:**
   - Markdown documents are scanned for referenced image paths (`.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp`).
   - Images are uploaded to Outline via `attachments.create` and indexed by SHA-256 hash in `permission_api.doc_sync_attachments`. Re-syncing the same image reuses the cached URL without re-uploading.
   - Markdown image references are dynamically rewritten to Outline CDN/storage URLs before saving.
   - First `# H1 Heading` is extracted as the document title (falls back to filename). Directory nodes use `README.md` or `index.md` as container documents.

3. **Advisory Locking & Live Diff Planning:**
   - Uses PostgreSQL session advisory locking (`pg_try_advisory_lock` keyed on `collection_id`) to prevent race conditions during concurrent syncs (returns `409 SYNC_IN_PROGRESS` if busy).
   - Fetches live Outline documents via a single paginated `documents.list({ collectionId })` call and joins with `permission_api.doc_sync_documents`.
   - Computes SHA-256 content hashes (`title + body + parentId`) to plan minimal mutations (`create`, `update`, `move`, `archive`, `restore`, `skip`).
   - Deleted files in Git repository are automatically archived in Outline.

4. **Sync Token Management & Rotation:**
   - Scoped service tokens with scope `docs:sync` bound to specific projects.
   - Admin/Planner endpoints (`docs-sync-tokens:write`) with dual routing (`/collections/:id/sync-tokens` and `/projects/:key/sync-tokens`).
   - Default limit of 1 active token per collection/project.
   - Atomic 1-click **Rotate** (`POST .../rotate`) revokes previous active key(s) and issues a new key in a single database transaction, preventing onboarding and CI rotation deadlocks.
   - Fast `lastUsedAt` lookup via index on `permission_api.api_audit_log (service_client_id, occurred_at DESC)`.

---

## Database & schema

### `app-database`
- **Postgres** (v16): ICU locale vi-VN
- **Schemas:**
  - `bridge`: SSO handoffs, audit log
  - `permission_api`: Users, projects, role grants, sync mappings, attachments, service client keys
- **Migrations:** node-pg-migrate (numeric prefix)

### Key tables

| Table | Schema | Purpose |
|-------|--------|---------|
| `sso_handoffs` | `bridge` | One-time token handoff (JWT jti or upstream UUID) |
| `auth_audit_log` | `bridge` | Audit events (upstream_login, sso_login, admin_login) |
| `erp_users` | `permission_api` | User from ERP (erp_user_id, email, status) |
| `project_collection_map` | `permission_api` | Map between ERP projectKey and Outline collectionId |
| `service_clients` | `permission_api` | Service client API keys and docs sync tokens |
| `api_audit_log` | `permission_api` | Audit log for permission API calls (indexed for token `lastUsedAt`) |
| `user_outline_grants` | `permission_api` | Sealed OAuth user tokens for document creation and tree read |
| `pending_document_requests` | `permission_api` | Transient user consent requests |
| `idempotency_keys` | `permission_api` | Idempotent mutation tracking |
| `doc_sync_documents` | `permission_api` | Maps repository paths to Outline documents, content hashes, and folder flags |
| `doc_sync_attachments` | `permission_api` | Deduplication cache for image attachments uploaded to Outline by SHA-256 |

---

## Containers & deployment

### Docker Compose
- **Postgres 16** (role isolation: `outline`, `hd_document_apps`)
- **Redis 7** (Outline cache)
- **Outline 1.10.1** (digest pinned, local file storage)
- **OIDC Bridge** (Node.js, tsx runtime)
- **Permission API** (Node.js, tsx runtime, pending)

### Backup
- Custom-format Postgres dump + tar.gz volume
- Script: `docker compose run --rm backup`
- Restore testing: phase 6

---

## Dependencies & libraries

### OIDC Bridge (`apps/oidc-bridge`)
- `openid-client@^6.8.8`: OIDC client (panva; discovery, PKCE, id_token verification)
- `jose`: JWT operations
- `zod`: Config validation
- `oidc-provider`: Test fixture (fake IdP)

### Outline Permission API (`apps/outline-permission-api`)
- `koa` & `@koa/router`: Web application framework and routing
- `zod`: Request body validation, environment parsing, and `.erp-docs.config.json` schema validation
- `tar-stream`: Streaming, memory-bounded extraction of POSIX tar archives without disk I/O
- `picomatch`: High-performance glob matching for `include` and `exclude` path filtering
- `uuid`: RFC4122 UUID v5 deterministic namespace hashing for document/folder identifiers
- `@hd-document/outline-api-client`: Outline JSON-RPC client for document state queries, attachment uploads, and document mutations
- `@hd-document/app-database`: Postgres connection pool and migration management

### Outline API Client (`packages/outline-api-client`)
- Fetch API (built-in); no HTTP library used
- TypeScript + Zod: typed requests/responses, error handling

### Outline Workspace Setup (`packages/outline-workspace-setup`)
- `@hd-document/outline-api-client`: consumes the wrapper client
- `zod`: validates workspace settings from env

### Database
- `node-pg-migrate@9.0.0`: Schema migrations

---

## Appendix: Follow-up items

- **Phase 3+:** Production hardening (HTTPS, TRUST_PROXY, IdP discovery cache invalidation).
- **Phase 4:** Provision `erp_users.erp_user_id` = IdP `sub` UUID.
- **Phase 4+:** Remove `/sso`, `ERP_SSO_*` once upstream is proven.
- **TBD:** Logout (end_session_endpoint callback); profile refresh (email/name from IdP).
