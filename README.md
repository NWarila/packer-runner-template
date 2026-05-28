# packer-runner-template

[![CI](https://github.com/NWarila/packer-runner-template/actions/workflows/ci.yaml/badge.svg)](https://github.com/NWarila/packer-runner-template/actions/workflows/ci.yaml)
[![Repo Hygiene](https://github.com/NWarila/packer-runner-template/actions/workflows/repo-hygiene.yaml/badge.svg)](https://github.com/NWarila/packer-runner-template/actions/workflows/repo-hygiene.yaml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A template for Packer repos that consume a framework: they own data (the inputs that describe what to build) but not the Packer module itself. Use it to scaffold a new Packer runner repo with CI, drift-gate, security workflows, and release evidence already wired up.

The Packer runner model is intentionally thin — runners own inventory (var files, install templates, build inputs), the framework reusable owns executable build logic. The two are versioned independently and consumers pin to the framework by 40-character SHA.

> Implements the data-only Packer runner pattern defined in [`packer-framework-template` ADR-0005](https://github.com/NWarila/packer-framework-template/blob/main/docs/decision-records/template/0005-define-data-only-packer-runner-repositories.md): the contract, caller workflows, drift-gate, repo-hygiene, and the contract-test fixture suite are wired up and enforced in CI.

## Architecture

This template participates in the three-tier ADR model formalised in [`NWarila/.github` ADR-0001](https://github.com/NWarila/.github/blob/main/docs/decision-records/0001-use-architecture-decision-records.md):

- **Org tier** — ADRs apply to every repo regardless of stack. Mirrored byte-identical from [`NWarila/.github`](https://github.com/NWarila/.github). Drift-gated.
- **Template tier** — ADRs apply to every Packer-runner consumer derived from this template. Master copies live here.
- **Repo tier** — ADRs specific to one consumer repo, in that consumer's `docs/decision-records/repo/`.

## Versioning

Conventional Commits + release-please. Consumers pin to commit SHAs (per the same rule the contract enforces on them) and let Renovate carry pins forward.
