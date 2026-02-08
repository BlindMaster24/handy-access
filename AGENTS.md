# AGENTS.md

Operational guide for working in this repository and maintaining a personal fork release line.

## Local Rules

- Never push directly to `upstream`.
- Use `main` as the only release branch.

## Fork Ownership Model

This repo is used in dual mode:

- `origin` = your product fork (release source).
- `upstream` = original project (source of incoming updates).

Expected remotes:

```powershell
git remote -v
```

Expected output pattern:

- `origin` -> `https://github.com/BlindMaster24/handy-access.git`
- `upstream` -> `https://github.com/cjpais/Handy.git`

## Product Scope Policy

Current product policy for your fork:

- Keep model download sources as-is (do not migrate model hosting).
- Change update and release source to your GitHub repo.
- Keep contribution path to upstream available at all times.

Meaning:

- `src-tauri/src/managers/model.rs` model URLs (`blob.handy.computer`) remain unchanged unless explicitly decided later.
- Tauri updater endpoint must point to your release JSON in your repo.

## Branching Strategy (Main Only)

Release branch:

- `main` only.

Working branch types:

- `feat/<topic>`
- `fix/<topic>`
- `chore/<topic>`
- `hotfix/<topic>`
- `upstream-sync/<date-or-topic>`
- `contrib/<topic>` (for PRs to upstream)

### Daily Feature Flow

```powershell
git checkout main
git pull origin main
git checkout -b feat/<short-topic>
```

Work, run checks, push branch, open PR to `origin/main`.

After merge:

```powershell
git checkout main
git pull origin main
git branch -d feat/<short-topic>
git push origin --delete feat/<short-topic>
```

Branch cleanup policy:

- Delete merged topic branches (local and remote).
- Keep only active branches and `main`.
- Keep long-lived branches only if there is a clear operational reason.

## Upstream Sync Strategy

Sync from original project on a schedule (for example weekly or per needed feature/fix).

### Safe Sync Flow

```powershell
git fetch upstream
git checkout main
git pull origin main
git checkout -b upstream-sync/<yyyy-mm-dd>
git merge upstream/main
```

Then run full checks, open PR `upstream-sync/* -> main`, and merge through PR.

### If You Do Not Want Some Upstream Changes

- Do not merge upstream wholesale.
- Use selective adoption:
  - cherry-pick specific commits you want.
  - skip or revert changes you do not want.

Example:

```powershell
git fetch upstream
git log upstream/main --oneline
git checkout -b upstream-sync/selective-<topic> main
git cherry-pick <sha1> <sha2>
```

Resolve conflicts, run checks, open PR to `main`.

## Contribution Back to Upstream

Use a clean branch based on `upstream/main`, not from your product-custom branch.

```powershell
git fetch upstream
git checkout -b contrib/<topic> upstream/main
```

Implement isolated change, push to your fork, open PR to upstream:

```powershell
git push -u origin contrib/<topic>
gh pr create --repo cjpais/Handy --base main --head BlindMaster24:contrib/<topic> --title "<type: short-title>" --body-file NUL
```

Rules for upstream PRs:

- No fork branding changes.
- No fork-specific release/update endpoint changes.
- Keep PR scope minimal and clean.

## Release and Auto-Update Strategy (Your GitHub)

Goal:

- App updates are downloaded from your GitHub Releases.
- Model downloads stay on existing model URLs.

### 1) Update Endpoint Must Point to Your Repo

File:

- `src-tauri/tauri.conf.json`

Change updater endpoint to your repo:

```json
"plugins": {
  "updater": {
    "endpoints": [
      "https://github.com/BlindMaster24/handy-access/releases/latest/download/latest.json"
    ]
  }
}
```

Do not change model URLs in `src-tauri/src/managers/model.rs`.

### 2) Updater Signing Keys (Required)

Generate your own updater key pair (do not reuse upstream keys):

```powershell
bun run tauri signer generate -w ./.keys/tauri/updater.key
```

