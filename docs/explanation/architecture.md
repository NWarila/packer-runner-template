# Architecture

## Template boundary

`packer-runner-template` is the type-template for Packer-runner repos. It owns:

- The contract every Packer-runner repo must satisfy ([`contract/packer-runner-template-contract.yaml`](../../contract/packer-runner-template-contract.yaml)).
- The six stack-agnostic reusable workflows that consumer runners call by SHA — CodeQL, IaC security (Trivy + Gitleaks + zizmor), Scorecard, release-please, release-evidence, auto-merge.
- The canonical runner scaffold consumers adopt: shared dotfiles, OPA policy hooks (when added later), drift-gate layout, baseline-manifest schema.
- The seed runner data and integration fixture that new runner repos inherit (`packer/repos/`, `packer/fixtures/runtime/`, `tests/fixtures/`).
- The template-tier baseline manifest that every Packer-runner consumer uses to mirror standardized scaffold files and template ADRs.

It does NOT own:

- An executable Packer module of its own. Runners are data-only deployers: `packer build` runs against the framework's tree with the runner's `packer/repos/` inventory overlaid onto the framework's runtime path.
- The `reusable-packer-framework-build.yaml` workflow. That lives in [`nwarila-platform/packer-framework-template/.github/workflows/`](https://github.com/NWarila/packer-framework-template/blob/main/.github/workflows/reusable-packer-framework-build.yaml). The contract's content_rule explicitly allows either `NWarila` or `nwarila-platform` as the host org for that reusable, so framework-tier and template-tier callers both satisfy the contract.

## Inputs and outputs

A consumer runner pins to a SHA of this template. On adoption, the consumer:

- Inherits the standardized scaffold from `baseline-manifest.json`, then pins:
  - The framework `uses:` SHA in `pr-verify.yaml` and `packer.yaml`.
  - The framework `framework_ref:` input (kept in lockstep with the `uses:` SHA).
  - The drift-gate `source-ref` for both `NWarila/.github` and `NWarila/packer-runner-template`.
- Populates `packer/repos/` and `packer/fixtures/runtime/` with the inventory specific to its image domain.

Renovate keeps the framework `uses:` SHA and the body `framework_ref` in lockstep with the framework's `main` branch. Renovate keeps the drift-gate source-refs in lockstep with the upstream's `main` branches.

## External dependencies

- [`NWarila/.github`](https://github.com/NWarila/.github) — provides org-baseline policy files, ADR masters, and documentation layout sentinels. Mirrored into this template and every consumer runner; verified by [`NWarila/drift-gate`](https://github.com/NWarila/drift-gate) on every PR.
- The framework being consumed at deploy time, typically [`NWarila/packer-framework-template`](https://github.com/NWarila/packer-framework-template) (the credential-free reference framework) or a real framework like [`nwarila-platform/proxmox-packer-framework`](https://github.com/nwarila-platform/proxmox-packer-framework). The framework is outside this template's scope; consumer runners pin to it directly via `framework_ref`.
- [`NWarila/drift-gate`](https://github.com/NWarila/drift-gate) — composite GitHub Action invoked by every consumer's `drift-gate.yaml` workflow. SHA-pinned per ADR-0004's workflow-pinning rule.

## Layering at a glance

```text
┌─────────────────────────────────────────────────────┐
│ NWarila/.github                                     │
│   • org ADRs 0001-0005                              │
│   • CODE_OF_CONDUCT, CONTRIBUTING, SECURITY, ...    │
│   • baseline-manifest.json                          │
└─────────────────┬───────────────────────────────────┘
                  │ drift-gate composite action
                  ▼
┌─────────────────────────────────────────────────────┐
│ NWarila/packer-runner-template (THIS REPO)          │
│   • template ADRs (pinning, runner shape, ...)      │
│   • 6 reusable workflows (codeql, iac-security, ...)│
│   • contract YAML + validator + harness + fixtures  │
│   • seed scaffold for runner repos                  │
└─────────────────┬───────────────────────────────────┘
                  │ uses: ...@<template-sha>
                  │ baseline-manifest.json drift-gate
                  ▼
┌─────────────────────────────────────────────────────┐
│ NWarila/<name>-packer-runner (consumer)             │
│   • packer/repos/   ← inventory (data only)         │
│   • packer/fixtures/runtime/                        │
│   • .github/workflows/{packer,pr-verify,           │
│     drift-gate,security,release}.yaml               │
│   • docs/decision-records/repo/                     │
└─────────────────┬───────────────────────────────────┘
                  │ uses: ...@<framework-sha>
                  ▼
┌─────────────────────────────────────────────────────┐
│ nwarila-platform/packer-framework-template          │
│   • reusable-packer-framework-build.yaml            │
│   • packer/packer.pkr.hcl (CLI + plugin pins)       │
│   • packer/source.pkr.hcl, builds.pkr.hcl, ...      │
└─────────────────────────────────────────────────────┘
```

## Why thin runners

Per [`packer-framework-template/docs/decision-records/template/0005-define-data-only-packer-runner-repositories.md`](https://github.com/NWarila/packer-framework-template/blob/main/docs/decision-records/template/0005-define-data-only-packer-runner-repositories.md):

- **Thin ownership boundary.** Runners own input data and orchestration, not framework internals.
- **Reviewer clarity.** A reviewer can tell at a glance whether a PR is changing inputs (low-risk) or framework behavior (high-risk).
- **Automation-first evidence.** Confidence comes from GitHub PR validation, drift gates, security summaries, and release evidence — not from "I tested it locally".
- **Derivation safety.** Each new runner derives from a contract, not from broad copy-paste of a previous runner.
- **Consistency with Terraform runners.** Same pattern as `terraform-runner-template`; reviewers don't context-switch between stacks.
