# Packer runner protocol

Runner repositories derived from `NWarila/packer-runner-template` call into the framework's reusable build workflow with a pinned `framework_ref` and (usually) a pinned `input_ref`. This document defines the protocol — what runner repositories pass, what they get back, and how the pinning convention stays correct over time.

## Caller signature

A runner repo calls the framework reusable from `.github/workflows/packer.yaml` (the build path) and `.github/workflows/pr-verify.yaml` (the validate-only path):

```yaml
# .github/workflows/packer.yaml — build path
on:
  push:
    branches: [main]
    paths: [packer/**]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  build:
    uses: nwarila-platform/packer-framework-template/.github/workflows/reusable-packer-framework-build.yaml@<framework-sha>
    with:
      # renovate: depName=nwarila-platform/packer-framework-template packageName=nwarila-platform/packer-framework-template currentValue=main
      framework_ref: <framework-sha>
      input_repo: ${{ github.repository }}
      input_ref: ${{ github.sha }}
      mode: build
    secrets: inherit
```

```yaml
# .github/workflows/pr-verify.yaml — validate-only path on PRs
on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  verify:
    uses: nwarila-platform/packer-framework-template/.github/workflows/reusable-packer-framework-build.yaml@<framework-sha>
    with:
      # renovate: depName=nwarila-platform/packer-framework-template packageName=nwarila-platform/packer-framework-template currentValue=main
      framework_ref: <framework-sha>
      input_repo: ${{ github.repository }}
      input_ref: ${{ github.event.pull_request.head.sha }}
      mode: validate
```

The `# renovate: ...` comment lets Renovate's git-refs datasource track the framework's `main` branch and propose bumps as the framework advances.

The `uses:` SHA and the body `framework_ref` MUST move together. The runner contract's content rules enforce that the `uses:` SHA is a 40-character commit SHA; the same SHA SHOULD be reflected in `framework_ref:` so the framework checkout matches the reusable shape.

## Required inputs

| Input | Type | Default | Notes |
|---|---|---|---|
| `framework_ref` | 40-char SHA | (required) | Framework repo commit to check out. Floating refs are rejected before checkout. |
| `input_repo` | `owner/repo` | `${{ github.repository }}` | The runner repo providing inventory. |
| `input_ref` | 40-char SHA | `${{ github.sha }}` | Required to be a SHA unless `allow_floating_input_ref: true`. |
| `allow_floating_input_ref` | bool | `false` | Emergency escape hatch. Runner inventory is trusted input to the framework; pin with the same discipline as `framework_ref`. |
| `mode` | `validate` \| `build` | `validate` | PR-time uses validate; main-branch push uses build. |
| `packer_version` | string | (framework default) | Override the framework's pinned Packer CLI version only when ADR-template/0001 allows. |

## Overlay paths

`overlay_paths` is a newline-separated list of `<input-source>=><framework-destination>` entries. Sources are relative to the input checkout. Destinations are relative to the framework checkout and are allowlisted by the framework to:

- `packer/repos/`
- `packer/fixtures/runtime/`

Examples:

```yaml
overlay_paths: |
  packer/repos => packer/repos
  packer/fixtures/runtime => packer/fixtures/runtime
```

The default overlay matches this template's seed structure; runners that deviate from the seed names declare their own `overlay_paths` explicitly.

## Pin management

Runner repositories should let Renovate update `framework_ref` instead of hand-bumping SHAs. The shared Renovate baseline in this template reads `# renovate:` annotations in workflow YAML using the `git-refs` datasource. Put the annotation directly above the input it manages:

```yaml
with:
  # renovate: depName=nwarila-platform/packer-framework-template packageName=nwarila-platform/packer-framework-template currentValue=main
  framework_ref: 0123456789abcdef0123456789abcdef01234567
```

Keep the reusable workflow `uses:` SHA and the body `framework_ref` under review together. The exact Renovate policy comes from [org ADR-0004](../decision-records/org/0004-use-renovate-for-dependency-updates.md) and the template's `.github/renovate.json5`.

## What the framework provides

In return for the protocol-compliant call, the framework reusable:

- Checks out itself at `framework_ref`.
- Checks out `input_repo` at `input_ref`.
- Applies `overlay_paths` (with path-traversal protection).
- Installs the pinned Packer CLI and OPA versions per [framework ADR-template/0001](https://github.com/NWarila/packer-framework-template/blob/main/docs/decision-records/template/0001-pin-packer-and-plugin-versions-exactly.md).
- Runs `packer init` + syntax validation + `validate-safe` + `inspect`. If `mode: build`, also runs `packer build`.
- Uploads generated artifacts/manifests under retention rules per the framework's release evidence ADR.

## What the runner does NOT provide

Per [packer-framework-template ADR-template/0005](https://github.com/NWarila/packer-framework-template/blob/main/docs/decision-records/template/0005-define-data-only-packer-runner-repositories.md), runner repos do NOT:

- Host their own `packer.pkr.hcl`, `source.pkr.hcl`, or `builds.pkr.hcl` (framework owns these).
- Run `packer build` locally (only via the framework reusable in CI).
- Carry Makefile, `tools/`, `policies/`, or `contract/` directories.
- Host local copies of any non-auto reusable workflow.

## Adopting the protocol

When deriving a real runner from this template:

1. Use this template via GitHub's "Use this template" button or clone manually.
2. Populate `packer/repos/` and `packer/fixtures/runtime/` with the inventory specific to your image domain.
3. Update `.github/workflows/packer.yaml` and `pr-verify.yaml` with the framework SHA pin appropriate for your derivation.
4. Update `.github/CODEOWNERS` for repo-specific owners.
5. Add repo-specific ADRs under `docs/decision-records/repo/`.
6. Open the first PR and verify the contract validator + drift-gate jobs go green.
