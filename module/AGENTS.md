# Packaging Skills

RPM packaging workflow automation for Foreman, Katello, and Pulp projects using community tools.

## Skills

### obal
**Triggers:** mentions of obal, COPR, Koji, package updates, spec files, RPM builds, packaging repository operations
**Use for:** Orchestrating RPM packaging workflows — version updates, mock/scratch/release builds, dependency verification with repoclosure, changelog generation, and PR testing in obal-managed repositories (foreman-packaging, pulpcore-packaging).

### packaging-systems-thinker
**Triggers:** dependency analysis, version planning, rebuild cascades, ABI/API changes, new package creation, security patches, major version updates
**Use for:** Shifting perspective from individual packages to the packaging ecosystem. Provides mental models for dependency graphs, version constraint matrices, rebuild cascade planning, and upstream-to-user pipelines. Pair with obal for the full picture — systems-thinker for the "how to think" and obal for the "how to do."

## Conventions

All content in this module is 100% upstream. Only community tools: COPR, Koji, obal, tito, GitHub, Fedora packaging guidelines. No internal build systems, authentication, or proprietary tooling.

### Repository Detection

Before running packaging commands, verify you're in an obal-managed repository:

```bash
ls package_manifest.yaml
```

Common repositories:
- `foreman-packaging` — Foreman and Katello packages
- `pulpcore-packaging` — Pulp server packages
