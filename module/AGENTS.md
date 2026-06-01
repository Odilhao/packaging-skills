# Packaging Skills

RPM packaging workflow automation for Foreman, Katello, and Pulp projects using community tools.

## Skills

### obal
**Triggers:** mentions of obal, COPR, Koji, package updates, spec files, RPM builds, packaging repository operations
**Use for:** Orchestrating RPM packaging workflows — version updates, mock/scratch/release builds, dependency verification with repoclosure, changelog generation, and PR testing in obal-managed repositories (foreman-packaging, pulpcore-packaging).

### packaging-systems-thinker
**Triggers:** dependency analysis, version planning, rebuild cascades, ABI/API changes, new package creation, security patches, major version updates
**Use for:** Reasoning about packaging as a system — dependency graphs, version constraint matrices, rebuild cascade planning, upstream-to-user pipelines. This skill provides interpretation and planning only, not commands. Activate after obal has identified the task; use systems-thinker to understand impact and plan the approach.

## Agents

### rpm-packager
**Use for:** End-to-end packaging automation in obal-managed repos. Wires together obal (commands) and packaging-systems-thinker (reasoning) with authority boundaries and a mandatory review gate before commits.

See [agents/rpm-packager.md](agents/rpm-packager.md) for the full agent definition.

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
