# Packaging Skills

RPM packaging workflow automation for Foreman, Katello, and Pulp projects using community tools.

## Skills

### obal
**Triggers:** obal, COPR, Koji, package updates, spec files, RPM builds, version bumps, mock/scratch builds  
**Use for:** Orchestrating RPM packaging workflows — version updates, mock/scratch/release builds, dependency verification with repoclosure, changelog generation, and PR testing in obal-managed repositories (foreman-packaging).

### copr-cli
**Triggers:** build failures, COPR logs, debugging builds, monitoring builds, analyzing build errors  
**Use for:** Inspecting COPR build status, retrieving and analyzing logs, classifying build failures, troubleshooting. Use when obal or a user reports a failed build and you need to understand what went wrong.

### packaging-repositories
**Triggers:** foreman-packaging, repository structure, spec files, package types, git-annex, source management, dependencies  
**Use for:** Understanding repository structure, editing specs manually, understanding package types and patterns, managing dependencies. Defers source file management to obal skill. Use when working directly with repository structure or understanding how specs are organized.

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

**Primary Repository:**
- `foreman-packaging` — Foreman, Katello, and related packages (ruby gems, python, node.js, ansible collections, go)
  - Components: `foreman/`, `katello/`, `client/`, `satellite/`, `plugins/`
  - Branches: `rpm/develop`, `rpm/3.19`, `rpm/3.18`, `rpm/3.17`, `deb/*` (Debian equivalents)
  - Source: https://github.com/theforeman/foreman-packaging
