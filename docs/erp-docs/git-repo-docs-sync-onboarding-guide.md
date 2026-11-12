# Git Repository Documentation Sync — Onboarding Guide

> Technical guide for integrating consumer Git repositories with the **HD ERP Documentation Platform (Outline)** using server-side synchronization.

---

## 1. Overview & Architecture

The **Git Repo Docs Sync** mechanism allows development teams to maintain project documentation directly in Git repositories alongside their codebase while automatically publishing and syncing the rendered documentation to Outline collections.

Unlike traditional client-side sync tools that require bulky Node.js runtimes, repository write access, or administrative API keys on client machines, this architecture uses a **lightweight server-side sync model**:

```
+─────────────────────────────────────────────────────────────+
| Consumer Repository (Local / Developer Workstation)         |
|                                                             |
|   - Tracked Docs (Markdown + Images)                        |
|   - .erp-docs.config.json       (Committed - placeholders)  |
|   - .env.erp-docs               (Gitignored - real secrets) |
|   - scripts/docs-sync/run.sh (POSIX sh: git/tar/curl) |
+──────────────────────────────┬──────────────────────────────+
                               │
               POST /sync-docs (tar.gz payload)
               Authorization: Bearer <token>
                               │
                               v
+─────────────────────────────────────────────────────────────+
| HD ERP Outline Permission API (/sync-docs)                  |
|                                                             |
|   1. Authenticates token & verifies collection access (403) |
|   2. Acquires per-collection PostgreSQL advisory lock (409) |
|   3. Safely unpacks in-memory gzip stream (size/entry caps) |
|   4. Parses .erp-docs.config.json (rejects literal secrets) |
|   5. Filters files via include/exclude globs                |
|   6. Uploads & deduplicates images via SHA-256 hashes       |
|   7. Builds Outline doc hierarchy & maps UUIDv5 identities  |
|   8. Creates / Updates / Moves / Restores / Archives docs   |
|   9. Releases advisory lock and returns JSON sync summary   |
+─────────────────────────────────────────────────────────────+
```

---

## 2. Quickstart (Local Execution)

Follow these steps to set up docs sync in your repository:

### Step 1: Copy sync script and configuration templates
Copy the docs-sync tools into your repository:
```sh
mkdir -p scripts/docs-sync
# Copy run.sh, config example, and env example
cp <path-to-hd-erp-docs>/scripts/docs-sync/run.sh scripts/docs-sync/run.sh
cp <path-to-hd-erp-docs>/scripts/docs-sync/erp-docs.config.example.json .erp-docs.config.json
cp <path-to-hd-erp-docs>/scripts/docs-sync/env.erp-docs.example .env.erp-docs
chmod +x scripts/docs-sync/run.sh
```

### Step 2: Request credentials from the Project Planner / Admin
To sync docs, you need:
1. **`collectionId`**: The UUID of the target collection in Outline.
2. **`ERP_DOCS_SYNC_TOKEN`**: A scoped service client token (begins with `hdk_`) with `docs:sync` permission for your project collection.

The platform admin creates this token using the management CLI or API:
```sh
pnpm --filter @hd-document/outline-permission-api manage-service-client create sync-<projectKey> \
  --scopes docs:sync --project-keys <projectKey>
```

### Step 3: Configure `.env.erp-docs` (Never Commit!)
Open `.env.erp-docs` and fill in your target API URL and token:
```env
PERMISSION_API_PUBLIC_URL=http://localhost:4100
ERP_DOCS_SYNC_TOKEN=hdk_your_project_sync_token
```

> [!IMPORTANT]
> **Ensure `.env.erp-docs` is added to `.gitignore`!**
> ```sh
> echo ".env.erp-docs" >> .gitignore
> git status
> ```
> Never commit real authentication tokens to version control.

### Step 4: Configure `.erp-docs.config.json` (Commit to Git)
Edit `.erp-docs.config.json` in your repository root with your collection details:
```json
{
  "apiUrl": "${PERMISSION_API_PUBLIC_URL}",
  "token": "${ERP_DOCS_SYNC_TOKEN}",
  "envFile": ".env.erp-docs",
  "collectionId": "b1836545-21d4-42b7-a36c-2f924df07c12",
  "root": "docs/",
  "include": [
    "docs/**"
  ],
  "exclude": [
    "docs/internal/**"
  ]
}
```
Commit `.erp-docs.config.json` and `scripts/docs-sync/run.sh` to your repository:
```sh
git add .erp-docs.config.json scripts/docs-sync/run.sh .gitignore
git commit -m "docs: add erp-docs sync script and configuration"
```

