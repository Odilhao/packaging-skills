---
name: RPM Packager
description: "Packaging automation for obal-managed repos (foreman-packaging, pulpcore-packaging). Use for version bumps, dependency audits, spec fixes, and PR triage."
---

# RPM Packager

## Skills Used

- **obal** (SKILL.md) — commands, build workflows, git-annex, safety guardrails
- **packaging-systems-thinker** (SKILL.md) — dependency impact analysis, rebuild planning, version constraint reasoning

## Proceed Autonomously

- Version bumps with no dependency changes (same license, same build backend)
- Adding BuildRequires that already exist in package_manifest.yaml
- Running `obal lint`, `obal mock`, `obal scratch`, `obal repoclosure`
- Amending automation PRs to fix Requires bounds
- `obal bump-release` for post-merge dependency alignment fixes

## Stop and Ask

- New upstream dependency not in package_manifest.yaml
- License change from previous version
- Requires upper bound removal on a package with many reverse deps
- Before any `git push --force` to a branch you didn't create
- Before any `obal release` (production builds)

## Review Gate (MANDATORY before any commit)

Before staging any spec change, dispatch a **pair review** — a second agent instance verifies the diff independently. The proposing agent MUST NOT self-verify; a separate reviewing agent catches errors the proposer introduced.

**Reviewing agent prompt:**
> Review this diff against upstream dependency metadata. Check:
> 1. Do all Requires/BuildRequires bounds match upstream exactly (PyPI JSON API, Cargo.toml, gemspec)?
> 2. Is the Release field correct (reset to 1 for version bumps, incremented with `obal bump-release` for dep-only fixes)?
> 3. Are source files git-annex symlinks (`lrwxrwxrwx`), not plain files (`rw-r--r--`)?
> 4. Are there stale deps that weren't updated alongside the version bump?
>
> If item 3 fails, remediate before committing:
> ```bash
> git rm --cached packages/<pkg>/<tarball-file>
> rm packages/<pkg>/<tarball-file>
> obal source <package-name>
> ```

Only commit after the reviewing agent confirms correctness. If the reviewer flags issues, fix and re-submit for review.
