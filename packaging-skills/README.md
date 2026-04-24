# Packaging Skills

AI agent skills for RPM packaging workflows with obal, COPR, Koji, and Brew.

## Overview

This lola module provides comprehensive skills for working with RPM packaging repositories used in Foreman, Katello, Pulp, and Satellite projects. It enables AI assistants to safely orchestrate common packaging operations with built-in safety guardrails.

## Installation

```bash
# Add to lola registry
lola mod add https://github.com/theforeman/packaging-skills

# Install to a project
lola install packaging-skills
```

Or install locally during development:

```bash
# From this repository root
lola mod add $(pwd)/packaging-skills
lola install packaging-skills
```

## Components

### Skills

#### obal
Orchestrates RPM packaging workflows for repositories like foreman-packaging, pulpcore-packaging, satellite-packaging, and candlepin-packaging.

**Features:**
- Package version updates with automatic spec file modification
- Local mock builds for rapid testing
- Scratch builds on COPR for pre-release verification
- Dependency verification with repoclosure
- Release builds with approval gates
- Changelog generation
- Pull request testing workflows

**Safety Features:**
- Location verification before git operations (prevents destructive operations in wrong directory)
- Release approval gates (prevents unauthorized production releases)
- Best practices enforcement

**Auto-triggers when:**
- User mentions obal, COPR, Koji, or Brew
- Working with package updates or spec files
- Running builds or verifying dependencies
- Testing packaging pull requests

### Commands

None currently. Custom commands can be added to `module/commands/`.

### Agents

None currently. Specialized agents can be added to `module/agents/`.

### MCP Servers

None currently. MCP server configurations can be added to `module/mcps.json`.

## Development

This module follows the lola module structure:

```
packaging-skills/
├── README.md           # This file (repo documentation)
└── module/             # Lola-importable content
    ├── skills/
    │   └── obal/
    │       └── SKILL.md
    ├── commands/       # (empty - no custom commands)
    ├── agents/         # (empty - no custom agents)
    ├── mcps.json       # (empty - no MCP servers)
    └── AGENTS.md       # Module-level instructions
```

### Testing

The skill has been validated through comprehensive evaluations:
- Iteration 2 achieved 100% pass rate across all test scenarios
- Git safety guardrails verified to prevent destructive operations
- Release approval gates confirmed working

### Contributing

1. Create a feature branch
2. Make changes to `module/skills/obal/SKILL.md` or add new components
3. Test locally with `lola install packaging-skills`
4. Submit a pull request

## License

Apache License 2.0 - See LICENSE file for details.