### Step 5: Validate with Dry Run
Execute a dry run to inspect the plan without modifying Outline:
```sh
sh scripts/docs-sync/run.sh --dry-run
```
Example output:
```json
{
  "ok": true,
  "dryRun": true,
  "collectionId": "b1836545-21d4-42b7-a36c-2f924df07c12",
  "created": 4,
  "updated": 0,
  "moved": 0,
  "restored": 0,
  "skipped": 0,
  "archived": 0,
  "failed": [],
  "warnings": []
}
```

### Step 6: Perform Real Sync
Run the script without `--dry-run`:
```sh
sh scripts/docs-sync/run.sh
```

---

## 3. Configuration Reference (`.erp-docs.config.json`)

The configuration file resides at the repository root and is parsed on both client and server:

| Field | Type | Default | Description |
|---|---|---|---|
| `collectionId` | `string` (UUID) | *Required* | Outline Collection UUID where docs will be published. |
| `root` | `string` | `"docs/"` | Base documentation folder. When set, this prefix is stripped from the Outline hierarchy so docs directly inside appear at collection top-level. |
| `include` | `string[]` | *Required* (min 1) | Picomatch glob patterns for markdown/assets to include. |
| `exclude` | `string[]` | `[]` | Picomatch glob patterns to exclude. |
| `apiUrl` | `string` | `"${PERMISSION_API_PUBLIC_URL}"` | Endpoint URL or environment variable placeholder. |
| `token` | `string` | `"${ERP_DOCS_SYNC_TOKEN}"` | Environment variable placeholder for authentication. |
| `envFile` | `string` | `".env.erp-docs"` | Relative path to local gitignored env file for client resolution. |

### Environment Variable Placeholders
To guarantee that tokens are never leaked into committed files:
- `token` **MUST** be an environment placeholder matching `"${NAME}"` where `NAME` conforms to `^[A-Z_][A-Z0-9_]*$`.
- Literal tokens (e.g. `"hdk_..."`) are **strictly prohibited**; both `run.sh` (exit code 2) and the server (`400 CONFIG_SECRET_LITERAL`) reject them.
- `apiUrl` may be either an env placeholder (e.g. `"${PERMISSION_API_PUBLIC_URL}"`) or a literal URL (e.g. `"https://docs.example.com"`).

### Resolution Precedence
When `run.sh` resolves placeholder variables:
1. **Process Environment:** Existing shell environment variables (e.g. `ERP_DOCS_SYNC_TOKEN=... sh run.sh` or CI secret injection).
2. **Local Env File:** Reads the file designated by `envFile` (default `.env.erp-docs`) line-by-line via `grep`/`sed` without sourcing or executing the file.

---

## 4. Documentation Hierarchy & Formatting Rules

### Document Title
The title displayed in Outline is determined by the **first level-1 heading** in the Markdown file:
```markdown
# Architecture Overview

Content begins here...
```
- The `# Architecture Overview` line is extracted as the document title and removed from the body rendered in Outline.
- If no `# H1` heading exists, the document filename stem (without extension) is used as the title.

### Directory Nodes & `README.md`
- In Outline, folders are themselves documents that can contain child documents.
- If a directory contains `README.md` or `index.md` (case-insensitive, `README.md` takes precedence), that file's content becomes the body of the folder document. The `README.md` is **not** duplicated as a nested child node.
- If a folder does not contain a `README.md`, an empty placeholder document is automatically generated using the folder name as the title.
- A top-level `README.md` located in the repository root or directly inside `root` is created as a normal top-level document.

### Images and Attachments
- Supported formats: `.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp`.
- Reference images in Markdown using standard relative paths:
  ```markdown
  ![Database Schema](./images/db-schema.png)
  ```
- **Deduplication:** When synced, images are checked against existing attachments in the database using SHA-256 hashes. Identical images are reused rather than uploaded multiple times.
- **URL Rewriting:** Relative image links in your Markdown are automatically rewritten server-side to Outline attachment URLs upon creation and update.

