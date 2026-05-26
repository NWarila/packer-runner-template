# Threat model

What this template defends against, what it deliberately accepts as out of scope, and where each mitigation lives.

## Scope of this document

The threat model covers the **template tier** and the **runner consumer tier**. Threats at the framework tier (executable Packer build logic, plugin trust, image cryptographic provenance) live in [`packer-framework-template`'s threat model](https://github.com/NWarila/packer-framework-template/blob/main/docs/explanation/threat-model.md). Threats at the org tier (community-health policy, branch protection, secret-push-protection) live in [`NWarila/.github`'s DESIGN.md](https://github.com/NWarila/.github/blob/main/DESIGN.md).

A runner consumer is a leaf node — its threat surface combines this template's threats with the framework's threats and the org's threats. Each tier owns its own mitigations.

## Threats addressed

### T1 — Compromised third-party GitHub Action

**Scenario:** A maintainer of a third-party Action (or an attacker who compromised that maintainer's account) ships a malicious release. A repo that uses `uses: someorg/some-action@v4` automatically picks up the malicious version on the next run.

**Mitigation:**

- The contract's `workflow_pinning` rule (per [org ADR-0004](../decision-records/org/0004-use-renovate-for-dependency-updates.md)) requires every `uses:` to be a 40-character commit SHA with a tag comment. A compromised release tagged as `v4.1.1` does not change the SHA already pinned in the repo.
- Renovate's `minimumReleaseAge: 7 days` quarantine in `.github/renovate.json5` blocks bumps to releases newer than 7 days, giving the community time to surface a compromise before it propagates.
- `internalChecksFilter: strict` treats unresolvable timestamps as failing the quarantine, closing the loophole where a malicious release with absent metadata could otherwise slip through.

### T2 — Drift between mirrored org-baseline files and canonical source

**Scenario:** A maintainer edits `CODE_OF_CONDUCT.md` locally in a runner repo (or in this template) to weaken a policy without updating `NWarila/.github`. The drift goes unnoticed.

**Mitigation:**

- `.github/workflows/drift-gate.yaml` calls `NWarila/drift-gate@<sha>` on every PR. The composite action fetches the upstream files at the pinned `source-ref`, byte-compares them to the consumer copy, and fails the job on any divergence.
- The contract's `sync_drift` rule (in `tools/check_template_contract.py`) re-implements the same byte-identity check during `python tools/check_template_contract.py --type template`, so the comparison runs at template self-CI time in addition to runtime.

### T3 — A runner repo accidentally hosts framework-maintainer surface

**Scenario:** A maintainer copies a `tools/`, `policies/`, or local reusable workflow from `terraform-runner-template` into a Packer runner consumer, thinking it's needed. The runner repo becomes a duplicate framework, drifts from the canonical version, and tickets accumulate as the duplicate slowly diverges.

**Mitigation:**

- Contract `forbidden_paths` rules reject `tools/`, `policies/`, `Makefile`, `contract/`, and the glob `.github/workflows/reusable-*.yaml` (with only `reusable-auto-merge.yaml` allowed) at PR-validation time.
- ADR-template/0005 (from `packer-framework-template`) makes the data-only boundary explicit.

### T4 — Runner repo points its framework `uses:` at a tag instead of a SHA

**Scenario:** A maintainer writes `uses: NWarila/packer-framework-template/.github/workflows/reusable-packer-framework-build.yaml@main`. The framework's `main` advances; a regression lands silently in the runner's next build.

**Mitigation:**

- Contract `content_rule` enforces the SHA-pin regex pattern on the framework `uses:` line. Tags and branches fail with a `required pattern not found` marker.
- Renovate's `git-refs` custom manager keeps the SHA current with the framework's `main` branch via reviewable PRs.

### T5 — Renovate config drift across consumers

**Scenario:** A maintainer hand-rolls Renovate settings per repo. Common rules (cadence, quarantine) drift across repos. A repo accidentally disables `minimumReleaseAge`.

**Mitigation:**

- Contract `content_rule` requires every consumer's `.github/renovate.json5` to extend `github>NWarila/packer-runner-template//.github/renovate.json5` exactly. Consumers add only repo-specific overrides.
- The forbidden pattern `github>NWarila/.github` rejects consumers that try to extend the org repo (which intentionally has no Renovate config per [org ADR-0004](../decision-records/org/0004-use-renovate-for-dependency-updates.md)).

### T6 — Secrets accidentally committed

**Scenario:** A maintainer commits a `.env` file with an AWS key, or hard-codes a token in a workflow.

**Mitigation:**

- `.gitignore` is deny-all per [org ADR-0003](../decision-records/org/0003-use-deny-all-gitignore-strategy.md). Any new file is invisible to Git until explicitly allowlisted, which forces a maintainer to think about each path.
- The IaC security workflow runs Gitleaks on every PR and on a weekly schedule against full history.
- GitHub's org-level secret push protection (per `NWarila/.github` DESIGN.md) blocks pushes containing detected secrets before they enter the repo.

### T7 — Privileged workflow injection

**Scenario:** An attacker opens a malicious PR with crafted content. A workflow running under `pull_request_target` (which has write tokens) processes the PR's untrusted input and exfiltrates secrets or pushes malicious commits.

**Mitigation:**

- `reusable-auto-merge.yaml` is the only `pull_request_target` workflow in this template. It is byte-identical with the version in `terraform-runner-template` and `packer-framework-template`, and is mirrored byte-identically in every downstream consumer so the call graph is fully visible to static analyzers.
- zizmor runs in CI on every workflow YAML, flagging template-injection and command-injection patterns.
- The reusable's logic gates auto-merge on the PR author being a trusted bot identity and on all required checks passing. No PR content is interpolated into shell.

## Threats NOT addressed (out of scope)

| Out of scope | Why | Where mitigation lives |
|---|---|---|
| Mend (Renovate) infrastructure compromise | Third-party SaaS dependency. | Upstream supplier risk; revisit ADR-0004 if Mend's posture changes. |
| GitHub Actions runner sandbox escape | GitHub-managed; out of our control. | GitHub's own platform security. |
| `terraform-provider-github` provider-level bugs | Upstream code we don't own. | Track via provider's GitHub issues; pin minor versions. The `allow_forks` PATCH-payload bug documented in the framework's history is one example — workarounds live in the framework, not here. |
| The integrity of Packer plugin downloads | Framework concern. | `packer-framework-template`'s `packer/packer.pkr.hcl` pins plugins by exact version; framework-CI verifies plugin checksums. |
| Image artifact integrity (post-build) | Framework concern. | Framework's OPA `packer_artifact` policy + manifest custom_data fingerprints. |
| Compromised maintainer of this template | A maintainer with admin can bypass all gates. | Out-of-band: limit admin to NWarila + audit `gh api orgs/NWarila/audit-log`. |
| Compromised credentials for the GitHub PAT used in deploy | Token has org-write scope; if leaked, attacker can manage repos. | OIDC where possible (terraform-deploy already uses OIDC for AWS, but the GitHub PAT for Terraform is still a long-lived secret); store in repo secret with limited environment scope. |

## Reporting a security issue

Per [`SECURITY.md`](../../SECURITY.md), report vulnerabilities privately via GitHub's security advisory channel for this repository. Do not file public issues for security-sensitive findings.
