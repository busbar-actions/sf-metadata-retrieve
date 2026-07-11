> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-metadata-retrieve

Retrieve Salesforce metadata from an org via the Metadata API and write it into the repo, then optionally commit the result back to the branch and comment a summary on the PR.

Backed by the `sf-metadata-retrieve` Rust binary (published to `busbar-actions/actions-dist`), which authenticates through `busbar-auth`, retrieves through `busbar_sf_api::metadata`, and unzips the returned package with `sf-mdpkg`.

## What it does

1. Authenticates: **by default it self-mints** — with no `sf-access-token` supplied it exchanges the runner's GitHub OIDC id-token in-process for a short-lived Salesforce session against the Busbar-equipped org at `target-instance`, and revokes + zeroizes it (`session.dispose()`) at exit. If you instead pass `sf-access-token` + `sf-instance-url`, it uses them directly and skips OIDC (see Auth model below).
2. Builds a `package.xml` from one of `package-xml` (a manifest file), `components` (`Type:Name` refs), or `types` (wildcard members per type).
3. Issues a Metadata API `retrieve()` and polls until the result zip is ready (600s timeout, 5s poll).
4. Unzips the package and writes every entry into `target` in **MDAPI layout** (verbatim file tree from the retrieve zip).
5. Optionally writes a manifest of the retrieved components as JSON (see `emit-manifest` note).
6. Computes `file-count` / `component-count`, writes `GITHUB_OUTPUT`, a job summary, a notice annotation, and a PR-comment body.
7. Optionally commits the written tree back to the current branch and pushes it; emits `changed`.

> Note on output layout: the binary always writes the raw MDAPI tree (verbatim from the retrieve zip). A decomposed source-format path is not implemented.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `target-instance` | yes* | `` | Busbar-equipped org instance URL for the OIDC self-mint (e.g. `https://acme.my.salesforce.com`). *Required unless you supply the `sf-access-token`/`sf-instance-url` override. Requires `permissions: id-token: write`. |
| `eca-client-id` | no | `` | Override the External Client App consumer key for the OIDC exchange. PBO-pinned default; set only on a rotation. |
| `token-handler` | no | `` | Override the Apex token-exchange handler dev name. Defaults to `GitHubTokenExchangeHandler`. |
| `oidc-audience` | no | `` | Override the audience requested in the GitHub OIDC token. Defaults to `target-instance`. |
| `sf-access-token` | no | `` | OPTIONAL local-dev/advanced override: a handed-off Salesforce session access token. When set (with `sf-instance-url`) the binary uses it directly and skips OIDC. |
| `sf-instance-url` | no | `` | OPTIONAL override paired with `sf-access-token`, e.g. `https://yourdomain.my.salesforce.com`. |
| `target` | no | `force-app/main/default` | Output directory for the retrieved tree (relative to repo root). |
| `package-xml` | no | `` | Path to a `package.xml` manifest. When set, it overrides the `types`/`components` filters. |
| `types` | no | `` | Comma-separated metadata types retrieved with wildcard (`*`) members (e.g. `CustomObject,ApexClass,Flow`). |
| `components` | no | `` | Comma-separated `Type:Name` component refs (e.g. `CustomObject:Account,ApexClass:OrderUtil`). |
| `api-version` | no | `` | Metadata API version (e.g. `v62.0`). Empty resolves to the binary default (`v62.0`). |
| `emit-manifest` | no | `true` | When true, write a manifest of the retrieved components into `target`. **The emitted manifest is a JSON dump of the retrieve file properties, not a `package.xml`.** |
| `manifest-path` | no | `package.xml` | Relative-to-`target` filename for the emitted manifest. |
| `commit` | no | `true` | Commit (and push) the retrieved tree back to the current branch. |
| `commit-message` | no | `chore(metadata): refresh Salesforce metadata` | Commit message when `commit=true`. |
| `git-user-name` | no | `busbar-bot` | git `user.name` for the commit. |
| `git-user-email` | no | `bot@busbar.agency` | git `user.email` for the commit. |
| `comment-pr` | no | `true` | Post a summary comment on `pull_request` events. |
| `upload-artifact` | no | `false` | Upload the retrieved tree as a workflow artifact. |
| `artifact-name` | no | `metadata` | Artifact name when `upload-artifact=true`. |
| `version` | no | `latest` | `sf-metadata-retrieve` release tag to download. `latest` resolves the most recent release. |
| `binary-repo` | no | `busbar-actions/actions-dist` | GitHub repo publishing the binary releases. |

