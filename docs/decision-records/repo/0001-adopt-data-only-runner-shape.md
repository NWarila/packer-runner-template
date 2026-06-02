# ADR-repo/0001: Adopt the Data-Only Runner Shape

| Field          | Value                                   |
| -------------- | --------------------------------------- |
| Status         | Accepted                                |
| Date           | 2026-06-02                              |
| Authors        | Nick Warila (@NWarila)                  |
| Decision-maker | Nick Warila (sole portfolio maintainer) |
| Consulted      | packer-framework-template ADR-template/0005 |
| Informed       | All downstream Packer runner consumers  |
| Reversibility  | Low (changes the fundamental shape of derived repos) |
| Last reviewed  | 2026-06-02                              |

## TL;DR

This template repo itself adopts the data-only runner shape it prescribes to
consumers: no local Packer HCL, no reusable workflows, no build tooling. The
template's value is the contract surface and the seed scaffold, not executable
logic.

## Context and Problem Statement

A template repo can carry two kinds of content:

1. **Seed content** — files a consumer copies on derivation and owns afterward.
2. **Executable content** — scripts, build logic, and reusables that run
   against the template repo itself.

For framework-template repos (which own Packer build logic), both kinds are
appropriate. For runner-template repos (which own the contract and scaffold for
data-only consumers), executable content would contradict the pattern being
taught. A template that itself violates the pattern it enforces undermines
reviewer confidence and creates a confusing derivation model.

## Decision Drivers

1. **Exemplar fidelity.** The template must be a correct example of the pattern
   it propagates. Reviewers of consumer PRs use this template as the reference
   shape.
2. **Contract clarity.** The contract (`packer-runner-template-contract.yaml`)
   forbids local reusables and executable Packer HCL in consumers. Allowing
   them in the template would require contract exemptions and complicate the
   validator.
3. **Thin ownership boundary.** Runners own data; frameworks own logic. Mixing
   concerns in the template weakens the boundary for every downstream runner.

## Decision Outcome

`packer-runner-template` carries no local `reusable-*.yaml` workflow files, no
`packer.pkr.hcl`, and no `Makefile` or `tools/` build harness of its own. It
carries:

- The contract YAML and validator (`contract/`, `tools/`).
- The seed caller workflows (`packer.yaml`, `pr-verify.yaml`,
  `drift-gate.yaml`, `security.yaml`, `repo-hygiene.yaml`).
- The seed inventory scaffold (`packer/repos/`, `packer/fixtures/runtime/`).
- The CI suite that tests the contract validator itself (`tests/fixtures/`).
- Documentation covering the runner protocol, architecture, and ADRs.

## Consequences

### Positive

- The template is a valid input to its own contract validator.
- Reviewers learning the runner pattern see a consistent, executable example.
- No confusion between "template self-tooling" and "consumer seed content".

### Negative

- Template-maintainer changes to build logic (e.g., adding a new validator
  check) require changes in `tools/` only, not in any local build workflow.
  That is a constraint, not a drawback, but it means CI gates on the template
  itself must be exercised via the contract-test harness, not via a local
  framework build.

## Supersedes

None.

## Superseded by

None (current).

## Related ADRs

- [Template ADR-template/0001](../template/0001-pin-packer-and-plugin-versions-exactly.md) —
  pin policy for any local `.pkr.hcl` (the thin shape means there are none).
- `packer-framework-template/docs/decision-records/template/0005-define-data-only-packer-runner-repositories.md` —
  defines the data-only runner pattern this template instantiates.
