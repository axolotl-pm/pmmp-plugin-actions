<p align="center">
  <a href="https://github.com/axolotl-pm/pmmp-plugin-actions/actions"><img alt="pmmp-plugin-actions status" src="https://github.com/axolotl-pm/pmmp-plugin-actions/workflows/linter/badge.svg"></a>
</p>

# pmmp-plugin-actions

GitHub Actions utilities for building and releasing [PocketMine-MP](https://github.com/pmmp/PocketMine-MP) plugins.

This collection provides reusable workflows and composite actions to build plugin phars, validate release tags, and publish releases or nightly builds.

## Actions

| Action | Description |
|:-------|:------------|
| [`build-phar`](#build-phar) | Set up PHP, build, compress and upload the plugin phar as an artifact |
| [`validate-release`](#validate-release) | Assert that the pushed tag matches the version in `plugin.yml` |
| [`publish-release`](#publish-release) | Download the phar artifact and publish a GitHub release |
| [`publish-nightly`](#publish-nightly) | Force-push the `nightly` tag and publish a pre-release |

## Usage

### Nightly builds

Triggered on every push to your development branch. Builds the phar and publishes it under the `nightly` tag.

```yaml
name: Nightly

on:
  push:
    branches: [main] # Ensure you target your working branch

jobs:
  build:
    uses: axolotl-pm/pmmp-plugin-actions/.github/workflows/build-phar.yml@v1.0.0
    with:
      php-version: '8.2'
      stage-poggit: true
    permissions:
      contents: write  # Required when `stage-poggit: true`; otherwise, use read

  publish:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: axolotl-pm/pmmp-plugin-actions/publish-nightly@v1.0.0
        with:
          artifact-id:    ${{ needs.build.outputs.artifact-id }}
          plugin-name:    ${{ needs.build.outputs.plugin-name }}
          plugin-version: ${{ needs.build.outputs.plugin-version }}
          pm-version:     ${{ needs.build.outputs.pm-version }}
```

### Releases

Triggered on version tags. Validates that the tag matches `plugin.yml`, builds the phar with a SLSA build provenance attestation, and publishes a GitHub release. Tags containing `-` (e.g. `v1.0.0-beta.1`) are automatically marked as pre-releases.

```yaml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'    # v1.0.0
      - 'v[0-9]+.[0-9]+.[0-9]+-*'  # v1.0.0-beta.1, v1.0.0-rc.1
      - '[0-9]+.[0-9]+.[0-9]+'     # 1.0.0
      - '[0-9]+.[0-9]+.[0-9]+-*'   # 1.0.0-beta.1, 1.0.0-rc.1

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: axolotl-pm/pmmp-plugin-actions/validate-release@v1.0.0

  build:
    needs: validate
    uses: axolotl-pm/pmmp-plugin-actions/.github/workflows/build-phar.yml@v1.0.0
    with:
      php-version: '8.2'
      attest: true
    permissions:
      id-token: write
      contents: read
      attestations: write

  publish:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: axolotl-pm/pmmp-plugin-actions/publish-release@v1.0.0
        with:
          artifact-id: ${{ needs.build.outputs.artifact-id }}
          plugin-name: ${{ needs.build.outputs.plugin-name }}
          pm-version:  ${{ needs.build.outputs.pm-version }}
```

### Verifying attestations

When a plugin is released using this pipeline with `attest: true`, you can cryptographically verify that the `.phar` file was built exclusively by `build-phar` and has not been tampered with since — not even by the repository owner.

First, install the [GitHub CLI](https://cli.github.com/) if you haven't already, then run:

**Linux / macOS**
```bash
gh attestation verify Plugin.phar \
  --repo owner/PluginRepo \
  --signer-workflow axolotl-pm/pmmp-plugin-actions/.github/workflows/build-phar.yml
```

**Windows (PowerShell)**
```powershell
gh attestation verify Plugin.phar `
  --repo owner/PluginRepo `
  --signer-workflow axolotl-pm/pmmp-plugin-actions/.github/workflows/build-phar.yml
```

**Windows (CMD)**
```bat
gh attestation verify Plugin.phar ^
  --repo owner/PluginRepo ^
  --signer-workflow axolotl-pm/pmmp-plugin-actions/.github/workflows/build-phar.yml
```

A successful verification confirms:
- The phar was built from some commit in `owner/PluginRepo`
- The build ran exclusively through `axolotl-pm/pmmp-plugin-actions/.github/workflows/build-phar.yml`
- The file has not been modified after the build

To additionally pin verification to a specific commit or ref, use `--source-digest` or `--source-ref`.

---

## Reference

### `build-phar`

**Type:** Reusable workflow

Set up PHP, build and compress a PocketMine-MP plugin phar, upload it as a workflow artifact, and optionally generate a [SLSA build provenance](https://slsa.dev/spec/v1.0/provenance) attestation (SLSA v1.0 Build Level 3).

#### Inputs

| Name | Required | Default | Description |
|:-----|:--------:|:--------|:------------|
| `php-version` | YES | — | Any version available in [`pmmp/PHP-Binaries`](https://github.com/pmmp/PHP-Binaries) (e.g. `8.2`, `8.3`) |
| `attest` | NO | `false` | Generate a SLSA build provenance attestation for the phar. When `true`, the calling job must grant `id-token: write` and `attestations: write` (see [Permissions](#permissions) below). |
| `retention-days` | NO | `1` | Days to retain the uploaded artifact |
| `artifact-name` | NO | `<plugin-name>-phar` | Artifact name override. Useful in matrix builds to avoid collisions. |
| `plugin-dir` | NO | `${{ github.workspace }}` | Directory containing `plugin.yml` and `composer.json` |
| `composer` | NO | `true` | Whether pharynx should include composer dependencies |
| `composer-version` | NO | `2.5.5` | Composer version to use |
| `pharynx-version` | NO | `latest` | Pharynx version to use |
| `additional-sources` | NO | — | Additional source directories, colon-separated, absolute or relative to workspace |
| `additional-assets` | NO | — | Additional assets to include (one pathspec per line, works like `git add`) |
| `stage-poggit` | NO | `false` | Whether to stage the build on Poggit CI |

#### Outputs

| Name | Description |
|:-----|:------------|
| `plugin-name` | Value of the `name` field in `plugin.yml` |
| `plugin-version` | Value of the `version` field in `plugin.yml` |
| `pm-version` | PocketMine-MP API version(s) from `plugin.yml` |
| `artifact-name` | Name of the uploaded artifact |
| `artifact-id` | ID of the uploaded artifact. Use for immutable download to guarantee the phar has not been tampered with after attestation. |

#### Permissions

This workflow does **not** declare its own `permissions` block — it inherits whatever the calling job grants. This is intentional: it avoids forcing callers to grant sensitive permissions (`id-token: write`, `attestations: write`) when attestation is not requested.

The minimum required permissions on the calling job are:

| Permission | Level | When required |
|:-----------|:------|:-------------|
| `contents` | `read` | Always required for `actions/checkout` |
| `contents` | `write` | Only when `stage-poggit: true` |
| `id-token` | `write` | Only when `attest: true` |
| `attestations` | `write` | Only when `attest: true` |

The [usage examples](#usage) above already reflect the correct permissions for each case.

---

### `validate-release`

**Type:** Composite action

Verify that the pushed tag matches the version declared in `plugin.yml`. Fails the job if they differ, preventing mismatched releases.

Strips a leading `v` and any pre-release suffix before comparing (e.g. tag `v1.2.0-beta.1` is compared as `1.2.0`).

**Prerequisites:** `actions/checkout` must run before this action in the same job.

No inputs or outputs — this action's only contract is pass/fail.

---

### `publish-release`

**Type:** Composite action

Download the plugin phar artifact and publish a GitHub release. Pre-releases are detected automatically from the tag name (any tag containing `-`).

**Prerequisites:** `permissions: contents: write` on the calling job. No checkout needed.

#### Inputs

| Name | Required | Default | Description |
|:-----|:--------:|:--------|:------------|
| `artifact-id` | YES | — | From `build-phar` outputs. Used for immutable download. |
| `plugin-name` | YES | — | From `build-phar` outputs |
| `pm-version` | YES | — | From `build-phar` outputs |
| `body` | NO | `For PocketMine-MP <pm-version>` | Full release body text |

---

### `publish-nightly`

**Type:** Composite action

Force-push the `nightly` tag to the current commit and publish a pre-release with the plugin phar.

**Prerequisites:** `actions/checkout` + `permissions: contents: write` on the calling job.

#### Inputs

| Name | Required | Default | Description |
|:-----|:--------:|:--------|:------------|
| `artifact-id` | YES | — | From `build-phar` outputs |
| `plugin-name` | YES | — | From `build-phar` outputs |
| `plugin-version` | YES | — | From `build-phar` outputs |
| `pm-version` | YES | — | From `build-phar` outputs |
| `body` | NO | Standard nightly summary | Full release body text |
