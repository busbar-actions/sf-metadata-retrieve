# busbar-actions/sf-metadata-retrieve

Retrieve Salesforce metadata from an org and write it into the repo as a source-format tree (or as an MDAPI zip).

Built on `busbar_sf_api` for the retrieve, `busbar_sf_types` for the typed metadata model, `sf-mdpkg` to parse the retrieved zip, and `metadata-etl` to decompose into source-format files.

## What it does

Wraps `busbar-sf metadata pull` over a configurable scope (package.xml, type filters, component refs, or "modified since"). Output lands at the chosen target path. If a prior tree exists, the workflow naturally surfaces the diff via git.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `sf-access-token` | yes | — | SF session token (via secret). |
| `sf-instance-url` | yes | — | Org instance URL. |
| `target` | no | `force-app/main/default` | Output directory for the retrieved tree. |
| `format` | no | `source` | `source` decomposes via `metadata-etl`; `mdapi` writes the raw zip. |
| `package-xml` | no | `` | Path to a `package.xml` manifest. When set, overrides the type/component/since filters. |
| `types` | no | `` | Comma-separated metadata types (e.g. `CustomObject,ApexClass,Flow`). |
| `components` | no | `` | Comma-separated `Type:Name` refs. |
| `since` | no | `` | ISO-8601 timestamp. Only include components last modified at or after this. |
| `api-version` | no | `` | Metadata API version. Empty = org default. |
| `emit-manifest` | no | `true` | Write the resolved `package.xml` actually used into `target`. |
| `manifest-path` | no | `package.xml` | Filename inside `target`. |
| `commit` | no | `true` | Commit retrieved metadata back to the branch. |
| `commit-message` | no | `chore(metadata): refresh Salesforce metadata` | |
| `git-user-name` | no | `busbar-bot` | |
| `git-user-email` | no | `bot@busbar.agency` | |
| `comment-pr` | no | `true` | Post diff summary on pull_request events. |
| `upload-artifact` | no | `false` | Upload retrieved metadata as a workflow artifact. |
| `artifact-name` | no | `metadata` | Artifact name. |
| `version` | no | `latest` | `busbar-sf` release tag. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Outputs

| Output | Description |
|---|---|
| `changed` | `"true"` if the pull produced changes vs the prior tree. |
| `target-path` | Echo of the input target path. |
| `manifest-path` | Path to the emitted `package.xml` (empty if `emit-manifest=false`). |
| `component-count` | Number of components retrieved. |

## Example: full source refresh on a manual trigger

```yaml
name: Pull SF Metadata

on:
  workflow_dispatch:
    inputs:
      types:
        description: 'Comma-separated metadata types. Leave blank for everything in scope.'
        required: false
        default: 'CustomObject,ApexClass,ApexTrigger,Flow,LightningComponentBundle,PermissionSet,Profile'

permissions:
  contents: write
  pull-requests: write

jobs:
  pull:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-metadata-retrieve@v1
        with:
          sf-access-token: ${{ secrets.SF_ACCESS_TOKEN }}
          sf-instance-url: ${{ secrets.SF_INSTANCE_URL }}
          types: ${{ inputs.types }}
```

## Example: nightly "modified since" pull

```yaml
name: Nightly Metadata Refresh

on:
  schedule:
    - cron: '0 8 * * *'

permissions:
  contents: write

jobs:
  refresh:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-metadata-retrieve@v1
        with:
          sf-access-token: ${{ secrets.SF_ACCESS_TOKEN }}
          sf-instance-url: ${{ secrets.SF_INSTANCE_URL }}
          since: ${{ github.event.schedule_previous_run_iso || '2025-01-01T00:00:00Z' }}
          commit-message: 'chore(metadata): nightly refresh'
```

## Example: pin to a manifest

```yaml
- uses: busbar-actions/sf-metadata-retrieve@v1
  with:
    sf-access-token: ${{ secrets.SF_ACCESS_TOKEN }}
    sf-instance-url: ${{ secrets.SF_INSTANCE_URL }}
    package-xml: manifests/critical.xml
    target: force-app/main/default
```

## Dependencies (current status)

This action is fully scaffolded but **not yet runnable end-to-end**. Two upstream pieces need to land:

1. **`busbar-sf metadata pull` subcommand** — the existing `busbar-sf` binary has `schema` and `data`; it needs a `metadata pull` family. **Implementation must route through `busbar_sf_api` for all SF API interactions and use `busbar_sf_types` for the typed metadata model** — no rolling your own HTTP, no ad-hoc XML parsing.

   Expected shape:
   ```
   busbar-sf metadata pull \
     --target <dir> \
     [--format source|mdapi] \
     [--package-xml <path>] \
     [--types <Type1,Type2,...>] \
     [--components <Type:Name,...>] \
     [--since <iso8601>] \
     [--api-version <vXX.X>] \
     [--emit-manifest <path>] \
     [--json]
   ```

   Pipeline:
   - Issue a Metadata API `retrieve()` via `busbar_sf_api` with the resolved package.xml.
   - Poll until the result zip is available.
   - Hand the bytes to `MdPackage::from_zip_bytes` (`sf-mdpkg`) for the typed file index.
   - When `--format source`: use `metadata-etl` to decompose `CustomObject`, `Profile`, `PermissionSet`, `Flow`, etc. into per-component / per-child XML files in source-format layout. When `--format mdapi`: just unzip into `target`.
   - Emit a final summary JSON line: `{"component_count": N, "types": [...], "manifest_path": "..."}` when `--json` is set, so the action can parse counts.

2. **Binary publication to `busbar-actions/actions-dist`** — same dependency as the other actions in this org. Build `busbar-sf` for the five target triples and push to `actions-dist` releases under the asset names this action expects.

Once both land, tag this action `v1` and consumers can pin it.
