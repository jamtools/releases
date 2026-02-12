# SongDrive Releases - Auto-Update Setup

## Overview

The `releases-tauri-update` branch is **ready to go** with automatic version bumping and Tauri auto-updater support.

## What's Already Implemented

✅ **Concurrency Control** - Prevents concurrent builds from causing version conflicts
✅ **Automatic Version Bumping** - Fetches current version from R2, auto-increments patch version
✅ **Multi-Platform Builds** - macOS (universal), Windows
✅ **Signature Generation** - Minisign signatures for update verification
✅ **Manifest Generation** - Creates `latest.json` with all platform data
✅ **R2 Upload** - Uploads artifacts and `latest.json` to R2 bucket

## Required GitHub Secrets

Configure these in the repository settings (Settings → Secrets and variables → Actions):

### Application Repository Access
- `REPO` - Full repository name (e.g., `username/ffmpeg-songdrive`)
- `REPO_TOKEN` - GitHub PAT with repo access

### R2 Bucket Configuration
- `R2_DESKTOP_APP_KEY_ID` - R2 access key ID
- `R2_DESKTOP_APP_KEY_SECRET` - R2 secret access key
- `R2_DESKTOP_APP_BUCKET` - Bucket name (e.g., `songdrive-desktop-releases`)
- `R2_DESKTOP_APP_ENDPOINT` - R2 endpoint URL (e.g., `https://<account-id>.r2.cloudflarestorage.com`)
- `R2_DESKTOP_BUILDS_PUBLIC_URL` - Public URL for downloads (e.g., `https://releases.songdrive.com`)

### Apple Code Signing (for macOS)
- `APPLE_CERTIFICATE` - Base64-encoded .p12 certificate
- `APPLE_CERTIFICATE_PASSWORD` - Certificate password
- `APPLE_SIGNING_IDENTITY` - Signing identity name
- `APPLE_ID` - Apple ID for notarization
- `APPLE_PASSWORD` - App-specific password for notarization
- `APPLE_TEAM_ID` - Apple Developer Team ID

### Tauri Code Signing
- `TAURI_SIGNING_PRIVATE_KEY` - Minisign private key for update signatures
- `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` - Password for the signing key

## Workflow Trigger

The workflow is triggered by `repository_dispatch` with type `new-commit-on-main`:

```yaml
on:
  repository_dispatch:
    types: [new-commit-on-main]
```

### Trigger from ffmpeg-songdrive repo

Add a GitHub Actions workflow in `ffmpeg-songdrive/.github/workflows/`:

```yaml
name: Trigger Desktop Build

on:
  push:
    branches: [main]

jobs:
  trigger-build:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger releases repo
        uses: peter-evans/repository-dispatch@v2
        with:
          token: ${{ secrets.RELEASES_REPO_TOKEN }}
          repository: username/songdrive-releases
          event-type: new-commit-on-main
          client-payload: |
            {
              "sha": "${{ github.sha }}",
              "profile": "production",
              "site_url": "https://songdrive.app"
            }
```

Required secret in ffmpeg-songdrive repo:
- `RELEASES_REPO_TOKEN` - GitHub PAT with `repo` and `workflow` scopes

## How It Works

### 1. Version Computation
```python
# Fetches https://releases.songdrive.com/builds/desktop/latest.json
# Reads current version (e.g., 0.2.0)
# Increments patch: 0.2.0 → 0.2.1
# If latest.json doesn't exist, bootstraps to 0.1.0
```

### 2. Concurrency Protection
```yaml
concurrency:
  group: desktop-build-all
  cancel-in-progress: false  # Queue instead of cancel
```

Multiple rapid commits will queue, not conflict:
- Commit 1 → Builds v0.2.1
- Commit 2 (queued) → Waits for Commit 1
- Commit 2 starts → Fetches v0.2.1, builds v0.2.2

### 3. Build Process
- Checks out ffmpeg-songdrive at specified commit SHA
- Updates `tauri.base.conf.json` to new version
- Builds for all platforms
- Signs with production keys
- Creates platform-specific manifests with signatures

