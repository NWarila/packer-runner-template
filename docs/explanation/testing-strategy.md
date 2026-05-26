# Testing strategy

What is tested at each layer, what is intentionally NOT tested, and where each gate runs.

## What the contract tests (this template's CI)

The contract layer is the strictest gate in the template and the one downstream Packer runner consumers inherit by satisfying it. Three jobs run on every PR to `main`:

| Job | What it asserts |
|---|---|
| `actionlint` | Every workflow YAML in `.github/workflows/` is syntactically valid GitHub Actions. Catches typos, invalid refs, deprecated syntax. |
| `markdownlint` | Every Markdown file is well-formed per the org's shared `.markdownlint-cli2.jsonc` rules. |
| `contract validator` | Two sub-steps: (1) `python tools/check_template_contract.py --type template` validates this template against its own contract; (2) `python tools/run_contract_tests.py` runs every fixture in `tests/fixtures/contract/` and asserts the validator emits the expected `[FAIL]` markers for negative fixtures and exits cleanly on `good/`. |
| `mypy (advisory)` | Static type analysis of `tools/`. Emits `::warning::` annotations on findings; **never** blocks merge. Surfaces regressions without forcing churn on existing stylistic patterns. |

In addition, the `Drift Gate` workflow runs `NWarila/drift-gate@<sha>` as a SHA-pinned composite action that verifies the org-baseline mirrors in this template still byte-match `NWarila/.github` at the pinned `source-ref`.

## What the contract harness tests, specifically

`tools/run_contract_tests.py` iterates `tests/fixtures/contract/{good,bad-*}/`:

- `good/` — the positive fixture. Overlay onto a pruned template tree, run `check_template_contract.py --type runner`, expect exit 0.
- `bad-build-missing-framework-reusable/` — `packer.yaml` references no `reusable-packer-framework-build.yaml@<sha>` → contract rule fires → expected `[FAIL]` marker present.
- `bad-drift-gate-missing-org-source/` — drift-gate has only template source → org-baseline content rule fires.
- `bad-drift-gate-missing-template-source/` — drift-gate has only org source → template-baseline content rule fires.
- `bad-pr-verify-manual-only/` — `pr-verify.yaml` missing `pull_request:` trigger → trigger content rule fires.
- `bad-local-reusable-workflow/` — runner has `reusable-codeql.yaml` locally → `forbidden_paths` rule fires.
- `bad-security-uses-local-codeql/` — `security.yaml` uses `./reusable-codeql.yaml` instead of the template-owned remote → `forbidden_pattern present` rule fires.
- `bad-renovate-org-baseline/` — `renovate.json5` extends `github>NWarila/.github` → forbidden pattern fires.

Each fixture's expected marker is pinned in `EXPECTED_BAD_CONTRACT_FAILURES`, so a regression in the validator's failure-message format would itself fail the test.

`tools/run_contract_tests.py` also runs a `malformed-forbidden-path` check that copies the contract YAML, injects a malformed `forbidden_paths` entry (a `name:` key in place of `path:`/`glob:`), and confirms the validator rejects it with the `forbidden:<malformed-entry>` marker.

## What downstream runners verify (CI surface inherited from this template)

Each runner repo derived from this template gets:

| Workflow | What it tests |
|---|---|
| `.github/workflows/pr-verify.yaml` | Calls the framework's `reusable-packer-framework-build.yaml@<sha>` in `mode: validate`. Framework checks out itself + the runner's inputs, overlays into `packer/repos/`, runs `packer init` + syntax validate + `validate-safe` + `inspect`. |
| `.github/workflows/packer.yaml` | Same as above in `mode: build`. Adds the real `packer build`. Triggered by `push: main, paths: packer/**` and `workflow_dispatch`. |
| `.github/workflows/drift-gate.yaml` | Mirrors verified byte-identical against both `NWarila/.github` and `NWarila/packer-runner-template` baseline manifests. |
| `.github/workflows/security.yaml` | This template's reusables: CodeQL, Trivy + Gitleaks + zizmor, OpenSSF Scorecard. |

## What is deliberately NOT tested here

| Out of scope | Why | Where it lives |
|---|---|---|
| Real Packer plugin downloads | The contract layer is static — no Packer CLI runs at template-CI time. | Framework `Validate & Test` workflow exercises real `packer init` against pinned plugins. |
| `packer build` execution against real builders (Proxmox, AWS, vSphere) | Build infrastructure isn't reachable from public runners; would require credentials. | Real runner consumers run `packer build` on self-hosted runners in their `packer.yaml`. |
| Cross-repo end-to-end deploy | Requires real GitHub + framework + runner + inventory + S3 state | Runner consumers' CI does this on every `terraform/**`-touching push. |
| Application of OPA policies to built artifacts | This template doesn't ship OPA policies (per ADR-template/0005's data-only thin-runner shape). | Framework owns OPA policy + evaluation. |
| terraform-provider-github bug fixes | Provider behavior under `allow_forks` etc. is upstream concern. | Framework documents the workarounds; consumers benefit transitively via pin bumps. |

## What's covered by the framework, not this template

The Packer-build orchestration lives in `nwarila-platform/packer-framework-template/.github/workflows/reusable-packer-framework-build.yaml`. That workflow's tests (under `framework/terraform/tests/` if applicable, or under the framework's own `.github/workflows/`) cover:

- `packer init` against the pinned plugin SHAs
- `packer validate` + `packer inspect`
- `packer build` against the framework's credential-free reference source
- Provider-specific path resolution and pkrvars merging

This template's contract assumes that framework testing is sound. If the framework breaks, downstream runners fail at their `pr-verify` job — not at this template's CI.

## Adding a new test

Per the protocol in `docs/reference/quality-gates.md`:

1. Author the workflow / contract rule / validator addition.
2. If a new contract rule: add a negative fixture under `tests/fixtures/contract/bad-<descriptor>/` exercising it.
3. Add the expected `[FAIL]` marker to `EXPECTED_BAD_CONTRACT_FAILURES` in `tools/run_contract_tests.py`.
4. Open the PR. The harness asserts the new fixture fails the new rule with the expected marker.

This is the same loop used to land the 8 contract fixtures currently in place.