### Archival & Overwrite Behavior
- **Sync identity:** Every document has a deterministic UUIDv5 identifier derived from the collection ID and repository path (`uuidv5(collectionId:repoPath)`).
- **Archiving:** If a previously synced Markdown document is deleted or un-tracked from Git, the next sync automatically moves that document to the Outline archive. Documents created directly in Outline outside of the sync mapping are never touched or deleted.
- **Change Detection:** A document is only updated if its content hash (`sha256(title + parentId + body)`) changes or if the document was updated in Outline since the last sync.

---

## 5. Script Details (`run.sh`)

`scripts/docs-sync/run.sh` is a self-contained POSIX `/bin/sh` script requiring only standard operating system tools: `git`, `tar`, `curl`, `sed`, `grep`.

### Tar Compatibility
The script automatically tests tar capability at runtime:
- **GNU tar & BSD tar (Linux, macOS, Git Bash):** Uses null-delimited streaming (`tar --null -T -`) to safely handle file names with special characters or spaces.
- **BusyBox tar (Alpine Linux):** Transparently falls back to newline-delimited mode (`tar -T -`).
- **macOS AppleDouble Guard:** Automatically sets `COPYFILE_DISABLE=1` to prevent `._*` metadata files from being generated.

### Flags & Exit Codes
- `--dry-run`: Runs a non-destructive simulation (`POST /sync-docs?dryRun=1`).
- `--help` / `-h`: Displays command usage.

**Exit Codes:**
- `0`: Success (HTTP 200 and JSON response has `"ok": true`).
- `1`: Operation failed (HTTP >= 400, `"ok": false`, missing env var, network failure).
- `2`: Configuration error (e.g. literal token found in `.erp-docs.config.json`).

---

## 6. Continuous Integration (CI) Note

> [!NOTE]
> Automated CI/CD execution (GitHub Actions / GitLab CI) is currently deferred pending external runner connectivity verification.
>
> However, `run.sh` is **completely CI-ready**:
> In your CI pipeline, simply configure your CI secrets:
> - `PERMISSION_API_PUBLIC_URL`
> - `ERP_DOCS_SYNC_TOKEN`
>
> And run:
> ```sh
> sh scripts/docs-sync/run.sh
> ```
> Since process environment variables take precedence over `.env.erp-docs`, no `.env` file needs to be created in CI environments.

---

## 7. Troubleshooting & Common Errors

| HTTP Code | Error Code | Root Cause & Resolution |
|---|---|---|
| **`400`** | `CONFIG_MISSING` | `.erp-docs.config.json` was not found in the repository root or was not included in the tar archive. Ensure the file exists. |
| **`400`** | `CONFIG_SECRET_LITERAL` | The configuration file contains a literal token or URL instead of a `"${VAR_NAME}"` placeholder. Replace it with `"${ERP_DOCS_SYNC_TOKEN}"`. |
| **`400`** | `UNSAFE_ARCHIVE_PATH` | The archive contains paths with path traversal (`..`), absolute paths, or invalid characters. |
| **`400`** | `DISALLOWED_FILE_TYPE` | An unsupported file type was included in the archive. Only `.md` and supported images (`.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp`) are permitted. |
| **`401`** | `UNAUTHORIZED` | Token is invalid, expired, revoked, or the `Authorization: Bearer` header is missing. |
| **`403`** | `COLLECTION_FORBIDDEN` | The sync token does not belong to the project associated with the specified `collectionId`, or the token lacks the `docs:sync` scope. |
| **`409`** | `SYNC_IN_PROGRESS` | Another sync operation is currently running for this collection (locked by PostgreSQL advisory lock). Wait a moment and retry. |
| **`413`** | `ARCHIVE_TOO_LARGE` / `PAYLOAD_TOO_LARGE` / `FILE_TOO_LARGE` | The uploaded payload exceeds size limits (maximum 50MB compressed archive, 5,000 entries, 20MB per non-image file, 200MB uncompressed). Oversized image files (>20MB) are automatically skipped with a warning; oversized Markdown or configuration files trigger `FILE_TOO_LARGE`. Remove large binary assets or narrow the `include` glob. |
| **`415`** | `UNSUPPORTED_MEDIA_TYPE` | The request `Content-Type` header is not `application/gzip`. Ensure `run.sh` is used to send the request. |
| **`429`** | `RATE_LIMIT_EXCEEDED` | Exceeded route rate limit (maximum 6 sync requests per minute per client). Wait 60 seconds before initiating another sync. |