> One value flows through env that is not a declared input: the action sets `INPUT_COMMENT_OUTPUT` to `${RUNNER_TEMP}/metadata-comment.md`, the file the binary writes the PR-comment body into and the "Post PR comment" step reads back.

## Outputs

| Output | Description |
|---|---|
| `changed` | `"true"` if the retrieve produced changes vs the prior tree (a real commit when `commit=true`, otherwise a dirty-tree check). |
| `target-path` | Echo of the `target` input. |
| `manifest-path` | Path to the emitted manifest (empty if `emit-manifest=false`). |
| `component-count` | Number of components retrieved. |
| `file-count` | Number of files written to the target tree. |

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
  contents: write       # commit the retrieved tree back
  pull-requests: write  # post the PR comment
  id-token: write       # OIDC self-mint (the default auth path)

jobs:
  pull:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-metadata-retrieve@v1
        with:
          target-instance: https://acme.my.salesforce.com
          types: ${{ inputs.types }}
```

## Example: pin to a manifest

Both examples below need `permissions: id-token: write` on the job for the default OIDC self-mint.

```yaml
- uses: busbar-actions/sf-metadata-retrieve@v1
  with:
    target-instance: https://acme.my.salesforce.com
    package-xml: manifests/critical.xml
    target: force-app/main/default
```

## Example: retrieve specific components, no commit

```yaml
- uses: busbar-actions/sf-metadata-retrieve@v1
  with:
    target-instance: https://acme.my.salesforce.com
    components: 'CustomObject:Account,ApexClass:OrderUtil'
    commit: 'false'
    upload-artifact: 'true'
```

## Auth & permissions model

**Default (recommended): GitHub OIDC self-mint — zero stored SF credentials.** A Salesforce access token is never handed to a script, written to `GITHUB_ENV`, passed as an input/output, or persisted. With no `sf-access-token` supplied, the binary fetches the runner's OIDC id-token and exchanges it at the Busbar-equipped org's Apex token handler (against `target-instance`) for a short-lived session, holds it only in zeroizing memory (`busbar-auth` `CredentialContext`), uses it, and **revokes + zeroizes it (`session.dispose()`) at exit** because the token is OIDC-minted-and-owned. The org must trust this repo (`sf busbar trust request approve`); the first run from a new repo/workflow stops with a pending-trust error until approved.

```yaml
permissions:
  contents: write       # required when commit=true (the binary commits and pushes)
  pull-requests: write  # required when comment-pr=true (gh pr comment)
  id-token: write       # required for the OIDC token exchange (the default auth path)
```

**Optional override (local-dev / advanced): handed-off token.** Set `sf-access-token` + `sf-instance-url` and `busbar-auth` uses them as-is, skipping OIDC. A handed-off token is NOT OIDC-minted, so the binary does not revoke it — it only zeroizes its in-memory copy on drop; its lifecycle is the caller's responsibility. Do not use this path in CI when OIDC is available.

The token is never written to outputs and is masked by the reporter; the binary zeroizes the credential context on drop.

> Cleanup caveat: revoke/zeroize happen via `dispose()` only on the success path. If the retrieve fails, the binary exits without disposing the session. Because the session is OIDC-minted-and-owned (not handed off via `GITHUB_ENV`), a failed run currently leaves the short-lived token to expire on its own rather than being explicitly revoked. As a composite action there is no `post:` hook to guarantee revocation; a guaranteed-cleanup story would require disposing on the error path inside the binary (or converting to a JS action with a `post` script).

## Observability

The binary renders its outputs, job summary, and a notice annotation through the shared `github-actions-ux` `Reporter`, which auto-selects GitHub workflow commands on a runner and plain output locally. Progress lines (`Auth: …`, `Retrieving metadata…`, counts) and the final JSON summary are written to stderr/stdout directly rather than through the reporter, and the binary fails via `fail()` (annotated `::error::` + exit 1) rather than the newer `run_outcome` + `RecordingReporter` path, so a failure is annotated but does not always land a structured error in the Job Summary.