### 4. Manifest Publishing
- Collects platform manifests from all build jobs
- Merges into single `latest.json`:
  ```json
  {
    "version": "0.2.1",
    "pub_date": "2026-02-12T15:00:00Z",
    "notes": "Bug fixes and improvements",
    "platforms": {
      "darwin-aarch64": {
        "signature": "...",
        "url": "https://releases.songdrive.com/builds/desktop/universal-apple-darwin/0.2.1/SongDrive.app.tar.gz"
      },
      "darwin-x86_64": { ... },
      "windows-x86_64": { ... }
    }
  }
  ```
- Uploads to R2: `builds/desktop/latest.json`

### 5. Artifact Upload
- Uploads signed installers to R2
- Path structure: `builds/desktop/{platform}/{version}/{filename}`
- Makes artifacts publicly accessible via HTTP

## R2 Bucket Configuration

### Bucket Structure
```
songdrive-desktop-releases/
├── builds/
│   └── desktop/
│       ├── latest.json                                    ← Worker fetches this
│       ├── universal-apple-darwin/
│       │   └── 0.2.1/
│       │       └── SongDrive.app.tar.gz                   ← Desktop app downloads
│       └── windows-x86_64/
│           └── 0.2.1/
│               └── SongDrive.msi
```

### Public Access
Enable public HTTP access to the bucket:
- **Option A:** Enable R2.dev subdomain (automatic)
- **Option B:** Connect custom domain `releases.songdrive.com`

Update worker environment variable:
```bash
cd workers/tauri-updater
wrangler secret put ARTIFACT_BASE_URL --env production
# Enter: https://releases.songdrive.com
```

## Testing

### Manual Trigger
In GitHub Actions:
1. Go to Actions → Desktop Build All Platforms
2. Click "Run workflow"
3. Select branch: `releases-tauri-update`
4. Fill in required fields:
   - `sign_app`: true
   - `profile`: production
   - `site_url`: https://songdrive.app
   - Leave `publish_version` empty for auto-bump
5. Run workflow

### Verify Results
1. Check workflow logs for version computed
2. Verify R2 bucket has `builds/desktop/latest.json`
3. Test update endpoint:
   ```bash
   curl https://updates.songdrive.com/darwin/aarch64/0.1.0
   ```
4. Launch desktop app, should auto-update

## Merging to Main

Once tested, merge `releases-tauri-update` to `dev` and then to `main`:

```bash
cd ~/code/songdrive-releases
git checkout dev
git merge releases-tauri-update
git push origin dev

# After verification
git checkout main
git merge dev
git push origin main
```

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ ffmpeg-songdrive repo                                       │
│  • Commit pushed to main                                    │
│  • Triggers repository_dispatch                             │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ songdrive-releases repo (releases-tauri-update branch)      │
│                                                              │
│  1. Fetch latest.json from R2                               │
│  2. Bump patch version (0.2.0 → 0.2.1)                     │
│  3. Build all platforms with new version                    │
│  4. Sign artifacts                                          │
│  5. Generate latest.json manifest                           │
│  6. Upload to R2                                            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ R2 Bucket (songdrive-desktop-releases)                      │
│  • builds/desktop/latest.json                               │
│  • Signed installers for all platforms                      │
│  • Public HTTP access                                       │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ Cloudflare Worker (updates.songdrive.com)                   │
│  • Fetches latest.json via HTTP                             │
│  • Returns platform-specific update info                    │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ Desktop App                                                 │
│  • Checks for updates on startup                            │
│  • Auto-downloads if available                              │
│  • Auto-restarts to apply update                            │
└─────────────────────────────────────────────────────────────┘
```

## Status

✅ All workflows configured and ready
✅ Concurrency control implemented
✅ Auto version bumping from R2
✅ Multi-platform build support
✅ Signature generation
✅ Manifest generation
✅ R2 upload configured

**Next Steps:**
1. Configure required GitHub secrets (see above)
2. Set up R2 bucket with public HTTP access
3. Test manual workflow trigger
4. Configure trigger from ffmpeg-songdrive repo
5. Merge to main after successful test
