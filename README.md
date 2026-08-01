<p align="center">
  <a href="https://github.com/axolotl-pm/pmmp-plugin-actions/actions"><img alt="pmmp-plugin-actions status" src="https://github.com/axolotl-pm/pmmp-plugin-actions/workflows/linter/badge.svg"></a>
</p>

# pmmp-plugin-actions

GitHub Actions utilities for building and releasing [PocketMine-MP](https://github.com/pmmp/PocketMine-MP) plugins.

This collection provides reusable composite actions to build plugin phars, validate release tags, and publish releases or nightly builds.

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
    runs-on: ubuntu-latest
    outputs:
      artifact-name:  ${{ steps.build.outputs.artifact-name }}
      plugin-name:    ${{ steps.build.outputs.plugin-name }}
      plugin-version: ${{ steps.build.outputs.plugin-version }}
      pm-version:     ${{ steps.build.outputs.pm-version }}
    steps:
      - uses: actions/checkout@v4
      - uses: axolotl-pm/pmmp-plugin-actions/build-phar@v1
        id: build
        with:
          php-version: '8.2'
          stage-poggit: true

  publish:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - uses: axolotl-pm/pmmp-plugin-actions/publish-nightly@v1
        with:
          artifact-name:  ${{ needs.build.outputs.artifact-name }}
          plugin-name:    ${{ needs.build.outputs.plugin-name }}
          plugin-version: ${{ needs.build.outputs.plugin-version }}
          pm-version:     ${{ needs.build.outputs.pm-version }}
```

### Releases

Triggered on version tags. Validates that the tag matches `plugin.yml`, builds the phar, and publishes a GitHub release. Tags containing `-` (e.g. `v1.0.0-beta.1`) are automatically marked as pre-releases.

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
      - uses: axolotl-pm/pmmp-plugin-actions/validate-release@v1

  build:
    needs: validate
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.build.outputs.artifact-name }}
      plugin-name:   ${{ steps.build.outputs.plugin-name }}
      pm-version:    ${{ steps.build.outputs.pm-version }}
    steps:
      - uses: actions/checkout@v4
      - uses: axolotl-pm/pmmp-plugin-actions/build-phar@v1
        id: build
        with:
          php-version: '8.2'

  publish:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: axolotl-pm/pmmp-plugin-actions/publish-release@v1
        with:
          artifact-name: ${{ needs.build.outputs.artifact-name }}
          plugin-name:   ${{ needs.build.outputs.plugin-name }}
          pm-version:    ${{ needs.build.outputs.pm-version }}
```

> **Note:** The `outputs:` block in the `build` job is a required bridge — GitHub Actions does not propagate composite action outputs to the parent job automatically.

---

## Reference

### `build-phar`

Set up PHP, build and compress a PocketMine-MP plugin phar, and upload it as a workflow artifact.

**Prerequisites:** `actions/checkout` must run before this action in the same job.

#### Inputs

| Name | Required | Default | Description |
|:-----|:--------:|:--------|:------------|
| `php-version` | YES | — | Any version available in [`pmmp/PHP-Binaries`](https://github.com/pmmp/PHP-Binaries) (e.g. `8.2`, `8.3`) |
| `retention-days` | NO | `1` | Days to retain the uploaded artifact |
| `artifact-name` | NO | `<plugin-name>-phar` | Artifact name override. Useful in matrix builds to avoid collisions. |
| `plugin-dir` | NO | `${{ github.workspace }}` | Directory containing `plugin.yml` and `composer.json` |
| `composer` | NO | `true` | Whether pharynx should include composer dependencies |
| `composer-version` | NO | `2.5.5` | Composer version to use |
| `additional-sources` | NO | — | Additional source directories, colon-separated, absolute or relative to workspace |
| `additional-assets` | NO | — | Additional assets to include (one pathspec per line, works like `git add`) |
| `stage-poggit` | NO | `false` | Whether to stage the build on Poggit CI |

#### Outputs

| Name | Description |
|:-----|:------------|
| `plugin-name` | Value of the `name` field in `plugin.yml` |
| `plugin-version` | Value of the `version` field in `plugin.yml` |
| `pm-version` | PocketMine-MP API version(s) from `plugin.yml` |
| `artifact-name` | Name of the uploaded artifact (use with `actions/download-artifact`) |
| `output-phar` | Absolute path to the compiled phar file |
| `output-dir` | Absolute path to the bundled plugin directory before archiving |

---

### `validate-release`

Verify that the pushed tag matches the version declared in `plugin.yml`. Fails the job if they differ, preventing mismatched releases.

Strips a leading `v` and any pre-release suffix before comparing (e.g. tag `v1.2.0-beta.1` is compared as `1.2.0`).

**Prerequisites:** `actions/checkout` must run before this action in the same job.

No inputs or outputs — this action's only contract is pass/fail.

---

### `publish-release`

Download the plugin phar artifact and publish a GitHub release. Pre-releases are detected automatically from the tag name (any tag containing `-`).

**Prerequisites:** `permissions: contents: write` on the calling job. No checkout needed.

#### Inputs

| Name | Required | Default | Description |
|:-----|:--------:|:--------|:------------|
| `artifact-name` | YES | — | From `build-phar` outputs |
| `plugin-name` | YES | — | From `build-phar` outputs |
| `pm-version` | YES | — | From `build-phar` outputs |
| `body` | NO | `For PocketMine-MP <pm-version>` | Full release body text |

---

### `publish-nightly`

Force-push the `nightly` tag to the current commit and publish a pre-release with the plugin phar.

**Prerequisites:** `actions/checkout` + `permissions: contents: write` on the calling job.

#### Inputs

| Name | Required | Default | Description |
|:-----|:--------:|:--------|:------------|
| `artifact-name` | YES | — | From `build-phar` outputs |
| `plugin-name` | YES | — | From `build-phar` outputs |
| `plugin-version` | YES | — | From `build-phar` outputs |
| `pm-version` | YES | — | From `build-phar` outputs |
| `sha` | NO | `${{ github.sha }}` | Commit SHA to tag as nightly |
| `timestamp` | NO | `${{ github.event.head_commit.timestamp }}` | Build timestamp for the release body |
| `body` | NO | Standard nightly summary | Full release body text |

