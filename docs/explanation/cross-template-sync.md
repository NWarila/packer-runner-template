# Cross-template-tier sync

This template shares structural patterns with other runner templates in the portfolio (initially [`NWarila/terraform-runner-template`](https://github.com/NWarila/terraform-runner-template); future templates for additional stacks will follow the same pattern). Sections of code and configuration are intentionally identical across templates so a maintainer who learns one template can navigate any of them, and so improvements to the validator harness propagate uniformly.

This document records which parts of the template are **shape-shared** (must move together), which differences are **intentional** (one-of-a-kind to each template), and when to consider automating the sync.

## Files that are shape-shared

The following files are functionally near-identical across runner templates. When you change one, audit the others for the same change.

| File | Identical sections | Intentional differences |
|---|---|---|
| `tools/check_template_contract.py` | ~95% of the file — the entire rule walker, `RuleResult` dataclass, sync_drift check, argparse plumbing, validator main loop | `inventory_files()` walks stack-specific paths (`packer/repos/` here, `terraform/public/` elsewhere); default `--contract` path; module docstring's filename reference |
| `tools/run_contract_tests.py` | Harness skeleton, `Fixture` dataclass, fixture discovery, marker-matching logic, malformed-contract test | `EXPECTED_BAD_CONTRACT_FAILURES` (per-template `[FAIL]` markers); `prune_template_only_runner_paths()`'s pruned-paths set; default `--contract` path |
| `contract/<name>-contract.yaml` | Universal section structure (`required_root_files`, `required_github_files`, `required_documentation`); `workflow_pinning` schema; `forbidden_paths`/`content_rules` shape | Contract's name; runner-tier `required_paths` set (stack-specific); content_rule regexes (reference different reusable filenames per stack) |
| `baseline-manifest.json` | v2 schema (`byte_identical` + `scaffold_starter` arrays of `{source, target}`) | The list contents (each template mirrors a different baseline set) |
| `docs/explanation/architecture.md` + `testing-strategy.md` + `threat-model.md` | Layering diagram conventions, Diátaxis structure | Stack-specific content (Packer terms here, Terraform terms elsewhere) |
| `docs/decision-records/org/0001..0005-*.md` | **Byte-identical** | None — these are org-baseline mirrors and the drift gate enforces byte identity |

## Why this isn't automated yet

A `template-sync.yaml` workflow that periodically PRs the shared portions across templates is feasible but not justified at current portfolio size:

- **2 runner templates today** (`terraform-runner-template`, `packer-runner-template`). Human-mediated sync via this doc is sufficient — the marginal cost of "check the sibling when you touch a shape-shared file" is small.
- **Automation pays off at ~3+ templates**, where the human cost crosses the cost of building (and maintaining) the sync workflow itself.

A prior attempt to host the sync workflow in a `terraform-template-template` meta-repo was abandoned and the meta-repo was archived; resuming that path is non-trivial and out of scope until needed.

## Maintainer convention

When you change a shape-shared file in this template, follow this loop:

1. **Open the change in this template first.** Land the PR locally, get CI green.
2. **Audit the corresponding file in each sibling template** ([`NWarila/terraform-runner-template`](https://github.com/NWarila/terraform-runner-template) and any future siblings).
3. **For each sibling that needs the same change**, open a follow-up PR that mirrors the relevant lines. Keep the diff narrow — change only what genuinely needs to be in lockstep.
4. **Reference this template's PR in the sibling PR** so the linkage is visible in the git log.

Symmetrically, if you find a sibling has improved its validator/harness in a way this template should adopt, the same loop applies in the other direction.

## When to revisit this convention

Consider automation when any of these become true:

- A third runner template is being authored
- A shape-shared file accumulates >1 unintended divergence in 6 months
- A maintainer reports the manual loop is missing real updates

At that point, the right move is likely to publish the validator core as a separate Python package (`nwarila-runner-contract-tools`) that both templates depend on, or to host a `template-sync.yaml` workflow in a meta-repo that opens PRs in each template when shape-shared files diverge. Both are real engineering projects, not weekend work.

## What is explicitly NOT shape-shared

To avoid false expectations:

- `.github/workflows/` callers (the ones that consumers customize per-runner)
- `README.md` (each template owns its narrative)
- `.gitignore` (each template's deny-all allowlist reflects its own surface)
- `tests/fixtures/contract/*` (each template tests its own contract rules)
- `tools/` beyond the validator + harness (this template currently has only those two; the Terraform template has a larger toolchain that is template-specific)
- `packer/` or `terraform/` directories themselves (stack-specific by definition)

If a maintainer mistakes one of these for shape-shared and tries to mirror it, they should stop and consult this doc.
