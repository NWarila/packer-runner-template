# Mirroring rules

This template participates in a three-tier file-mirroring model defined by [org ADR-0001](../decision-records/org/0001-use-architecture-decision-records.md) and [org ADR-0003](../decision-records/org/0003-use-deny-all-gitignore-strategy.md). Files in this repository fall into four categories.

## 1. Org-baseline mirrors

Byte-identical copies of files owned by [`NWarila/.github`](https://github.com/NWarila/.github). Drift-gated against the org via `.github/workflows/drift-gate.yaml` on every PR.

| File | Mirrored from |
|---|---|
| `CODE_OF_CONDUCT.md` | `NWarila/.github/CODE_OF_CONDUCT.md` |
| `CONTRIBUTING.md` | `NWarila/.github/CONTRIBUTING.md` |
| `SECURITY.md` | `NWarila/.github/SECURITY.md` |
| `SUPPORT.md` | `NWarila/.github/SUPPORT.md` |
| `docs/decision-records/org/000{1..5}-*.md` | `NWarila/.github/docs/decision-records/000{1..5}-*.md` |
| `docs/{tutorials,how-to,reference,explanation}/.gitkeep` | `NWarila/.github/docs/{tutorials,how-to,reference,explanation}/.gitkeep` |

These files MUST NOT be edited directly in this repository. Open the change upstream in [`NWarila/.github`](https://github.com/NWarila/.github), then update the pinned `source-ref` in `.github/workflows/drift-gate.yaml` once the upstream commit lands.

## 2. Template-tier files (this repo owns)

Files this template authors and maintains. Downstream runner repositories adopt these via the `baseline-manifest.json`:

- **Byte-identical** entries: runner repos MUST keep these byte-for-byte identical.
- **Scaffold-starter** entries: runner repos MAY (and usually do) customize these per repo.

| File | Category | Notes |
|---|---|---|
| `.editorconfig` | byte_identical | Editor consistency baseline |
| `.gitattributes` | byte_identical | Text/binary normalization |
| `.markdownlint-cli2.jsonc` | byte_identical | Markdown lint config |
| `.github/PULL_REQUEST_TEMPLATE.md` | byte_identical | Runner-shape PR template |
| `.github/workflows/reusable-auto-merge.yaml` | byte_identical | Privileged `pull_request_target` caller; local mirror needed for static-analyzer call graph |
| `.github/CODEOWNERS` | scaffold_starter | Default `* @NWarila`; runners may add granular owners |
| `.github/renovate.json5` | scaffold_starter | Stub config inheriting this template's preset; runners add repo-specific package rules only |
| `.gitignore` | scaffold_starter | Deny-all allowlist tailored to each runner's actual file set |
| `baseline-manifest.json` | scaffold_starter | Each runner declares its own mirror set |
| `contract/packer-runner-template-contract.yaml` | scaffold_starter | Runners do NOT carry this; the template owns it and runners satisfy it via the validator |

The 5 non-auto reusable workflows (`reusable-codeql.yaml`, `reusable-iac-security.yaml`, `reusable-scorecard.yaml`, `reusable-release-please.yaml`, `reusable-release-evidence.yaml`) are **not** mirrored — runner repos call them remotely via `uses:` and SHA-pin to a specific revision of this template. Renovate keeps those SHAs current.

## 3. Repo-specific files (each consumer owns)

Files that each Packer runner consumer authors itself, derived from this template's seed structure:

- `packer/repos/*.{hcl,yaml,yml,pkrvars.hcl}` — runner inventory
- `packer/fixtures/runtime/**` — public-safe runtime fixtures used by PR-verify
- `.github/workflows/packer.yaml` — build caller pointing at the framework reusable by SHA
- `.github/workflows/pr-verify.yaml` — PR-time caller in validate-only mode
- `.github/workflows/drift-gate.yaml` — org + template baseline verification
- `.github/workflows/security.yaml` — security caller pointing at this template's reusables
- `docs/decision-records/repo/*.md` — repo-specific ADRs

This template's seed copies of these files (under `tests/fixtures/contract/good/...`) demonstrate the canonical shape; runners derive from those rather than from a brand-new GitHub template.

## 4. Generated / state files (never mirrored)

`.terraform/`, `.tmp/`, `packer-build.log`, `packer/manifests/*.json`, etc. The deny-all `.gitignore` keeps these out of the tree by default; they appear only in CI artifact uploads, not in source control.

## Update protocol

| Need to change... | Open the PR against... |
|---|---|
| An org-baseline file | [`NWarila/.github`](https://github.com/NWarila/.github), then bump the drift-gate `source-ref` here |
| A template-tier byte_identical file | this repo; consumers' next drift-gate run will fail and Renovate will propose a sync |
| A template-tier scaffold_starter file | this repo for the template's own copy; each consumer customizes independently |
| A consumer-owned file | the consumer's repo |