Command prints a public key. Put that value into:

- `src-tauri/tauri.conf.json` -> `plugins.updater.pubkey`

Do not commit private keys.

### 3) GitHub Secrets for Updater Signing

Set in your fork repo secrets:

- `TAURI_SIGNING_PRIVATE_KEY`
- `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`

Example:

```powershell
gh secret set TAURI_SIGNING_PRIVATE_KEY --repo BlindMaster24/handy-access < ./.keys/tauri/updater.key
echo "<your-password>" | gh secret set TAURI_SIGNING_PRIVATE_KEY_PASSWORD --repo BlindMaster24/handy-access
```

### 4) Verify Release Workflows

Workflows involved:

- `.github/workflows/release.yml`
- `.github/workflows/build.yml`

These workflows publish assets and updater metadata (`latest.json`) used by Tauri updater.

Important:

- Updater signing requires `TAURI_SIGNING_PRIVATE_KEY` and `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`.
- Current build workflow also supports platform code signing (Apple and Windows trusted signing). If those signing secrets are missing, some platform release jobs can fail even when updater key is correct.

Trigger release workflow manually:

```powershell
gh workflow run release.yml --repo BlindMaster24/handy-access
```

Check runs:

```powershell
gh run list --repo BlindMaster24/handy-access --limit 20
gh run view <run-id> --repo BlindMaster24/handy-access --log-failed
```

### 5) Release Procedure

1. Ensure `main` is clean and up to date.
2. Update version in `src-tauri/tauri.conf.json`.
3. Run checks.
4. Push `main`.
5. Trigger release workflow.
6. Verify release assets and `latest.json`.
7. Test update detection on an installed previous app version.

## Pre-Release Checklist

Run before every release:

```powershell
cd D:\downloads\repos\Handy
git checkout main
git pull origin main
bun run format:check
bun run lint
bun run build
$env:VULKAN_SDK = "C:\VulkanSDK\1.4.341.1"
cargo check --manifest-path src-tauri/Cargo.toml
git diff --name-only --cached
```

Checklist:

- No merge markers.
- Updater endpoint points to your repo.
- Updater pubkey matches your private key pair.
- Model source URLs unchanged unless explicitly planned.

## Rename and Rebrand Runbook

Recommended branding:

- Display name: `Handy Access`
- Repository slug: `handy-access`

Alternative names:

- `blind-handy`
- `screenfree-dictate`
- `voicebridge-access`
- `sightless-voice`

### Rename Steps

1. Rename repository in GitHub settings.
2. Update local remote URL.
3. Update updater endpoint URL in `tauri.conf.json`.
4. Verify workflows still run on renamed repo.

Commands:

```powershell
git remote set-url origin https://github.com/BlindMaster24/handy-access.git
git remote -v
gh repo view BlindMaster24/handy-access --json name,nameWithOwner,url,defaultBranchRef
```

Optional later branding updates (separate PR):

- `src-tauri/tauri.conf.json` -> `productName`
- `src-tauri/tauri.conf.json` -> `identifier`
- README/project badges/links

## Development Commands

