# HD ERP Docs Sync

Lightweight shell script (`run.sh`) to synchronize Markdown documentation and images from a Git repository to an HD ERP Outline collection.

---

## Prerequisites

- **Git repo** with standard CLI tools: `git`, `tar`, `curl`, `sed`, `grep`.
  - **Linux / macOS**: Available by default.
  - **Windows**: Available out-of-the-box via **Git Bash** or Windows PowerShell with Git in `PATH`.
- **Target `collectionId`** (UUID) and a valid **Sync Token** (`ERP_DOCS_SYNC_TOKEN` starting with `hdk_`).

---

## Quick Setup (3 Steps)

### 1. Copy Configuration Templates
From the root of your repository:
```sh
cp scripts/docs-sync/erp-docs.config.example.json .erp-docs.config.json
cp scripts/docs-sync/env.erp-docs.example .env.erp-docs
```

### 2. Configure Secrets (`.env.erp-docs`)
Edit `.env.erp-docs` with your actual URL and token:
```env
PERMISSION_API_PUBLIC_URL=https://docs.erp.example.com
ERP_DOCS_SYNC_TOKEN=hdk_your_project_sync_token
```
> **Security Rule**: Add `.env.erp-docs` to `.gitignore`. Never commit tokens!
```sh
echo ".env.erp-docs" >> .gitignore
```

### 3. Configure Project Settings (`.erp-docs.config.json`)
Edit `.erp-docs.config.json` in your repository root:
```json
{
  "apiUrl": "${PERMISSION_API_PUBLIC_URL}",
  "token": "${ERP_DOCS_SYNC_TOKEN}",
  "envFile": "scripts/docs-sync/.env.erp-docs",
  "collectionId": "<YOUR_COLLECTION_UUID>",
  "root": "docs/",
  "include": [
    "docs/**"
  ],
  "exclude": [
    "docs/internal/**"
  ]
}
```
> **Important**: Keep `"${PERMISSION_API_PUBLIC_URL}"` and `"${ERP_DOCS_SYNC_TOKEN}"` as literal placeholder strings. Do not paste real tokens into this JSON file.

---

## How to Run

Run the script from anywhere inside the Git repository:

### Dry Run (Preview changes safely)
```sh
sh scripts/docs-sync/run.sh --dry-run
```
*(On Windows PowerShell, run: `sh scripts/docs-sync/run.sh --dry-run` or `bash scripts/docs-sync/run.sh --dry-run`)*

### Live Sync
```sh
sh scripts/docs-sync/run.sh
```

---

## CI/CD Pipeline Setup

### GitHub Actions (`.github/workflows/docs-sync.yml`)

```yaml
name: HD ERP Docs Sync

on:
  push:
    branches:
      - main  # or dev, master
    paths:
      - 'docs/**'
      - '.erp-docs.config.json'
      - 'scripts/docs-sync/**'
  workflow_dispatch:

jobs:
  sync-docs:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Run HD ERP Docs Sync
        env:
          PERMISSION_API_PUBLIC_URL: ${{ vars.PERMISSION_API_PUBLIC_URL || 'https://hd-erp-permission.dev.hdwebsoft.co' }}
          ERP_DOCS_SYNC_TOKEN: ${{ secrets.ERP_DOCS_SYNC_TOKEN }}
        run: |
          chmod +x scripts/docs-sync/run.sh
          sh scripts/docs-sync/run.sh
```

**GitHub Secrets & Variables:**
- Add `ERP_DOCS_SYNC_TOKEN` to **Settings → Secrets and variables → Actions → Repository secrets**.
- (Optional) Add `PERMISSION_API_PUBLIC_URL` to **Repository variables** if not using default endpoint.

### GitLab CI (`.gitlab-ci.yml`)

```yaml
# HD ERP Docs Sync
docs-sync:
  stage: deploy
  image: ubuntu:latest
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      changes:
        - 'docs/**/*'
        - '.erp-docs.config.json'
        - 'scripts/docs-sync/**/*'
  variables:
    PERMISSION_API_PUBLIC_URL: 'https://hd-erp-permission.dev.hdwebsoft.co'
  before_script:
    - apt-get update -qq && apt-get install -y -qq git tar curl sed grep
  script:
    - chmod +x scripts/docs-sync/run.sh
    - sh scripts/docs-sync/run.sh
```

**GitLab Variables:**
- Add `ERP_DOCS_SYNC_TOKEN` to **Settings → CI/CD → Variables** (Mask variable).

---

## Documentation Rules

| Item | Rule |
|---|---|
| **Document Title** | First `# H1 Heading` in each `.md` file becomes the Outline document title. If missing, filename is used. |
| **Folder Node** | Place `README.md` or `index.md` inside a folder to serve as the folder's parent document. |
| **Images** | Supported formats: `.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp`. Use standard relative paths like `![alt](./images/pic.png)`. |
| **Archiving** | If a tracked markdown file is deleted or un-tracked in Git, the next sync archives it in Outline. |

---

## Exit Codes

- `0`: Sync succeeded (`"ok": true`).
- `1`: Sync failed (HTTP error, connection issue, or missing variable).
- `2`: Configuration error (e.g., literal token committed in `.erp-docs.config.json`).
