# Project Changelog

## [2026-11-10] Server-side Git repo docs sync & sync token management

**Status:** ✓ Done  
**Scope:** `apps/outline-permission-api/src/docs-sync/`, `apps/outline-permission-api/src/docs-sync-tokens/`, `packages/app-database/migrations/0007-create-doc-sync-mapping-and-attachment-tables.sql`, `packages/outline-api-client/src/`, `scripts/docs-sync/`, `docs/erp-integration-api-guide.md`, `docs/system-architecture.md`

Replaces the reverted client-side sync approach (PR #7) with a secure, centralized server-side sync service:
- **Consumer repo footprint minimized:** Consumer repos only maintain `.erp-docs.config.json` and a POSIX shell script (`run.sh`, using standard git/tar/curl). No Node.js runtime, Outline dependencies, or Outline API tokens needed in consumer repos.
- **New endpoint `POST /sync-docs` (scope `docs:sync`):**
  - Accepts gzipped POSIX tar archive (`application/gzip`). Supports `?dryRun=1` to simulate and preview action plans.
  - Safe in-memory streaming extraction with strict zip-bomb and payload caps: 50 MB max request body (`MAX_ARCHIVE_REQUEST_BYTES`), 5,000 files max (`MAX_ARCHIVE_ENTRIES`), 20 MB max per uncompressed file (`MAX_FILE_BYTES`), 200 MB max total uncompressed size (`MAX_UNCOMPRESSED_BYTES`).
  - Path security sanitization: Rejects path traversal (`..`), absolute paths, null bytes, backslashes, symlinks, FIFOs, and devices.
  - Configuration parsing and validation via Zod with placeholder resolution (`${VAR_NAME}`). Rejects literal secrets.
  - Markdown processing: Extracts first `# H1 Heading` as document title (fallback to filename), maps folder `README.md`/`index.md` to folder parent nodes.
  - Image attachments: Extracts markdown images (`.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp`), uploads to Outline via `attachments.create`, deduplicates via SHA-256 in `permission_api.doc_sync_attachments`, and rewrites markdown image URLs to Outline URLs.
  - Live diff planning: Queries Outline `documents.list({ collectionId })` in a single paginated pass and joins with `permission_api.doc_sync_documents` to plan `create`, `update`, `move`, `archive`, `restore`, or `skip` actions. Deleted repository files are automatically archived in Outline.
  - Concurrency safety: Non-dry-run syncs acquire a PostgreSQL session advisory lock (`pg_try_advisory_lock`) per `collection_id`, returning `409 SYNC_IN_PROGRESS` if concurrent sync is detected.
- **New sync token management endpoints (scope `docs-sync-tokens:write`):**
  - Dual routing: supports both `/collections/:collectionId/sync-tokens` and `/projects/:projectKey/sync-tokens`.
  - `GET .../sync-tokens`: lists active tokens with `id`, `name`, `createdAt`, and `lastUsedAt` (queried from `api_audit_log`).
  - `POST .../sync-tokens`: issues a new scoped token (`docs:sync`). Enforces `maxAllowed` (default 1 active token) with `409 SYNC_TOKEN_LIMIT_REACHED`.
  - `POST .../sync-tokens/rotate`: atomic 1-click token rotation that revokes old token(s) and creates a new token in a single transaction, preventing lockout deadlocks.
  - `DELETE .../sync-tokens/:id`: revokes single token by 16-hex ID (`204 No Content` / `404 SYNC_TOKEN_NOT_FOUND`).
  - `DELETE .../sync-tokens?all=true`: emergency revocation of all active tokens for the target.
- **Database migration `0007-create-doc-sync-mapping-and-attachment-tables.sql`:**
  - Creates `permission_api.doc_sync_documents` for repo-to-Outline document mappings and SHA-256 content hashes.
  - Creates `permission_api.doc_sync_attachments` for attachment URL deduplication by SHA-256.
  - Creates index `api_audit_log_service_client_occurred_at_idx` for fast `lastUsedAt` lookup.
- **Client tooling and documentation:**
  - Client shell script `scripts/docs-sync/run.sh`, config template `erp-docs.config.example.json`, and env template `env.erp-docs.example`.
  - Consumer onboarding documentation in `scripts/docs-sync/README.md`.
  - Full API and UI contract for Project Planner team documented in `docs/erp-integration-api-guide.md` sections 3.9 & 3.10.

### Testing

- Unit tests for docs-sync pure modules:
  - `doc-file-glob-filter.test.ts` (7 tests)
  - `doc-sync-content-hash.test.ts` (3 tests)
  - `doc-sync-tree-builder.test.ts` (8 tests)
  - `docs-sync-config-schema.test.ts` (10 tests)
  - `markdown-processor.test.ts` (9 tests)
  - `plan-doc-sync-actions.test.ts` (8 tests)
  - `safe-docs-archive-extractor.test.ts` (12 tests)
  - `upload-referenced-images.test.ts` (4 tests)
  - `resolve-sync-collection-access.test.ts` (4 tests)
  - `collection-advisory-lock.test.ts` (3 tests)
  - `apply-doc-sync-actions.test.ts` (8 tests)
  - `sync-docs-service.test.ts` (3 tests)
  - `sync-docs-routes.test.ts` (3 tests)
- Unit & integration tests for sync-token management:
  - `docs-sync-token-service.test.ts` (10 tests)
  - `docs-sync-token-routes.test.ts` (12 tests)
  - `docs-sync-tokens.integration.test.ts` (1 test)
- Repository tests:
  - `doc-sync-mapping-repository.test.ts` (4 tests)
  - `doc-sync-attachment-repository.test.ts` (3 tests)
- Overall suite: `pnpm --filter @hd-document/outline-permission-api test` → 42 test files / 262 tests pass; `typecheck` clean.

## [2026-11-03] Document view reminder mail API

**Status:** ✓ Done
**Scope:** `apps/outline-permission-api/src/analytics/` (new service + routes + tests), `src/mail/mailer.ts`, `src/create-permission-api-application.ts`, `src/cli/manage-service-client-key-cli.ts`, `docs/erp-integration-api-guide.md`

New endpoint `POST /documents/:documentId/view-reminders` (new scope `analytics:notify`), see guide section 3.7c:
- Gentle "you haven't opened this document yet" mail with document title + absolute link. Body mirrors the invite endpoints but carries no permission: `{ "emails": [...] }`, 1–200 valid addresses, required.
- **No collection-level validation on purpose** — a reminder grants nothing and mutates nothing in Outline, so dropping both `resolveDocumentProjectScope` and `assertDocumentInSharedCollection` is what makes one route work for a node in a private collection *and* in a workspace-shared collection. Only check: the document exists (`404 DOCUMENT_NOT_FOUND`).
- Recipients come from the caller (typically `unviewedMembers` of 3.7 / 3.7b); the API does not recompute who has viewed. Case-insensitive duplicates and the reserved `SYSTEM_ADMIN_EMAIL` land in `skipped` instead of being mailed, so `reminded.length + skipped.length` equals the request size.
- Response: `{ documentId, documentTitle, documentUrl, reminded, skipped }`. One `documents.info` call regardless of recipient count; mails sent sequentially so a single bad address cannot abort the batch.
- New scope `analytics:notify` (separate from read-only `analytics:read`) added to the CLI allow-list — existing keys must be rotated with the scope to use the endpoint.
- `MailerService.sendDocumentViewReminderEmail()` added (English wording, consistent with the existing invite mails) plus HTML escaping of the document title, which unlike the existing enum-only interpolations is free text from Outline.

### Testing

- `send-document-view-reminders-service.test.ts` (9 cases): mail payload (title + joined URL) per recipient, case-insensitive dedupe, reserved account skipped, reminded/skipped accounting, `/doc/<id>` URL fallback, single `documents.info` per call, 404 with no mail sent, unexpected Outline error propagated (not masked as 404), service without `systemAdminEmail` configured.
- `pnpm --filter @hd-document/outline-permission-api test` → 22 files / 138 tests pass; `typecheck` clean; `eslint` clean on changed files (also fixed the stale `MailerService` double in `set-project-member-role-service.test.ts` that broke `tsc`).

## [2026-11-02] Shared node view stats (implicit member set)

**Status:** ✓ Done
**Scope:** `apps/outline-permission-api/src/analytics/` (new service + route), `src/shared-collection-tree/assert-document-in-shared-collection.ts` (new), `src/shared-collection-tree/get-shared-document-content-service.ts`, `src/users/erp-user-repository.ts`, `create-permission-api-application.ts`, `docs/erp-integration-api-guide.md`

New endpoint `POST /shared-documents/:documentId/view-stats` (scope `analytics:read`), see guide section 3.7b:
- Complements `POST /documents/:id/view-stats`, which derives its member list from node invitations. A workspace-shared node has no invitation list, so the member set is **implicit**: every active ERP user in `permission_api.erp_users`, minus the reserved `SYSTEM_ADMIN_EMAIL` account. A newly created shared node reports all of them in `unviewedMembers`; the list shrinks as `views.list` records arrive.
- Response adds `totalMembers` (member-set size) next to `totalViews` / `uniqueViewers` / `viewers` / `unviewedMembers`. `unviewedMembers` entries are `{ erpUserId, name, email }` — never null-id, since members come from the ERP DB rather than from Outline.
- Members with no `outline_user_id` yet (not synced into Outline) are reported as unviewed instead of being dropped. `totalViews` still sums every view record, including viewers with no ERP mapping.
- `POST` with an optional body: `{ "excludeErpUserIds": [...] }` (max 500) **narrows** the member set — excluded users leave it entirely, so they appear in neither `viewers` nor `unviewedMembers` and drop out of `totalMembers`. Unknown IDs are ignored. No `memberIds` override: unlike 3.7 the caller cannot define the member list, only shrink it. The body may be omitted (empty body parses as `{}`).
- `totalViews` deliberately stays unfiltered (raw node total, includes excluded and non-ERP viewers).

Supporting changes:
- **Extracted** `assertDocumentInSharedCollection()` from `get-shared-document-content-service.ts` into its own module, now shared by both `/shared-documents/*` routes (content + view stats). Same behaviour: `404 DOCUMENT_NOT_FOUND`, `403 DOCUMENT_NOT_IN_SHARED_COLLECTION` for private/restricted/inaccessible collections.
- `ErpUserRepository.listActive(excludeEmails?)`: active users ordered by display name, reserved emails filtered in SQL.
- Fixed four test doubles of `ErpUserRepository` that had gone stale (they were missing `findByErpUserIds`, which broke `tsc --noEmit` before this change).

### Testing

- `get-shared-document-view-stats-service.test.ts` (9 cases): all-unviewed on a fresh node, member moving out of `unviewedMembers` after a view, member without Outline mapping, non-member views counted in `totalViews` only, reserved + deactivated users excluded, `excludeErpUserIds` removing a viewer and a non-viewer at once, unknown exclude IDs ignored, 404 and 403 guard paths.
- `pnpm --filter @hd-document/outline-permission-api test` → 21 files / 127 tests pass; `typecheck` clean; `eslint` clean on changed files.

## [2026-10-23] Shared collection tree API

**Status:** ✓ Done  
**Scope:** `apps/outline-permission-api/src/shared-collection-tree/` (routes + services), `create-permission-api-application.ts`, `cli/manage-service-client-key-cli.ts`, `docs/erp-integration-api-guide.md`

New endpoints under scope `shared-tree:read`:
- `GET /shared-collection-tree`: returns all workspace All-Members collections with their document trees as seen by the admin token. No OAuth grant flow — uses the admin Outline client directly. Input validated by Zod (`actingErpUserId: uuid`, `depth: 1–5 default 2`). User existence and active status checked; 404/403 on unknown or deactivated user. Per-collection 1000-node cap with `truncated` flag; concurrent Outline fetches bounded at 5. Scope added to CLI allow-list. See `docs/erp-integration-api-guide.md` section 3.6.
- `GET /shared-documents/:documentId/content`: returns document content (`documentId`, `title`, `text`, `url`) using the admin token. Enforces security access guard verifying the document belongs to a workspace-shared collection (`permission: "read"` or `"read_write"`), returning 403 `DOCUMENT_NOT_IN_SHARED_COLLECTION` if private, restricted, or inaccessible, and 404 `DOCUMENT_NOT_FOUND` if non-existent.

## [2026-10-16] Single Swarm stack: one root compose file, one dev pipeline, one shared env file

**Status:** ✓ Done (not yet exercised on the real runners/manager)  
**Scope:** `docker-compose.swarm.yml` (repo root, new), `infra/docker-compose.migrate.swarm.yml`, `.github/workflows/dev-pipeline.yaml` (new), `.dockerignore`, `infra/swarm.env.example` (new, replaces `infra/outline.env.example`), `apps/*/.env.example`, `infra/README.md`

### Context

- Three separate Swarm stacks/pipelines (`infra/docker-compose.swarm.yml`, `infra/docker-compose.outline-permission-api.swarm.yml`, `infra/docker-compose.outline.swarm.yml` + `dev-oidc-bridge-pipeline.yaml`, `dev-outline-permission-api-pipeline.yaml`, `dev-outline-pipeline.yaml`) each copied its own `config/.env` under its own `PROJECT_ROOT`. Requested: one compose file at the repo root, deployed as `${PROJECT_ROOT}/docker-compose.swarm.yml`, with one shared env file `config/.env` for all services (per-service files were considered first and dropped: no name collisions between the three services, and one file is simpler to maintain).

### Added / changed

- **`docker-compose.swarm.yml`** (root, new): `oidc-bridge`, `outline-permission-api`, `outline`, `outline-redis` in one stack (`hd-erp-docs-dev`), all with `env_file: ${PROJECT_ROOT}/config/.env`. Image names are fixed per service (`hd-erp-docs-oidc-bridge`, `hd-erp-docs-outline-permission-api`); ports via `OIDC_BRIDGE_PORT` / `PERMISSION_API_PORT` / `OUTLINE_PORT`; shared deploy/logging anchors. Verified the shared names (`OUTLINE_URL`, `OIDC_CLIENT_ID/SECRET`, `SYSTEM_ADMIN_EMAIL`, `TRUST_PROXY`, `ERP_PORTAL_URL`, `SMTP_HOST/PORT`) carry the same value for every service and that none of the env schemas rejects unknown keys.
- **Removed** the two unused Redis services (`oidc-bridge-redis`, `outline-permission-api-redis`): neither app reads `REDIS_URL`. Only `outline-redis` remains (`${PROJECT_ROOT}/data/outline-redis`).
- **`infra/docker-compose.migrate.swarm.yml`**: now always uses the oidc-bridge image (both app images bundle the migration CLI at `/app/migration`) and the same `config/.env` (must contain `APP_DATABASE_URL`); dropped the unused `PORT`/`CONTAINER_PORT`.
- **`.github/workflows/dev-pipeline.yaml`** (new, replaces the three per-service workflows): `build` (matrix of the 2 images) → `migrate` (once, `flock`-serialised) → `deploy` (single stack, rollout check + health curl for all three apps). Migration now runs **before** deploy instead of in parallel. Single env-file input `HD_ERP_DOCS_ENV_FILE` (a GitHub _variable_, so it can be read back and edited; only `DOCKER_REGISTRY_PASSWORD` and `SSH_PRIVATE_KEY` remain secrets) replaces `OIDC_BRIDGE_ENV_FILE` / `OUTLINE_PERMISSION_API_ENV_FILE` / `OUTLINE_ENV_FILE`. No `chmod` on the manager: the file is created with `umask 077` on the runner and `scp` keeps the mode. `PROJECT_ROOT=/mnt/nfs/apps/hd-erp-docs`.
- **`infra/swarm.env.example`** (new): single template for the shared env file (shared keys once, then per-service sections, `[SECRET]` markers); replaces `infra/outline.env.example`. `apps/*/.env.example` headers now point to it.
- Deleted `infra/docker-compose.swarm.yml`, `infra/docker-compose.outline-permission-api.swarm.yml`, `infra/docker-compose.outline.swarm.yml` and the three old workflow files. `.dockerignore` excludes `docker-compose*.yml` from image build contexts.
- `infra/README.md` "Deploy to Docker Swarm (CI/CD)" rewritten for the new layout (it still described a pipeline and scripts that no longer existed).

### Testing

- `docker stack config -c docker-compose.swarm.yml` and `-c infra/docker-compose.migrate.swarm.yml` render without errors with the workflow's variables exported and a dummy `config/.env` in place (YAML merge of two anchors per service resolves correctly).
- Workflow YAML parses; job graph `build → migrate → deploy` verified.

### Remaining work

- Create the `HD_ERP_DOCS_ENV_FILE` variable in the `dev` environment from `infra/swarm.env.example` (merge the values of the three old `*_ENV_FILE` entries) and run the workflow once against the real manager. The old per-service `PROJECT_ROOT` folders (`/mnt/nfs/apps/hd-erp-docs/{oidc-bridge,outline-permission-api,outline}`) and stacks (`hd-erp-docs-dev-oidc-bridge`, `-outline-permission-api`, `-outline`) are not removed automatically: `docker stack rm` them before the first deploy so the published ports are free, and move `outline/data/redis` to `data/outline-redis` if Outline's Redis data should be kept.

## [2026-10-16] CI/CD rewritten to the company template: self-hosted runners, Docker Hub images, Docker Swarm deploy

**Status:** ✓ Done (pipeline not yet exercised on the real runners/manager — see "Remaining work")  
**Scope:** `.github/workflows/hd-document-dev.yml`, `infra/docker-compose.swarm.yml`, `infra/scripts/deploy-swarm-remote.sh`, `infra/scripts/prepare-swarm-environment.mjs`, `tsconfig.json`, `infra/README.md`

### Context

- The company's standard pipelines (`backend-dev.yml` / `frontend-dev.yml` reference files) run on the `hd-runners` self-hosted group inside the `dev` GitHub environment, build images with Docker-in-Docker, push to Docker Hub with the shared `DOCKER_REGISTRY_*` secrets and deploy with `docker stack deploy` over SSH (`SSH_PRIVATE_KEY` / `SSH_USER` / `SWARM_MANAGER_HOST`). The previous project pipeline instead built on a VPS with `docker compose` through its own `VPS_*` secrets.

### Added / changed

- **`.github/workflows/hd-document-dev.yml`** (new, replaces `ci-lint-typecheck-test.yml` + `deploy-vps.yml`): `lint-and-test` (PR + push to `dev`; Postgres service container initialised by the real `postgres-init/01-create-databases.sql`, random host port so jobs can share a runner) → `build` (matrix of 3 images, dind, tags `latest` + commit SHA) → `deploy` (SSH to the Swarm manager, env from the `HD_DOCUMENT_ENV_FILE` environment variable, key/env files kept in `$RUNNER_TEMP` and removed afterwards). Workflow-level concurrency: deploys queue per branch, PR runs cancel stale ones.
- **`infra/docker-compose.swarm.yml`** (new): Swarm version of the stack from prebuilt images — no `depends_on`/`build`/`security_opt`, stateful services pinned to the manager, attachable overlay network, `update_config: start-first + failure_action: rollback` for the two apps, only proxy-facing ports published.
- **`infra/scripts/deploy-swarm-remote.sh`** (new, replaces `deploy-remote.sh`): first deploy bootstraps Postgres via the stack, then migrations run as a one-off `hd-document-migrate` container _before_ `docker stack deploy --detach=false`, followed by rollout-state checks and health curls.
- **`infra/scripts/prepare-swarm-environment.mjs`** (new): compose-style `.env` → `export KEY='…'` converter (`docker stack deploy` substitutes from the shell only); `tsconfig.json` now includes `infra/scripts/*.mjs` so it is typechecked and linted.
- `infra/README.md`: "Deploy to a VPS (CI/CD)" replaced by "Deploy to Docker Swarm (CI/CD)" (environment secrets/variable, what the env file must contain for the server, deploy flow, manual operations).

### Testing

- `docker stack config -c infra/docker-compose.swarm.yml` with a fully populated env renders without errors (images, ports, placement, network resolved).
- Converter verified on quoting edge cases (single/double quotes, `$`, JSON, spaces, `<`, inline comments, empty values) and rejects malformed lines; `bash -n` on the deploy script; `tsc`, `eslint`, `prettier --check` pass on all new files.

### Remaining work (not done, left to the user)

- Set `HD_DOCUMENT_ENV_FILE` in the `dev` environment and run the workflow once against the real Swarm manager (needs the reverse-proxy URLs and a Docker-Hub-logged-in SSH user).
- Remove the local reference copies in `.github/workflows/demo/` before committing (untracked; GitHub ignores sub-folders, kept only as the template source).
- Backups on the Swarm host are still manual.

## [2026-10-13] VPS deploy CI/CD: auto DB init + migration on first run

**Status:** ✓ Done  
**Scope:** `.github/workflows/deploy-vps.yml`, `infra/scripts/deploy-remote.sh`, `infra/docker-compose.yml`, `packages/app-database/`, `infra/README.md`

### Context
- No VPS deploy pipeline existed yet; `render.yaml` only covers Render (staging/test). Needed: push code → auto-update the VPS, first run auto-creates roles/databases + runs migrations, later runs stay safe to repeat.

### Added / changed
- **`infra/docker-compose.yml`**: new `migrate` service (profile `migrate`, built from `packages/app-database/Dockerfile`), always targets the internal `postgres` service directly (unlike `.env`'s `APP_DATABASE_URL`, which is written for host-side tools).
- **`packages/app-database/Dockerfile`** (new): image runs the migrate CLI directly via `tsx` (added `tsx` to the package's dependencies so the filtered install pulls it in).
- **`.github/workflows/deploy-vps.yml`** (new): push to `dev` (or `workflow_dispatch`) → SSH into the VPS → runs `infra/scripts/deploy-remote.sh`.
- **`infra/scripts/deploy-remote.sh`** (new): `git reset --hard` the VPS checkout → `docker compose ... run --rm --build migrate` (profile `local-db`: empty volume on first run → `postgres-init/*.sql` auto-creates roles/databases; `node-pg-migrate` is idempotent so re-running on every deploy stays safe) → `docker compose ... up -d --wait --build` for the remaining services.
- `infra/README.md`: added "Deploy to a VPS (CI/CD)" section — one-time VPS setup, required GitHub secrets, deploy flow.

### Testing
- `docker build -f packages/app-database/Dockerfile .` builds successfully; running the CLI against a fake DB fails with the expected connection error (no missing-module error).
- `docker compose --profile local-db --profile migrate config` resolves correctly: `migrate` has `restart: "no"`, `depends_on.postgres.condition: service_healthy`, `APP_DATABASE_URL` pointing at `postgres:5432`.
- `pnpm install` updated the lockfile for `app-database`'s new `tsx` dependency.

### Remaining work (not done, left to the user)
- Create the GitHub secrets (`VPS_HOST`/`VPS_PORT`/`VPS_USER`/`VPS_SSH_KEY`/`VPS_DEPLOY_PATH`) and clone + fill in `infra/.env` on the real VPS for the first time — needs concrete VPS details not available in this session.
- TLS reverse proxy in front of the VPS is out of scope (services still bind to `127.0.0.1`).

## [2026-10-06] Email invitations at collection level; added `admin` (Manage) node permission

**Status:** ✓ Done  
**Scope:** `apps/outline-permission-api/src/projects/`, `src/document-permissions/`, `src/users/`, `src/mail/`, `docs/`

### Context
- The ERP "Collection access" screen only allowed changing roles for existing ERP members (hardcoded 4 accounts) — external emails could not be invited like in Outline.
- The node permissions modal only supported Can view / Can edit; missing Manage (`admin`).

### Added / changed
- **`PUT /projects/:key/invitations`** `{ email, role }` + **`DELETE ...?email=`**: grant/revoke collection roles for arbitrary email addresses (creates an Outline account if one doesn't exist, sends an email containing the collection link). Returns `403 SYSTEM_ADMIN_EMAIL_RESERVED` for the system admin email.
- **`GET /projects/:key/members`**: now returns external users outside ERP (`erpUserId: null`), excluding the system admin email.
- **Node permissions**: now accepts `permission: admin` in both `/members/:erpUserId` and `/invitations`.
- DRY: extracted `users/find-or-invite-outline-user-by-email.ts` shared between doc + collection; added `sendCollectionInviteEmail` to mailer.
- ERP-fake: Collection access modal has email input + role + Invite; "invited by email" rows can be updated/revoked by email; node modal adds the Manage option.

### Testing
- Unit: 104 tests pass (added 4 tests for setRoleByEmail/removeMemberByEmail, listing external users). Typecheck passes. ERP-fake `next build` passes.

## [2026-10-06] Document tree for users with node-only shares; API for listing collection permissions

**Status:** ✓ Done  
**Scope:** `apps/outline-permission-api/src/document-tree/`, `src/projects/`, `packages/outline-api-client/`, `docs/`

### Context (bug report)
- During sync, ERP-fake was pushing **all** project members into the `manager` group → collection-level permissions blanketed everything, rendering individual node shares pointless (users could see everything in Outline). Fixed in ERP-fake: sync only assigns collection roles to project managers; added "Collection access" screen (Manage/Edit/View/No access).
- Tree API returned `403` for users with only node-level access (in Outline 1.10.1, `collections.documents` requires collection membership).

### Added / changed
- **Tree API fallback:** if `collections.documents` returns 403/404 → build tree from shared nodes (`userMemberships.list` filtered by collection, descendant nodes via `documents.list` + membership escape). If empty → still returns `403 ACTING_USER_FORBIDDEN` (no information leak). `parentDocumentId` in fallback is verified using `documents.info` with the user token.
- **`GET /projects/:projectKey/members`** (scope `permissions:write`): returns current collection roles of each ERP user (maps 3 groups → `manager|editor|viewer`).
- **Audit:** `PUT/DELETE /projects/:key/members/:erpUserId` now logs `erpUserId` and `role` (previously only `{}`).
- Outline client: added `listUserMembershipDocuments`, `listChildDocuments`, `getDocumentSummary`, `listGroupMemberUserIds`.

### Testing
- Unit: tree service (fallback, 403, 404 parent), list members (29 tests pass). Typecheck passes.
- Manual (real Outline, 2026-10-06): user without collection role, shared `demo1` → tree `hd-farm2` returns only `demo1`; other projects → 403; manager still sees all.

## [2026-10-05] Phase 7: Document tree API for ERP to inspect document hierarchy

**Status:** ✓ Done  
**Scope:** `apps/outline-permission-api/src/document-tree/`, `packages/app-database/migrations/0006`, `docs/`

### Added

- **Endpoint:** `GET /projects/{projectKey}/document-tree?actingErpUserId=<uuid>&parentDocumentId?=<uuid>&depth=<1-5>` (default depth=2).
  - Response `200 {projectKey, parentDocumentId, truncated, nodes:[{id,title,url,parentDocumentId,hasMoreChildren,children[]}]}`.
  - Max 1000 nodes/response; if exceeded → `truncated=true`, call again with `parentDocumentId` = child node.
  - Contains only metadata (no content) to save bandwidth.
- **Service key scope:** `tree:read` (new). Create key: `manage-service-client create ... --scopes tree:read`.
- **OAuth scope:** Default env `OUTLINE_OAUTH_SCOPE` is now `documents:create auth:read read` (auto-appends `read` if missing).
- **User permissions:** Uses user's OAuth token → Outline automatically enforces collection- and document-level permissions.
- **Grant flow:** First-time user (unauthorized `read` scope) → `409 OUTLINE_GRANT_REQUIRED {grantUrl}` → ERP directs user to open grantUrl (reuses `/pending/:id` consent-only flow, migration 0006) → user clicks "Accept" → retry API → `200`.
- **Error handling:** `400` validation, `403 ACTING_USER_FORBIDDEN` (no collection access), `404` user/project/doc not found, `403 USER_DEACTIVATED`, `409` grant required.
- **Audit:** Logs `actingErpUserId`/`projectKey`/`resultStatus`.

### New files
- `src/document-tree/load-project-document-tree-service.ts` — business logic (tree traversal, `findNode`, depth filtering, budget tracking).
- `src/document-tree/document-tree-routes.ts` — route handler + validation schema.
- `src/document-tree/load-project-document-tree-service.test.ts` — unit tests (8 tests).
- Migration `0006-allow-consent-only-pending-requests.sql` — removes NOT NULL from `payload`/`document_id` in `pending_document_requests` (supports consent-only requests without docs).

### Testing

- **Unit:** passes (tree service: depth, parentDocumentId, 409, 403, 1000-node truncation; token scope).
- **Integration (real Outline 1.10.1):** `tests/e2e/document-tree-real-outline-integration.spec.ts` passes (409 grantUrl → consent → 200; scope `read` sufficient for `collections.documents`; no content leak; parent node → multi-level hierarchy; external user → 403). Legacy grant missing `read` → 409 (manual check). All 14 e2e specs pass.

### Notes

- Max depth 5 to prevent deep recursion; query param `depth` validated to 1–5.
- Token scope `read` is requested on the first attempt; once granted (subsequent calls) → immediate `200`.
- Currently uses user tokens (approach a/b/c from plan); if an admin token + role-based filter is needed in the future → set `TREE_READ_OUTLINE_SCOPE = "admin"` and add filtering logic.

## [2026-10-02] Bridge auto-provisions ERP user on first SSO; permission-api runs in compose

**Status:** ✓ Done  
**Scope:** `apps/oidc-bridge/src/{accounts,upstream,config}`, `apps/outline-permission-api` (Dockerfile, config), `packages/outline-api-client`, `infra/`

### Added

- **Auto-provisioning:** IdP user not found in `erp_users` → bridge calls permission API `PUT /users/{sub}` (service key scope `users:write`) with email/name from id_token (or userinfo fallback); then re-queries and completes login. Outline's `inviteRequired` remains untouched as it uses the API's invite path. Bridge env: `PERMISSION_API_SERVICE_KEY` (enabled), `PERMISSION_API_INTERNAL_URL` (compose default `http://outline-permission-api:4100`).
- **Audit:** `upstream_login` success has `autoProvisioned: true`; rejected events add `profile_unusable`, `provision_rejected` (+`code`), `provision_unavailable`.
- **Testing:** `fake-permission-api-server.ts`; harness accepts `{ autoProvision: true }`; 2 new integration tests (successful provisioning, API rejection).

### Fixed

- `outline-permission-api` runs inside compose: `OUTLINE_INTERNAL_URL` (calls Outline via service name), `@hd-document/outline-api-client` moved to `dependencies` and built to `dist` in Dockerfile (Node does not strip `.ts` types in `node_modules`).

## [2026-10-02] Enable public sharing: "Publish to web" toggle for documents

**Status:** ✓ Done  
**Scope:** `packages/outline-workspace-setup/src/desired-workspace-settings.ts`, test, docs  
**Severity:** Configuration / behavior change

### Fixed

- `sharing: false` → `true` in desired team settings. Previously, phase 3 script disabled public sharing at the workspace level → Outline hid the "Publish to web" toggle across all documents even when `collections.sharing = true`.
- Re-ran `apply-workspace-settings` to apply to running Outline instance.

### Notes

- Outline does not support "publishing an entire collection": the Share button on collection pages only manages members. Publishing is on a per-document basis (Doc Share → "Publish to web" + "Include nested documents" for child pages).
- Documents without the toggle turned on remain private as before.

## [2026-10-02] Bridge authentication via real IdP (upstream OIDC)

**Status:** ✓ Done  
**Scope:** `apps/oidc-bridge/src/upstream/`, env, test, dependencies

### Added

- **Upstream OIDC login:** Bridge acts as relying party for real IdP (https://idp.dev.hdwebsoft.co, OpenIddict). Flow: `/interaction/:uid` → `/upstream/callback` with authorization code + PKCE S256.
- **New env:** `UPSTREAM_OIDC_ISSUER_URL`, `UPSTREAM_OIDC_CLIENT_ID`, `UPSTREAM_OIDC_CLIENT_SECRET` (optional; public client), `UPSTREAM_OIDC_SCOPES` (defaults to `openid profile email`).
- **Dependencies:** `openid-client@^6.8.8`.
- **New files:** `src/upstream/{upstream-oidc-client.ts, upstream-login-transaction-cookie.ts, upstream-login-routes.ts}`.
- **Audit:** Event `upstream_login` (success / rejected: `transaction_missing`, `idp_denied`, `callback_invalid`, `idp_unavailable`, `unknown_user`, `deactivated`, `email_reserved_for_system_admin`).
- **Login page:** When IdP is configured, displays "Log in with SSO" button (`GET /interaction/:uid/upstream`) + `system_admin` form below (break-glass fallback if IdP is unavailable).
- **Testing:** 7 integration tests + 89 unit tests pass.

### Fixed

- `ERP_SSO_*` + `/sso` route became **optional** (when either `UPSTREAM_OIDC_ISSUER_URL` or `ERP_SSO_JWKS_URL`/`ERP_SSO_PUBLIC_KEY_PEM` is present).
- Configuration requires at least 1 path (upstream or ERP key).
- `infra/README.md`: added "Login via real IdP" section.

### Notes

- Profile (email/name) comes from `erp_users`, not from IdP.
- User must exist in `erp_users` (active) → link `/upstream/callback` → verify `id_token` (RS256, `iss`, `aud`, `exp`) → match `sub` with `erp_user_id` → sso_handoffs → finishLogin.
- Production rejects HTTP upstream issuer; cookie `hd_upstream_login` (signed, httpOnly, lax, path `/upstream`, 10m, read-once).

## [2026-10-02] Phase 03: Outline configuration & branding, Outline API client

**Status:** ✓ Done  
**Scope:** `packages/outline-api-client`, `packages/outline-workspace-setup`, `infra/`

### Added

- **`packages/outline-api-client`:** Typed HTTP client wrapping Outline JSON-RPC API, built on fetch. Handles timeout (30s), retry with backoff (429/5xx up to 3 times, honors `Retry-After`), error mapping to typed exceptions.
- **`packages/outline-workspace-setup`:** Two idempotent CLI scripts:
  - `apply-outline-workspace-settings`: reads desired state from env, calls `team.update` only if diff detected (name, logo, theme, branding, security settings).
  - `register-outline-oauth-client`: `oauthClients.list` → match by name → create if missing (client ID/secret for phase 5), or update if redirect URI differs.
- **Infra updates:** `.env.example` + `docker-compose.yml` with Outline env (OIDC only, disable DCR, language, admin API token); `README.md` install sequence (login `system_admin` → create API key → run scripts).

### Fixed (Code Review)

- **[High]** `oauthClients.create` retry on all errors → risk of duplicate clients with orphaned secrets. Added `retry: false` option, `createOAuthClient` uses it.
- **[Medium]** `customTheme.accent`/`accentText` patched independently → partial overwrite if one changes. Fixed to always send both fields together in patch.

### Testing

- **Unit:** 23 tests (http client, API layers, settings diff, CLI parsing) pass.
- **Integration:** Live against Outline 1.10.1 Docker container. Both scripts ran twice: second run idempotent (no-op).
  - `team.update`: 11 fields applied (name, logo, theme colors, sharing, guestSignin, passkeys, inviteRequired, membersCanInvite, memberCollectionCreate, memberTeamCreate, membersCanCreateApiKey).
  - `oauthClients.create`: Client already existed, no retry; matched by name, verified no duplicate.

### Notes

- Visual UI confirmation (login buttons, share toggle, member permission buttons) flagged as open Todo, not blocking merge. Browser verification pending.

## [2026-10-02] "Login with ERP" button on bridge login form

**Status:** ✓ Done  
**Scope:** `apps/oidc-bridge` (login page), `infra/README.md`, erp-fake (separate repo)

### Added

- "Admin login" form now includes a **Login with ERP** button (only visible when `ERP_PORTAL_URL` is set) → `<ERP_PORTAL_URL>/sso/start?returnTo=<OUTLINE_URL>`. ERP renders an intermediary page then triggers `window.location` to the bridge's `/sso` endpoint so Referer matches ERP origin.
- `buildErpSsoStartUrl` + unit test `login-page-view.test.ts`.
- erp-fake: `GET /sso/start` → `public/sso-start.html` (auto-signs handoff if already logged into ERP; otherwise redirects to login then returns).

### Fixed (erp-fake)

- `GET /api/projects/:id/documents` was mistakenly mounted inside the `/api/documents` router → request hit the fallback `index.html`, and frontend threw `Unexpected token '<'`. Moved handler to `routes/projects.js`.

### Operational notes

- erp-fake user seed must use the MongoDB `_id` as `--erp-user-id` (= `sub` in JWT); custom arbitrary IDs lead to `unknown_user` → "permission denied" page.

## [2026-10-01] Scope change: skip custom UI, prioritize SSO

**Status:** Plan updated, no code changes yet  
**Scope:** plan, phase files, roadmap  
**Source:** [plan.md § Validation Log, Session 5](../plans/261001-0953-outline-plus-companion-document-system/plan.md)

### Changes

- **Removed:** Companion web app (Next.js, React, shadcn, TanStack Query, template → form → doc); username/password login form validating against ERP and `packages/erp-adapters`; AI generation; sync worker pulling permissions; real ERP adapter.
- **Added:** SSO token handoff (ERP signs short-lived JWT in URL, bridge verifies, 1-click login); headless `apps/outline-permission-api` (ERP pushes users + project/node permissions, creates doc nodes under actual user identity); Outline configuration & branding via env and `team.update`.
- **Retained:** Outline pinned to `1.10.1` + digest, no forks; local `system_admin` as admin / break-glass path; convention: 1 project = 1 private collection + 3 groups.
- **DB Schema:** `companion` renamed to `permission_api` in migration 0002 (schema currently empty); separate DB roles per service.

### Phases

- Renumbered (old → new): 2 → 2 (rewritten) · 3 → 3 + 4 · 4 → 5 · 7 → 6 · 10 → 7 (deferred) · 5, 8, 9 deleted. Phase 1 below references "phase 7" for backup schedule + restore drills: now phase 6.
- Effort: 240 hours (30 person-days) for phases 1-6, no contingency remaining; previously MVP 1 was 30 days + MVP 2 ~19 days.

### Recorded risks

- Doc author is the actual user → each user must consent once via Outline UI; users without an Outline session need to open the link twice on first access. Switching to a service account would save ~30h (pending user decision).
- Browser holds an existing Outline session for a different user: bridge cannot clear it (host-only cookie). Accepted limitation, documented in runbook.
- JWT contract with ERP not yet finalized; dev CLI signs JWTs for local development and testing.

## [2026-10-01] Phase 01: Monorepo & Outline Infrastructure

**Status:** ✓ Done  
**Scope:** Workspace scaffolding, Docker infra, database, backup  
**Validation:** [Tester report](../plans/reports/tester-261001-1419-phase-01-monorepo-and-outline-infra.md) · [Code review](../plans/reports/code-reviewer-261001-1419-phase-01-monorepo-and-outline-infra.md)

### Added

**Monorepo & build:**
- pnpm workspace (v10.30.2), Node 22.13.0+
- Root ESLint + Prettier
- TypeScript 6.0.3, Vitest 5.0.3 with projects config
- CI workflow: lint, typecheck, test, format (GitHub Actions)

**App database package:**
- `@hd-document/app-database` (TS module)
- node-pg-migrate 9.0.0 (advisory lock, numeric prefix ordering)
- Migration 0001: schemas `bridge` + `companion` in `hd_document_apps` database
- CLI: `pnpm --filter @hd-document/app-database migrate`

**Infrastructure (Docker Compose):**
- Postgres 16 (Debian, ICU vi-VN) — Outline + app separated databases
- Redis 7 (Outline cache)
- Outline 1.10.1 (digest pinned, local file storage)
- Role isolation: `outline` / `hd_document_apps` roles, CONNECT revoked from PUBLIC

**Backup:**
- Script: dump 2 Postgres databases (custom format) + tar.gz attachments volume
- COMPLETE marker: backup directory treated as corrupted if missing this file
- Manual execution: `docker compose run --rm backup`

**Infrastructure documentation:**
- `infra/README.md`: first-time setup, dev migration, manual backup, operating rules

### Decisions & Deviations

- **Postgres Debian, not Alpine:** Alpine (musl) does not support collation locales; changing libc with existing data corrupts text indexes.
- **Local file storage, not MinIO:** Official MinIO image was no longer pullable.
- **Outline 1.10.1, not tag with PR #13879:** PR remains open upstream, not included in any release yet (research mistakenly noted as merged).
- **`vitest.config.ts` (test.projects), not `vitest.workspace.ts`:** Vitest 5 deprecated workspace files.
- **TypeScript ~6.0, not 7:** typescript-eslint does not support TS 7 yet.
- Plan adherence: 1 Postgres instance, 2 databases (`outline` Outline-only, `hd_document_apps` for app); backup runs manually for now, automated schedule + restore drills in phase 6.

### Known Limitations & Deferred Items

**Limitations:**
- **Outline 1.10.1 lacks PR #13879 fix:** Moving parent doc with its own share → child doc loses inherited permissions. **Rule:** Do not move docs currently having their own shares (documented in infra/README.md).
- **Attachment upload not tested:** Requires OIDC login → phase 2. Backup service tested successfully.
- **CI workflow not yet run on GitHub:** Ready, pending push.

**Deferred → Phase 2:**
- DB role isolation (W3): `bridge` + `companion` sharing the same role → if companion is compromised, bridge secrets are exposed. Decided at start of phase 2, before real data exists.
- Production runtime path (W6): tsx, native type stripping, or build step? Decided when writing Dockerfile.
- Type-aware lint (S2): `no-floating-promises` deferred to phase 2.

### Testing

- Vitest: 6/6 pass when `APP_DATABASE_URL` is available (real Postgres); without it, 4 pass, 2 migration integration tests skipped.
- typecheck, lint, format:check: pass.
- Stack stood up from empty volume: 3 services healthy, Outline returns HTTP 200.
- Migration ran twice: second run was a no-op.
- Manual backup: tester successfully restored `hd_document_apps.dump` into temporary database.

Open criteria from phase 1: attachment upload (tested in phase 2 once login works).

### Key Files

- Monorepo: `package.json`, `pnpm-workspace.yaml`, `tsconfig.base.json`, `.github/workflows/ci-lint-typecheck-test.yml`
- App database: `packages/app-database/`
- Infrastructure: `infra/` (README, docker-compose.yml, .env.example, postgres-init, backup scripts)
