# ADR-template/0001: Pin Packer and Plugin Versions Exactly

| Field          | Value                                   |
| -------------- | --------------------------------------- |
| Status         | Accepted                                |
| Date           | 2026-05-25                              |
| Authors        | Nick Warila (@NWarila)                  |
| Decision-maker | Nick Warila (sole portfolio maintainer) |
| Consulted      | packer-framework-template ADR-template/0001 (same decision at the framework tier) |
| Informed       | All downstream Packer runner consumers |
| Reversibility  | Medium                                  |
| Review-by      | N/A (Accepted)                          |

## TL;DR

Packer runner repositories derived from `NWarila/packer-runner-template` MUST pin Packer CLI and every Packer plugin to exact versions in their `packer/packer.pkr.hcl` (when present locally) or accept the framework's pin transitively (when overlaying onto a SHA-pinned framework checkout). Range operators (`~>`, `>=`, etc.) are forbidden. Renovate keeps pins current.

## Context and Problem Statement

Packer's plugin system resolves the latest version satisfying a range constraint at `packer init` time. A range constraint that yesterday resolved to `0.6.5` can resolve to `0.6.7` today if the upstream publishes a new release between runs. For image-build pipelines this is unacceptable:

1. **Reproducibility.** A build that succeeded yesterday must succeed today and tomorrow against the same inputs. If `packer init` silently upgrades a plugin between runs, the artifact's SBOM and the build's reproducibility claim are both invalidated.
2. **Security review.** Pinning to an exact version makes plugin upgrades reviewable changes in git. A range bump that drags in a compromised plugin would never appear in a PR diff.
3. **Compliance.** STIG, CIS, and SLSA Level 3 all require that the build environment's tool versions be traceable to a reviewed configuration. Range constraints break that traceability.

The runner contract's content rules already enforce SHA-pinning on every `uses:` entry in `.github/workflows/*.yaml`. This ADR extends the same discipline to the Packer-side pinning surface.

## Decision Drivers

1. **Build reproducibility across time.** Same inputs + same pins + same framework SHA must produce the same artifact.
2. **Reviewable upgrades.** Every plugin version bump must appear as a diff in a PR, not as a silent `packer init` resolution change.
3. **Renovate as the bump driver.** Pins MUST be machine-trackable so Renovate can keep them current. Range constraints defeat Renovate's `terraform`/regex managers.
4. **Cross-framework consistency.** Terraform runner template ADR-template/0001 already pins Terraform and providers exactly; Packer runner template follows the same discipline.
5. **Plugin source-path remap.** Packer's `source = "github.com/<owner>/<short>"` convention strips the `packer-plugin-` prefix from the real upstream repo. Renovate's `github-releases` datasource needs the real repo name; the canonical Renovate baseline in this template handles the remap via `packageNameTemplate`.

## Considered Options

1. **Pin every Packer CLI and plugin to an exact version (chosen).**
2. **Allow caret/tilde ranges (`~> 1.15`) for the CLI but pin plugins exactly.** Hybrid.
3. **Allow ranges everywhere; rely on `packer.lock.hcl` to freeze resolved versions.**
4. **Inherit framework pins transitively; runners declare no Packer pins of their own.**

## Decision Outcome

**Option 1, pin Packer CLI and every Packer plugin to an exact version.**

In any runner-local `.pkr.hcl` file (most runners overlay inputs and have no local `.pkr.hcl`, but those that do):

```hcl
packer {
  required_version = "= 1.15.0"

  required_plugins {
    git = {
      source  = "github.com/ethanmdavidson/git"
      version = "= 0.6.7"
    }
    proxmox = {
      source  = "github.com/hashicorp/proxmox"
      version = "= 1.2.5"
    }
  }
}
```

Only `=` is permitted as the version operator. `>=`, `~>`, `>`, `<`, and the bare-version shorthand (which Packer interprets as `>=`) are all forbidden in `version =` and `required_version =` declarations.

Runners that overlay inputs onto a SHA-pinned framework checkout inherit the framework's pins transitively. Those runners have no `.pkr.hcl` of their own and are covered by the framework's ADR-template/0001 via the framework SHA pin in `framework_ref`. Either path satisfies this ADR.

## Pros and Cons of the Options

### Option 1: Exact pins (chosen)

- **Good, because** every build is byte-traceable: same pins + same inputs + same framework SHA → same artifact.
- **Good, because** every plugin upgrade is a reviewable PR diff with a Renovate-authored body.
- **Good, because** Renovate's `customManager` for Packer plugins (declared in this template's `.github/renovate.json5`) keeps pins current automatically.
- **Good, because** the framework's plugin source-path remap (`<owner>/packer-plugin-<short>` → real GitHub repo) makes pins resolvable by the github-releases datasource.
- **Bad, because** plugin authors who don't follow semver discipline can ship breaking changes that block consumers until the consumer fixes inputs to match.

