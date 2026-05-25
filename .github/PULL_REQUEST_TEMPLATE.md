## Summary

<!-- 1-3 bullets describing what this PR changes and why. -->

## Risk

<!-- What could break? What automated evidence should the reviewer trust? -->

## Automated evidence

- [ ] CI passes in GitHub
- [ ] Drift Gate passes when active (`org-baseline / verify` once mirrors are added)
- [ ] Security workflow passes, or advisory findings are reviewed and documented
- [ ] Documentation reflects the change (when applicable)

## Template-maintainer notes

- [ ] Contract changes ship with positive and negative fixtures
- [ ] Reusable workflow signature changes are documented as breaking with a major release
- [ ] ADR added when the change is architecturally load-bearing
- [ ] The diff respects the thin-runner boundary for downstream consumers
