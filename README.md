# Workflows

Shared GitHub Actions workflows for the Apollo Automation product repos.

## build-beta.yml

Builds a product's firmware from its beta branch and publishes it to a rolling
`beta-fw` pre-release. Devices on the Beta firmware channel read their OTA
manifest from that release, so the asset names are a device contract.

### Inputs

| Input | Default | Meaning |
|---|---|---|
| `variants` | required | JSON array of `{"key", "yaml", "name"}`, one per beta image. `key` is the manifest key: `manifest-<key>.json` with bins named `<key>-r<run>.<attempt>-*.bin`. `key: ""` gives plain `manifest.json` with bins named `r<run>.<attempt>-*.bin` (single-image repos). `yaml` is the file to build. `name` is the build artifact name; it must start with `firmware` and is not derived from `key`, so give it explicitly. Keys must match `ota_beta_manifest` in the device YAML. |
| `esphome-version` | `stable` | ESPHome version to build with. |
| `core-yaml-path` | `Integrations/ESPHome/Core.yaml` | File whose `substitutions: version:` is the beta version. |
| `firmware-paths` | `Integrations/ESPHome/` and `.github/workflows/build-beta.yml` | Newline list of paths that count as a firmware change. An entry ending in `/` matches everything under it; any other entry matches one file. Used by the newer-commit check below. |
| `beta-branch` | `beta` | The only branch that publishes. |
| `release-tag` | `beta-fw` | Release and tag to publish to. Change it only for a dry run. |
| `skip-non-beta-dispatch` | `false` | When true, a run that is not on `beta-branch` skips everything, builds included. When false it builds and then skips publishing. |

The product name in the release notes comes from the repository name.

### What it guarantees

- Only `beta-branch` creates the release, uploads, cleans up or moves the tag.
- It publishes only if no newer commit on `beta-branch` changes `firmware-paths`
  (renames count under both names). If the compare API's 300-file list may be
  hiding such a change, or `beta-branch` does not contain the commit, it skips.
  A re-run of an old build can never roll beta back.
- Each run uploads under its own names (`r<run>.<attempt>`), bins before
  manifests, and a failed upload fails the run.
- Old bins are deleted only when every live manifest was read in full, and the
  bins the previous manifests point at are kept for one more build.
- The release tag is moved to the built commit, with the same newer-commit check.

### Caller

The caller owns the triggers and the concurrency group, and must grant
`contents: write`. Shared-workflow changes do not trigger product rebuilds;
dispatch the caller on beta after one.

```yaml
name: Build and Publish Beta

on:
  push:
    branches: [beta]
    paths:
      - 'Integrations/ESPHome/**'
      # A push that only changes this file must rebuild too
      - '.github/workflows/build-beta.yml'
  workflow_dispatch:

permissions:
  contents: read

# One beta build per branch at a time; a newer push waits instead of
# cancelling a build mid-upload.
concurrency:
  group: build-beta-${{ github.ref }}
  cancel-in-progress: false

jobs:
  beta:
    permissions:
      contents: write
    uses: ApolloAutomation/Workflows/.github/workflows/build-beta.yml@main
    with:
      variants: >-
        [{"key": "standard", "yaml": "Integrations/ESPHome/AIR-1.yaml",     "name": "firmware-standard"},
         {"key": "ble",      "yaml": "Integrations/ESPHome/AIR-1_BLE.yaml", "name": "firmware-ble-beta"}]
```

A single-image repo that devices read as plain `manifest.json`:

```yaml
    with:
      variants: '[{"key": "", "yaml": "Integrations/ESPHome/CAST-1.yaml", "name": "firmware"}]'
```