### Option 2: Caret/tilde ranges for CLI, exact for plugins

- **Good, because** CLI updates within a minor are usually backward-compatible.
- **Bad, because** "usually" isn't "always" — a 1.15.x → 1.15.y bump introducing a behavior change would land silently.
- **Bad, because** asymmetric pinning is harder to reason about.

### Option 3: Ranges + `packer.lock.hcl` to freeze resolution

- **Good, because** `packer.lock.hcl` does freeze resolution between `packer init` runs.
- **Bad, because** `packer.lock.hcl` is updated by `packer init -upgrade`, which a maintainer might run accidentally.
- **Bad, because** the source of truth is split between two files; reviewers must read both.

### Option 4: Framework pins only; runners declare nothing

- **Good, because** runners stay even thinner.
- **Bad, because** runners that DO carry local `.pkr.hcl` would have no pinning discipline at all.
- **Bad, because** this ADR's intent is to apply the rule UNIVERSALLY across the Packer surface, not just at the framework tier.

## Confirmation

Adherence to this ADR is confirmed by the following mechanisms. The wording `MUST`, `SHOULD`, and `MAY` follows [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) conventions.

1. **No range operators.** Every `version = "..."` line in a runner's `.pkr.hcl` files MUST start with `=` followed by a complete `MAJOR.MINOR.PATCH` triple. A reviewer SHOULD reject a PR that introduces or restores a range operator.
2. **`required_version` discipline.** Every `required_version = "..."` line in a runner's `.pkr.hcl` files MUST use the `= MAJOR.MINOR.PATCH` form. Bare versions MUST NOT be used.
3. **Renovate-trackable pins.** Each pin MUST be expressed in a form the Renovate baseline can recognize (see `customManagers` in `.github/renovate.json5`). A pin that Renovate cannot track MUST be accompanied by a follow-up issue to restore trackability.
4. **Lockfile is supplementary.** `packer.lock.hcl` MAY be committed for additional belt-and-suspenders coverage, but it is supplementary — the `= MAJOR.MINOR.PATCH` line in `.pkr.hcl` remains the authoritative pin.
5. **Framework SHA pin counts.** Runners that have no local `.pkr.hcl` (the common case — inputs-only runners) satisfy this ADR via the SHA-pinned `framework_ref` they pass to `reusable-packer-framework-build.yaml`.

## Consequences

### Positive

- Reproducible artifacts across days, weeks, and (with framework pin held) years.
- Every plugin upgrade visible in `git log`.
- Renovate handles the bump cadence; maintainers don't hunt versions manually.
- Aligns with `terraform-runner-template/docs/decision-records/template/0001-pin-terraform-and-provider-versions-exactly.md` so the portfolio's pinning story is uniform across stacks.

### Negative

- One additional discipline to enforce at PR time (until automation catches it via OPA or contract content rules).
- Plugin authors who break semver force more frequent manual intervention.

### Neutral

- A few canonical Packer plugins (notably the official `hashicorp/*` plugins) follow strict semver; pinning is well-supported there.

## Assumptions

1. The Renovate App is installed in the `NWarila` org and runs against this template + downstream runners.
2. The Packer plugin ecosystem continues to publish GitHub releases (Renovate's `github-releases` datasource depends on this).
3. The `packageNameTemplate` source-path remap continues to work as Packer's plugin hosting convention evolves.

## Supersedes

None.

## Superseded by

None (current).

## Implementing PRs

This ADR is itself the implementing PR — runner repositories adopt the pinning discipline at derivation time.

## Related ADRs

- [Org ADR-0004](../org/0004-use-renovate-for-dependency-updates.md) — establishes Renovate as the dependency update tool and the per-template-baseline pattern this Renovate config inherits from.
- [Org ADR-0005](../org/0005-pin-terraform-and-provider-versions-exactly.md) — sister decision for Terraform across the portfolio; same pinning rationale.
- `packer-framework-template/docs/decision-records/template/0001-pin-packer-and-plugin-versions-exactly.md` — the framework-tier sibling of this ADR.

## Compliance Notes

| Framework              | Control / Practice ID                                                | Potential Evidence Contribution                                                                                                |
| ---------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| NIST SP 800-218 (SSDF) | PS.3 (Archive and Protect Each Software Release)                     | Exact-pinned plugin/CLI versions in `.pkr.hcl` are traceable in source control and reproducible.                                |
| NIST SP 800-53 Rev. 5  | CM-2 (Baseline Configuration)                                        | Pinned versions form a baseline configuration for the build tooling.                                                            |
| SLSA Framework Level 3 | Build Provenance                                                     | Reproducible builds require pinned tooling; this ADR establishes the pinning requirement.                                       |
