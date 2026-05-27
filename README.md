# Packaging Skills

AI agent skills for RPM packaging workflows with obal, COPR, Koji, and community packaging tools.

## Overview

This [lola](https://lobstertrap.org/lola/) module provides skills for working with RPM packaging repositories used in Foreman, Katello, and Pulp projects. It enables AI assistants to safely orchestrate common packaging operations with built-in safety guardrails.

All content is 100% upstream — only community tools and open source workflows.

## Prerequisites

- [obal](https://github.com/theforeman/obal) — Ansible-based RPM packaging orchestrator
- [copr-cli](https://docs.pagure.org/copr.copr/) — COPR build system client
- [gh](https://cli.github.com/) — GitHub CLI
- [git-annex](https://git-annex.branchable.com/) — Large file management for packaging repos
- `curl`, `jq` — Standard CLI utilities

## Installation

```bash
# Install lola
pip install lola-ai
# or
uv tool install lola-ai

# Add the module
lola mod add https://github.com/Odilhao/packaging-skills.git

# Install to a project
lola install packaging-skills
```

Or install locally during development:

```bash
# From this repository root
lola mod add $(pwd)
lola install packaging-skills
```

## Components

### Skills

#### obal
Orchestrates RPM packaging workflows for obal-managed repositories like foreman-packaging and pulpcore-packaging.

- Package version updates with automatic spec file modification
- Local mock builds for rapid testing
- Scratch builds on COPR for pre-release verification
- Dependency verification with repoclosure
- Release builds with approval gates
- Changelog generation
- Pull request testing workflows

**Safety features:**
- Location verification before git operations
- Release approval gates (prevents unauthorized production releases)

**Auto-triggers when:** user mentions obal, COPR, Koji, package updates, spec files, or RPM builds.

#### packaging-systems-thinker
A thinking framework for seeing software packaging as interconnected systems rather than individual packages.

- Dependency graph analysis (build vs runtime, direct vs transitive)
- Version constraint matrices and compatibility windows
- Rebuild cascade planning before making changes
- Upstream-to-user pipeline understanding
- Packaging trade-off analysis (bundling, subpackages, linking)

**Auto-triggers when:** analyzing dependencies, planning version bumps, debugging build failures, assessing ABI/API changes, or creating new packages.

### Commands

None currently. Custom commands can be added to `module/commands/`.

### Agents

None currently. Specialized agents can be added to `module/agents/`.

### MCP Servers

None currently. MCP server configurations can be added to `module/mcps.json`.

## Directory Structure

```
packaging-skills/
├── README.md               # This file
├── LICENSE
└── module/                 # Lola-importable content
    ├── AGENTS.md           # Module-level context and orchestration
    ├── mcps.json           # MCP server configs (empty)
    ├── skills/
    │   ├── obal/
    │   │   ├── SKILL.md    # obal packaging workflow skill
    │   │   ├── references/ # Deep-dive reference docs
    │   │   └── scripts/    # Helper scripts
    │   └── packaging-systems-thinker/
    │       └── SKILL.md    # Packaging systems thinking framework
    ├── commands/           # Custom commands (placeholder)
    └── agents/             # Specialized agents (placeholder)
```

## Contributing

1. Create a feature branch
2. Make changes under `module/`
3. Verify no downstream references: `grep -ri 'brew\|satellite-packaging\|kinit\|kerberos' module/`
4. Test locally with `lola mod add $(pwd) && lola install packaging-skills`
5. Submit a pull request

## Resources

- [Lola documentation](https://lobstertrap.org/lola/)
- [Module creation guide](https://lobstertrap.org/lola/guides/creating-modules/)
- [Skill format](https://lobstertrap.org/lola/guides/skill-format/)
- [obal](https://github.com/theforeman/obal)
- [Fedora Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)

## License

Apache License 2.0 — See LICENSE file for details.
