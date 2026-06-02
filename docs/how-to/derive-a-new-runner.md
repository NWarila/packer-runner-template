# How to derive a new Packer runner

A Packer runner repo owns inventory — var files, install templates, build
inputs — but not the Packer build logic itself. That logic lives in the
framework. This guide walks you from template to a passing CI run.

## Prerequisites

- Write access to create a new repo in the target org (NWarila or
  nwarila-platform).
- A framework SHA to pin to — either
  `NWarila/packer-framework-template` (credential-free reference) or a
  real deployed framework like `nwarila-platform/proxmox-packer-framework`.
- The Renovate App installed in the org so Renovate can keep pins current.

## Steps

### 1. Create the repo from this template

On GitHub: **Use this template → Create a new repository**.

Name it `<name>-packer-runner` in the appropriate org. Set visibility to
match the framework repo.

### 2. Clone and confirm the scaffold

```sh
git clone https://github.com/<org>/<name>-packer-runner
cd <name>-packer-runner
```

Verify the seed structure is present:

```
packer/repos/
packer/fixtures/runtime/
.github/workflows/packer.yaml
.github/workflows/pr-verify.yaml
.github/workflows/drift-gate.yaml
.github/workflows/security.yaml
.github/workflows/repo-hygiene.yaml
```

### 3. Resolve the framework SHA

Obtain the current `main` HEAD SHA from your chosen framework:

```sh
gh api repos/<org>/<framework-repo>/commits/main --jq .sha
```

### 4. Update the framework pin

In both `.github/workflows/packer.yaml` and `.github/workflows/pr-verify.yaml`,
replace the placeholder SHA with the real 40-character commit SHA in both the
`uses:` line and the `framework_ref:` input. Keep the `# renovate:` annotation
directly above the `framework_ref:` line so Renovate can track and bump it.

The two SHAs in each file must match:

```yaml
jobs:
  build:
    uses: NWarila/packer-framework-template/.github/workflows/reusable-packer-framework-build.yaml@<40-char-sha>
    with:
      # renovate: depName=NWarila/packer-framework-template packageName=NWarila/packer-framework-template currentValue=main
      framework_ref: <40-char-sha>
```

### 5. Populate inventory

Add your image-domain inputs to:

- `packer/repos/` — var files describing which images to build.
- `packer/fixtures/runtime/` — any runtime fixture data consumed by the
  framework build.

Do not add `.pkr.hcl` files unless the runner has a local Packer configuration
need; most runners overlay inputs onto the framework and carry no local HCL.

If you do add local `.pkr.hcl` files, pin every version with `=` per
[template ADR-0001](../decision-records/template/0001-pin-packer-and-plugin-versions-exactly.md).

### 6. Update CODEOWNERS

Edit `.github/CODEOWNERS` to list the real code owners for this runner.

### 7. Add repo-tier ADRs

Create `docs/decision-records/repo/0001-<slug>.md` for any decisions that are
specific to this runner and not already covered by org or template ADRs. See
[org ADR-0001](../decision-records/org/0001-use-architecture-decision-records.md)
for the ADR format.

### 8. Update the drift-gate source-refs

In `.github/workflows/drift-gate.yaml`, verify the `source-ref` values for
`NWarila/.github` and `NWarila/packer-runner-template` are 40-character SHAs.
Replace placeholder SHAs with the current `main` HEAD SHAs of each source repo:

```sh
gh api repos/NWarila/.github/commits/main --jq .sha
gh api repos/NWarila/packer-runner-template/commits/main --jq .sha
```

### 9. Open the first PR and verify CI

Commit and push the inventory + pin updates on a branch, then open a PR against
`main`. The required gates are:

| Check | What it validates |
|---|---|
| CI / contract | Contract shape, SHA-only `uses:`, no forbidden paths |
| Drift gate | Byte-identical org-baseline and template-baseline files |
| Repo hygiene | No `pull_request_target` in runner workflows |
| Security | CodeQL, IaC scan, Scorecard |

All four must pass before merging the initial PR.

## What the runner does NOT own

Per [packer-framework-template ADR-template/0005](https://github.com/NWarila/packer-framework-template/blob/main/docs/decision-records/template/0005-define-data-only-packer-runner-repositories.md):

- No local `reusable-*.yaml` workflow files.
- No `packer build` invocation outside of CI.
- No `Makefile`, `tools/`, `policies/`, or `contract/` directories.

See [runner-protocol.md](../reference/runner-protocol.md) for the full
caller signature and overlay specification.
