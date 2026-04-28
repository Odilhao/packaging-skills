# Packaging Skills

RPM packaging workflow automation skills for Foreman, Katello, Pulp, and Satellite projects.

## When to Use

- **obal**: Use the `obal` skill for RPM packaging workflows in repositories like foreman-packaging, pulpcore-packaging, satellite-packaging, and candlepin-packaging. Automatically triggers when:
  - User mentions obal, COPR, Koji, or Brew
  - Working with package updates, version bumps, or spec files
  - Running builds (scratch, mock, release)
  - Verifying dependencies with repoclosure
  - Testing packaging pull requests
  - Generating changelogs for package updates

## Configuration

No additional configuration required. The obal skill includes:
- Safety guardrails for destructive git operations
- Release approval gates (prevents unauthorized production releases)
- Best practices for RPM packaging workflows
- Integration with COPR, Koji, and Brew build systems

## Notes

The obal skill provides comprehensive guidance for:
- Package version updates and spec file modifications
- Local mock builds for rapid iteration
- Scratch builds on COPR for testing before release
- Dependency verification with repoclosure
- Full release workflow with changelog generation
- Pull request testing workflows

All operations include location verification to prevent accidental destructive operations in wrong directories.
