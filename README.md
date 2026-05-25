# packer-runner-template

A template for Packer repos that consume a framework: they own data (the inputs that describe what to build) but not the Packer module itself. Use it to scaffold a new Packer runner repo with CI, drift-gate, security workflows, and release evidence already wired up.

The Packer runner model is intentionally thin — runners own inventory (var files, install templates, build inputs), the framework reusable owns executable build logic. The two are versioned independently and consumers pin to the framework by 40-character SHA.

> **Status: scaffold in progress.** This repo's contract, reusable workflows, fixtures, and ADRs are landing across phased PRs. Once complete it will satisfy the conditions in [`packer-framework-template/docs/decision-records/template/0005-define-data-only-packer-runner-repositories.md`](https://github.com/NWarila/packer-framework-template/blob/main/docs/decision-records/template/0005-define-data-only-packer-runner-repositories.md) for deriving real Packer runner consumers.

## Architecture

This template participates in the three-tier ADR model formalised in [`NWarila/.github` ADR-0001](https://github.com/NWarila/.github/blob/main/docs/decision-records/0001-use-architecture-decision-records.md):

- **Org tier** — ADRs apply to every repo regardless of stack. Mirrored byte-identical from [`NWarila/.github`](https://github.com/NWarila/.github). Drift-gated.
- **Template tier** — ADRs apply to every Packer-runner consumer derived from this template. Master copies live here.
- **Repo tier** — ADRs specific to one consumer repo, in that consumer's `docs/decision-records/repo/`.

## Versioning

Conventional Commits + release-please. Consumers pin to commit SHAs (per the same rule the contract enforces on them) and let Renovate carry pins forward.
