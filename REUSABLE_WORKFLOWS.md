# Reusable Package Building Workflows

This repository provides reusable GitHub Actions workflows for building distribution packages (Alpine, Debian, Termux) for your Rust tools.

## Overview

Similar to our `rust-ci.yml` workflow, these workflows are designed to be called from your tool repositories:

- **`package-alpine.yml`** - Builds Alpine Linux `.apk` packages
- **`package-debian.yml`** - Builds Debian/Ubuntu `.deb` packages (amd64, arm64)
- **`package-termux.yml`** - Builds Termux/Android `.deb` packages (aarch64)

All packages are uploaded to **your tool's repository releases**, not to `homebrew-packages`.

## Quick Start

### 1. Setup in Your Tool Repository

Create `.github/workflows/packages.yml` in your tool repository (e.g., `quickctx`, `gdl`, `tapeshell`):

```yaml
name: Build Distribution Packages

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      version:
        description: 'Version (without v, e.g., 0.1.4)'
        required: true

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.normalize.outputs.version }}
      release_tag: ${{ steps.normalize.outputs.release_tag }}
    steps:
      - name: Normalize version and release tag
        id: normalize
        run: |
          RAW_VALUE="${{ github.event.release.tag_name || inputs.version }}"
          RAW_VALUE="${RAW_VALUE#refs/tags/}"
          VERSION="${RAW_VALUE#v}"
          echo "version=${VERSION}" >> "$GITHUB_OUTPUT"
          echo "release_tag=v${VERSION}" >> "$GITHUB_OUTPUT"

  alpine:
    needs: prepare
    uses: CaddyGlow/homebrew-packages/.github/workflows/package-alpine.yml@main
    permissions:
      contents: write
    with:
      tool: quickctx  # ← Change to your tool name
      version: ${{ needs.prepare.outputs.version }}
      repository: ${{ github.repository }}
      release_tag: ${{ needs.prepare.outputs.release_tag }}

  debian:
    needs: prepare
    uses: CaddyGlow/homebrew-packages/.github/workflows/package-debian.yml@main
    permissions:
      contents: write
    with:
      tool: quickctx  # ← Change to your tool name
      version: ${{ needs.prepare.outputs.version }}
      repository: ${{ github.repository }}
      release_tag: ${{ needs.prepare.outputs.release_tag }}

  termux:
    needs: prepare
    uses: CaddyGlow/homebrew-packages/.github/workflows/package-termux.yml@main
    permissions:
      contents: write
    with:
      tool: quickctx  # ← Change to your tool name
      version: ${{ needs.prepare.outputs.version }}
      repository: ${{ github.repository }}
      release_tag: ${{ needs.prepare.outputs.release_tag }}
```

### 2. Release Flow

When you create a new release:

```bash
# In your tool repository (e.g., quickctx)
git tag v0.1.5
git push --tags
gh release create v0.1.5 --generate-notes
```

This will automatically:
1. ✅ Build binaries via `rust-ci.yml` (if you're using it)
2. ✅ Build Alpine packages → upload to your release
3. ✅ Build Debian packages → upload to your release
4. ✅ Build Termux packages → upload to your release

### 3. Result

Your release will contain:

```
CaddyGlow/quickctx/releases/tag/v0.1.5/
├── quickctx-x86_64-unknown-linux-gnu.tar.gz        # From rust-ci.yml
├── quickctx-aarch64-unknown-linux-gnu.tar.gz       # From rust-ci.yml
├── quickctx-aarch64-linux-android.tar.gz           # From rust-ci.yml
├── quickctx-0.1.5-r0.apk                           # From package-alpine.yml
├── quickctx_0.1.5_amd64.deb                        # From package-debian.yml
├── quickctx_0.1.5_arm64.deb                        # From package-debian.yml
└── quickctx_0.1.5_android-aarch64.deb              # From package-termux.yml
```

## Workflow Inputs

All three workflows accept the same inputs:

| Input | Required | Description |
|-------|----------|-------------|
| `tool` | Yes | Tool name (e.g., "quickctx") |
| `version` | Yes | Version without `v` prefix (e.g., `0.1.4`) |
| `repository` | Yes | Repository in format owner/repo |
| `release_tag` | No | Release tag to upload to (default: `v{version}`) |

## Requirements

For the workflows to work, your tool repository must have:

1. **Required Artifacts** (from `rust-ci.yml` or similar):
   - Alpine: `{tool}-{release_tag}-x86_64-unknown-linux-musl.tar.gz`
   - Alpine: `{tool}-{release_tag}-aarch64-unknown-linux-musl.tar.gz`
   - Debian: `{tool}-{release_tag}-x86_64-unknown-linux-gnu.tar.gz`
   - Debian: `{tool}-{release_tag}-aarch64-unknown-linux-gnu.tar.gz`
   - Termux: `{tool}-{release_tag}-aarch64-linux-android.tar.gz`

2. **Optional Metadata** (in `homebrew-packages`):
   - `Formula/{tool}.rb` - For binary names and description
   - `alpine/APKBUILD.{tool}` - For custom Alpine build config
   - `termux/build.sh.{tool}` - For custom Termux build config

## How It Works

1. **Checkout**: Workflows checkout both `homebrew-packages` (for metadata) and your tool repo
2. **Download**: Fetch pre-built binaries from your release
3. **Package**: Build distribution-specific packages using Docker
4. **Upload**: Upload packages to your tool's release using `GITHUB_TOKEN`

## Permissions

The workflows use the caller's `GITHUB_TOKEN` automatically. Make sure to grant `contents: write`:

```yaml
jobs:
  alpine:
    permissions:
      contents: write  # ← Required
    uses: CaddyGlow/homebrew-packages/.github/workflows/package-alpine.yml@main
```

## Manual Triggering

You can trigger package builds manually:

```bash
# Via GitHub CLI
gh workflow run packages.yml -f version=0.1.4

# Via GitHub UI
Actions → Build Distribution Packages → Run workflow
```

## Comparison to Old System

| Aspect | Old System | New System |
|--------|-----------|------------|
| **Location** | Packages in `homebrew-packages` | Packages in tool repos |
| **Triggering** | Manual via `generate-apk-packages.yml` | Automatic on release |
| **Release Tags** | Date-based (`alpine-packages-v20251026`) | Version-based (`v0.1.4`) |
| **URLs** | Change daily | Permanent |
| **Discovery** | Hidden in separate repo | Standard GitHub releases |

## Migration Guide

To migrate an existing tool (e.g., quickctx):

1. Add `packages.yml` to tool repository
2. Create a new release or re-run existing release workflow
3. Verify packages appear in tool's release
4. Update documentation to point to tool's releases
5. (Optional) Clean up old date-based releases in `homebrew-packages`

## Troubleshooting

**Q: Workflow fails with "Resource not accessible by integration"**
A: Add `permissions: contents: write` to your job

**Q: Can't find artifact to download**
A: Ensure `rust-ci.yml` ran first and created the required `.tar.gz` files

**Q: Want custom APKBUILD or build.sh**
A: Create template files in `homebrew-packages/alpine/` or `homebrew-packages/termux/`

## Examples

See:
- `.github/workflows/EXAMPLE-packages.yml` - Template for tool repos
- Existing workflows like `rust-ci.yml` for similar patterns

## Support

For issues or questions, open an issue in the `homebrew-packages` repository.