**Prerequisites:** [Rust](https://rustup.rs/) (latest stable), [Bun](https://bun.sh/)

```bash
# Install dependencies
bun install

# Frontend only
bun run dev
bun run build
bun run preview

# Run in development mode
bun run tauri dev
# If cmake error on macOS:
CMAKE_POLICY_VERSION_MINIMUM=3.5 bun run tauri dev

# Build for production
bun run tauri build

# Linting and formatting (run before committing)
bun run lint              # ESLint for frontend
bun run lint:fix          # ESLint with auto-fix
bun run format            # Prettier + cargo fmt
bun run format:check      # Check formatting without changes
```

### Tauri CLI Helpers

```bash
# Show environment and toolchain info
bun run tauri info

# Updater signing key generation
bun run tauri signer generate -w ./.keys/tauri/updater.key

# Sign an updater artifact manually (example)
bun run tauri signer sign --private-key ./.keys/tauri/updater.key --password "<password>" ./path/to/artifact
```

### Windows Local Run Helpers

```powershell
# Ensure Bun in PATH (example)
$env:PATH = "C:\Users\<your-user>\.bun\bin;$env:PATH"

# Vulkan SDK env (if needed for local checks/build)
$env:VULKAN_SDK = "C:\VulkanSDK\1.4.341.1"

# Kill stale dev processes
Get-Process bun,node,cargo -ErrorAction SilentlyContinue | Stop-Process -Force

# Free Vite port 1420 if occupied
$p = Get-NetTCPConnection -LocalPort 1420 -State Listen -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty OwningProcess
if ($p) { Stop-Process -Id $p -Force }
```

**Model Setup (Required for Development):**

```bash
mkdir -p src-tauri/resources/models
curl -o src-tauri/resources/models/silero_vad_v4.onnx https://blob.handy.computer/silero_vad_v4.onnx
```

## Architecture Overview

Handy is a cross-platform desktop speech-to-text app built with Tauri 2.x (Rust backend + React/TypeScript frontend).

### Backend Structure (src-tauri/src/)

- `lib.rs` - Main entry point, Tauri setup, manager initialization
- `managers/` - Core business logic:
  - `audio.rs` - Audio recording and device management
  - `model.rs` - Model downloading and management
  - `transcription.rs` - Speech-to-text processing pipeline
  - `history.rs` - Transcription history storage
- `audio_toolkit/` - Low-level audio processing:
  - `audio/` - Device enumeration, recording, resampling
  - `vad/` - Voice Activity Detection (Silero VAD)
- `commands/` - Tauri command handlers for frontend communication
- `shortcut.rs` - Global keyboard shortcut handling
- `settings.rs` - Application settings management

### Frontend Structure (src/)

- `App.tsx` - Main component with onboarding flow
- `components/settings/` - Settings UI (35+ files)
- `components/model-selector/` - Model management interface
- `components/onboarding/` - First-run experience
- `hooks/useSettings.ts`, `useModels.ts` - State management hooks
- `stores/settingsStore.ts` - Zustand store for settings
- `bindings.ts` - Auto-generated Tauri type bindings (via tauri-specta)
- `overlay/` - Recording overlay window code

### Key Patterns

**Manager Pattern:** Core functionality organized into managers (Audio, Model, Transcription) initialized at startup and managed via Tauri state.

**Command-Event Architecture:** Frontend to Backend via Tauri commands; Backend to Frontend via events.

**Pipeline Processing:** Audio to VAD to Whisper/Parakeet to Text output to Clipboard/Paste.

**State Flow:** Zustand to Tauri command to Rust state to persistence via `tauri-plugin-store`.

## Internationalization (i18n)

All user-facing strings must use i18next translations. ESLint enforces this (no hardcoded strings in JSX).

**Adding new text:**

1. Add key to `src/i18n/locales/en/translation.json`
2. Use in component: `const { t } = useTranslation(); t('key.path')`

## Code Style

**Rust:**

- Run `cargo fmt` and `cargo clippy` before committing.
- Handle errors explicitly (avoid unwrap in production).
- Use descriptive names and doc comments for public APIs.

**TypeScript/React:**

- Strict TypeScript, avoid `any` types.
- Functional components with hooks.
- Tailwind CSS for styling.
- Path aliases: `@/` -> `./src/`.

## Commit Guidelines

Use conventional commits:

- `feat:` new features
- `fix:` bug fixes
- `docs:` documentation
- `refactor:` code refactoring
- `chore:` maintenance

## Debug Mode

Access debug features: `Cmd+Shift+D` (macOS) or `Ctrl+Shift+D` (Windows/Linux).

## Platform Notes

- **macOS**: Metal acceleration, accessibility permissions required
- **Windows**: Vulkan acceleration, code signing
- **Linux**: OpenBLAS + Vulkan, limited Wayland support, overlay disabled by default
