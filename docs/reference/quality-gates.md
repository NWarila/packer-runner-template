# Quality gates

What runs against every PR and `main` push on a `packer-runner-template` repository, and where each gate lives.

## Template self-validation (this repo)

The `.github/workflows/ci.yaml` workflow runs on every PR to `main`:

| Job | What it does |
|---|---|
| `actionlint` | Static analysis of every workflow YAML (catches typos, bad refs, deprecated syntax). |
| `markdownlint` | Markdown lint via `markdownlint-cli2`. |
| `contract validator` | `python tools/check_template_contract.py --type template` validates this template against its own contract; `python tools/run_contract_tests.py` runs the contract harness against `tests/fixtures/contract/{good,bad-*}` to assert the validator emits the expected `[FAIL]` markers. |
| `mypy (advisory)` | Static type analysis of `tools/`. Emits `::warning::` annotations on findings; never blocks merge. |

In addition, the `Drift Gate` workflow runs `NWarila/drift-gate` as a SHA-pinned composite action verifying that the org-baseline mirrors in this template still byte-match upstream `NWarila/.github`.

## Runner consumer gates

Each runner repo derived from this template inherits the following CI surface:

| Workflow | Triggered by | What it does |
|---|---|---|
| `.github/workflows/pr-verify.yaml` | `pull_request` | Calls `NWarila/packer-framework-template/.github/workflows/reusable-packer-framework-build.yaml@<sha>` with `build: false`. The framework checks out itself, overlays the runner's inputs, runs `packer init` + `validate-safe` + `inspect`. |
| `.github/workflows/packer.yaml` | `push: main, paths: packer/**` and `workflow_dispatch` | Same as above with `build: true`. Adds the real `packer build` step and uploads manifests as a CI artifact. |
| `.github/workflows/drift-gate.yaml` | `pull_request` | Verifies the runner's mirrored org-baseline files (and template-baseline files) byte-match upstream `NWarila/.github@<sha>` and `NWarila/packer-runner-template@<sha>`. |
| `.github/workflows/security.yaml` | `pull_request`, scheduled, `branch_protection_rule` | Calls this template's reusable security suite: CodeQL (Actions language), Trivy (filesystem + secrets), Gitleaks (history), zizmor (Actions injection), OpenSSF Scorecard (non-PR runs). |
| `.github/workflows/release.yaml` (optional) | `push: main`, `release: published`, `workflow_dispatch` | Thin caller: release-please from `NWarila/.github`, and release evidence from the Packer framework's `reusable-release-evidence.yaml@<sha>` with `repo_type: runner`. Runners own no local reusable. Only added when the runner publishes versioned releases. |

## Required statuses

A runner's branch-protection rules on `main` should require at minimum:

- `org-baseline / verify`
- `runner-template / verify`
- `pr-verify` (or whatever the framework-build caller's check is named)
- `IaC and secret scan / Gitleaks (secret scan)`
- `IaC and secret scan / Trivy (filesystem & secrets)`
- `IaC and secret scan / zizmor (Actions security)`
- `CodeQL / CodeQL (actions)`

Renovate-authored PRs that pass all required checks are auto-mergeable by adding a thin `auto-merge.yaml` caller that `uses:` the org-owned `NWarila/.github/.github/workflows/reusable-auto-merge.yaml@<sha>` by SHA. Runner repos own **no** local reusable workflows (the contract `forbidden_paths` reject every `.github/workflows/reusable-*.yaml`), so the privileged `reusable-auto-merge.yaml` is never mirrored locally. Its `pull_request_target` safety is enforced by the org `repo_hygiene` policy via the `repo-hygiene.yaml` caller — which denies PR-head reads in that workflow — not by keeping the reusable local for static-analyzer call-graph visibility.

## How to add a new gate

1. Author the reusable workflow in the layer that owns it — universal gates in `NWarila/.github`, type-specific gates in the framework template (`NWarila/packer-framework-template`). Runner repos own **no** local `reusable-*.yaml` (the contract `forbidden_paths` reject them); this template adds only a thin caller that `uses:` the owning reusable by SHA.
2. Add a `content_rule` in `contract/packer-runner-template-contract.yaml` requiring runner consumers to call it by SHA.
3. Add the workflow to the `template` type's `required_paths` in the contract so template-mode self-validation enforces presence.
4. Add a negative fixture under `tests/fixtures/contract/bad-<descriptor>/` that exercises the new rule.
5. Update `tools/run_contract_tests.py`'s `EXPECTED_BAD_CONTRACT_FAILURES` with the expected `[FAIL]` marker.
6. Open the PR. The contract harness will assert the new fixture fails the new rule with the expected marker.
