---
name: "obal"
description: "Orchestrate RPM packaging workflows with obal for repositories like foreman-packaging, pulpcore-packaging, and satellite-packaging. Use when updating package versions, running builds (scratch, mock, release), verifying dependencies with repoclosure, linting spec files, or generating changelogs. Provides safety guardrails for destructive operations and release approval gates."
---

# obal Packaging Workflow Skill

## Overview

obal is an Ansible-based orchestrator for RPM packaging repositories. It wraps complex Ansible playbooks into simple commands for common packaging tasks like version updates, builds, and verification.

**New to obal?** See reference documentation:
- [Installation](references/installation.md) - Installing obal
- [Authentication](references/authentication.md) - COPR/Koji/Brew setup  
- [Repository Structure](references/repository-structure.md) - Packaging repo anatomy

**Always execute obal commands from the repository root directory**, not from package subdirectories. The repository root contains `package_manifest.yaml` which defines all packages and their metadata.

**Why:** obal uses `package_manifest.yaml` as an Ansible inventory to discover packages and their configuration. Running from subdirectories causes "package not found" errors because obal cannot locate the inventory file.

## Repository Detection

Before running any obal commands, verify you're in an obal-managed packaging repository.

**What is a packaging repository?** A specialized Git repository with:
- `package_manifest.yaml` (Ansible inventory defining packages)
- `packages/` directory with subdirectories for each package
- Each package directory contains `.spec` file and sources
- Source files (tarballs, gems, or managed via git-annex)

```bash
# Check for package_manifest.yaml
ls package_manifest.yaml

# Identify which repository you're in
basename $(pwd)
```

**Detailed repository structure:** See [Repository Structure Guide](references/repository-structure.md)

Common obal-managed repositories:
- `foreman-packaging` - Foreman and Katello packages
- `pulpcore-packaging` - Pulp packages
- `satellite-packaging` - Red Hat Satellite packages
- `candlepin-packaging` - Candlepin packages

## CRITICAL: Git/GitHub Operations Location Safety

**All git and gh operations MUST only be executed inside packaging repository directories.**

**Before ANY git or gh operation, verify your location:**

```bash
pwd  # Check current directory
ls package_manifest.yaml  # Verify it's a packaging repository
```

**If you're not in a packaging repository, navigate there first:**
```bash
cd <path-to-packaging-repo>
```

**Destructive git operations** (`git reset --hard`, `git clean -fd`, `git checkout -- ...`) **are ONLY permitted after verifying you're in a packaging repository.** If location verification fails, do not perform the operation.

**Safe alternatives when managing repository state:**
- Use `git stash` to save work instead of discarding
- Create a branch: `git checkout -b temp-save`

## Core Workflows

### 1. Updating Package Versions

**IMPORTANT:** The `obal update` command **requires** the `--version` flag to update a package. Without `--version`, the command will not perform an update.

**Why this requirement exists:** obal is a straightforward tool that does exactly what you tell it. Without `--version`, obal has no way to know what version you want, so it simply doesn't perform any update. The explicit `--version` flag ensures you're making intentional, documented version changes.

**Update to specific version:**
```bash
obal update <package-name> --version <version-number>
```

This updates the spec file to the **specified version** AND fetches the sources for that version. This is the primary way to update packages to new versions.

**Update with git commit:**
```bash
obal update <package-name> --version <version-number> --commit
```

**What to verify after updating:**
- Version field matches the target version
- Release field is reset to 1
- Source0 URL resolves correctly
- BuildRequires match upstream dependencies (setup.cfg/pyproject.toml/requirements.txt for Python; .gemspec/Gemfile for Ruby)
- Changelog entry was auto-generated (unless you provided `--changelog`)

**Always review the diff before committing** to catch issues early:
```bash
git diff
```

### 2. Linting Packages

Before building, run lint checks to catch spec file issues:

```bash
obal lint <package-name>
```

This runs rpmlint on the spec file and reports warnings/errors. Fix any critical issues before proceeding with builds.

**Common lint issues:**
- Missing or incorrect BuildRequires
- Spec file syntax errors
- Macro issues
- Missing documentation files

### 3. Verifying Dependencies (repoclosure)

After updating packages, verify all dependencies are available:

```bash
obal repoclosure <package-name>
```

This checks whether all BuildRequires and Requires can be satisfied from available repositories. If repoclosure fails, you may need to build or update dependency packages first.

**Testing against scratch builds:** After a scratch build with `-e build_package_archive_build_info=True`, use the `scripts/extract_copr_repos.py` helper to extract repository URLs:

```bash
# Extract COPR repo info and run repoclosure
while read -r url dist; do
    obal repoclosure <package-name> --check "$url" --dist "$dist"
done < <(scripts/extract_copr_repos.py --bash <package-name>)
```

See [Repoclosure Usage](references/repoclosure-usage.md) for detailed patterns and troubleshooting.

### 4. Local Mock Builds

Build packages locally using mock to catch issues before submitting to remote build systems:

```bash
obal mock <package-name>
```

Mock builds:
- Create an isolated chroot environment
- Install all BuildRequires
- Build the SRPM and RPM
- Verify the build completes successfully

**Mock builds are fast and run locally**, making them ideal for rapid iteration during development.

### 5. Scratch Builds (COPR/Koji)

Scratch builds test packages without publishing to repositories.

**COPR scratch build (default):**
```bash
obal scratch <package-name>
```

**COPR with custom config file:**
```bash
obal scratch <package-name> --copr-config /path/to/copr-config
```

**Koji scratch build:**
Requires `package_manifest.yaml` to specify:
```yaml
all:
  vars:
    build_package_build_system: koji
    build_package_koji_command: koji  # or 'brew' for Brew
```

Then run: `obal scratch <package-name>`

**IMPORTANT:** The `--copr-config` flag is ONLY for COPR builds. Koji/Brew builds are controlled by variables in `package_manifest.yaml`, not command-line flags.

After submitting, obal prints the build URL. **Save this URL and monitor the build** to verify success:
- Check build logs for errors
- Verify all sub-packages were built
- Test install the resulting RPMs if needed

**Scratch builds run automatically without approval** because they don't affect any release repositories.

**See also:** [Scratch Build Workflow](references/scratch-build-workflow.md) for real-world CI patterns.

### 6. Release Builds

Release builds publish packages to production repositories and should be used with caution.

**CRITICAL SAFETY CHECK:** Never run `obal release` without explicit user authorization. Always confirm:
1. The package has been tested via scratch or mock builds
2. All dependencies pass repoclosure
3. The user explicitly requested a release build
4. You understand which chroot/target the release is for

**Why this safety check exists:** Release builds publish packages to production repositories used by downstream consumers. Accidental releases can:
- Break production systems depending on stable packages
- Publish untested code to users
- Require emergency rollbacks and notifications
- Violate release processes and compliance requirements

The approval gate ensures releases are intentional and tested.

**Release to COPR:**
```bash
obal release <package-name>
```

**Release to specific COPR chroot:**
```bash
obal release <package-name> --copr-chroot <chroot-name>
```

**Release to Koji:**
Requires `package_manifest.yaml` configuration (see scratch build section above).
```bash
obal release <package-name>
```

**Before releasing, ask the user:**
- "Are you sure you want to release `<package-name>` to `<target>`? This will publish to production repositories."
- Wait for explicit confirmation before proceeding

### 7. Generating Changelogs

**Note:** `obal update` automatically generates a changelog entry like `"- Release {package} {version}"` unless you provide `--changelog` to customize it.

Use the standalone `obal changelog` command when you need to add a changelog entry WITHOUT updating the package version (e.g., for rebuilds, patches, or spec file fixes):

```bash
obal changelog <package-name>
```

This writes a changelog entry for the current version and release. Customize the entry text:

```bash
obal changelog <package-name> --changelog "Fixed bug #12345"
```

**When to use each:**
- `obal update --version X.Y.Z` - automatically adds changelog entry
- `obal update --version X.Y.Z --changelog "Custom message"` - updates with custom changelog
- `obal changelog --changelog "Message"` - adds entry without updating version (rebuilds, patches)

### 8. Fetching Sources (git-annex)

Some repositories use git-annex to store large source files. Retrieve sources:

```bash
obal source <package-name>
```

This downloads source tarballs or gems from the remote git-annex repository. Required before building if sources aren't already present.

## Common Workflow Patterns

### Pattern 1: Update and Quick Test

**Use when:** Testing package updates locally before submitting builds.

**Copy this checklist:**

```markdown
- [ ] Update to new version: `obal update mypackage --version 2.3.4`
- [ ] Lint first (fast error detection): `obal lint mypackage`
- [ ] Review changes: `git diff`
- [ ] Quick local build: `obal mock mypackage`
```

**Why this workflow:** Linting catches spec file errors in ~5 seconds, while mock builds take 5-15 minutes. Catching errors early saves time and prevents failed builds.

### Pattern 2: Update and Scratch Build

**Use when:** Testing package updates in the target build environment (COPR/Koji) before release.

**Copy this checklist:**

```markdown
- [ ] Update package: `obal update mypackage --version 2.3.4`
- [ ] Lint before building: `obal lint mypackage`
- [ ] Scratch build on COPR: `obal scratch mypackage -e build_package_archive_build_info=True`
- [ ] Monitor build URL and verify success
- [ ] Verify dependencies (after build completes):
      ```bash
      while read -r url dist; do
          obal repoclosure mypackage --check "$url" --dist "$dist"
      done < <(scripts/extract_copr_repos.py --bash mypackage)
      ```
```

**Why this workflow:** Scratch builds test in the real build environment without publishing. The `build_package_archive_build_info=True` flag enables repoclosure testing against the built RPMs.

### Pattern 3: Full Release Workflow

**Use when:** Publishing a package update to production repositories.

**Copy this checklist:**

```markdown
- [ ] Update package (auto-generates changelog, commits to new branch): `obal update mypackage --version 2.3.4 --commit`
- [ ] Lint: `obal lint mypackage`
- [ ] Local test: `obal mock mypackage`
- [ ] Scratch test: `obal scratch mypackage -e build_package_archive_build_info=True`
- [ ] Verify dependencies (after scratch build completes):
      ```bash
      while read -r url dist; do
          obal repoclosure mypackage --check "$url" --dist "$dist"
      done < <(scripts/extract_copr_repos.py --bash mypackage)
      ```
- [ ] Get explicit user approval for release
- [ ] Release to production: `obal release mypackage`
- [ ] Verify release appears in repository
```

**Why this workflow:** Each step acts as a quality gate. Local tests (lint, mock) are fast and catch most issues. Remote tests (scratch, repoclosure) verify behavior in target environment. Explicit approval prevents accidental production releases.

**CRITICAL:** Never skip the approval step. Release builds publish to production repositories and can affect downstream users.

### Pattern 4: Testing a Pull Request

**Use when:** Validating packaging changes in a PR before merging.

**Copy this checklist:**

```markdown
- [ ] Fetch the PR: `gh pr checkout <pr-number>`
- [ ] Identify changed packages:
      ```bash
      git diff --name-only origin/develop | grep package_manifest.yaml || \
      git diff --name-only origin/develop | grep "packages/"
      ```
- [ ] For each changed package:
  - [ ] Lint: `obal lint <package-name>`
  - [ ] Local build: `obal mock <package-name>` OR
  - [ ] Scratch build: `obal scratch <package-name>`
- [ ] Verify builds succeed and add review comments to PR
```

**Why this workflow:** PR testing catches integration issues before merging. Running lint and builds on changed packages ensures the PR doesn't break existing functionality.

## Safety Guardrails

### Destructive Operations Require Approval

**Block these operations unless explicitly requested:**
- `obal release` - publishes to production repositories
- Any operation with `--copr-rebuild` - forces rebuild of existing packages
- Operations targeting Brew/Koji without clear user intent

**Before running destructive operations:**
1. Summarize what will happen: "This will release `package-name` version `X.Y.Z` to `target-repository`"
2. Ask: "Do you want to proceed with this release?"
3. Wait for explicit "yes" or confirmation
4. Only then execute the command

### Auto-Approved Operations

**These operations are safe to run automatically:**
- `obal source` - read-only download of sources
- `obal lint` - read-only check
- `obal repoclosure` - read-only verification
- `obal mock` - local build only
- `obal scratch` - temporary builds not tagged in repos
- `obal changelog` - only modifies local files

## Package Manifest Reference

The `package_manifest.yaml` is an Ansible inventory defining all packages and their metadata.

**Query package information:**
```bash
# Look up a specific package and see all its variables
ansible-inventory -i package_manifest.yaml --host mypackage
```

**See also:** [Package Manifest Structure](references/package-manifest-structure.md) for complete reference including:
- YAML structure and examples
- Group operations and hierarchy
- Listing all packages
- Build system configuration variables
- Advanced operations (custom release numbers, prerelease versions)

## Troubleshooting

**Quick diagnostics:**
- Build fails: Check BuildRequires, verify Source0 URL, review logs
- Authentication error: Verify config files and credentials (COPR/Koji/Brew)
- Package not found: Check spelling, see [Package Manifest Structure](references/package-manifest-structure.md#listing-all-packages) for how to list all packages
- Not in git repository: Change to repository root directory

**For detailed troubleshooting:** See [Troubleshooting Guide](references/troubleshooting-guide.md) covering:
- Setup issues (permissions, authentication, repository detection)
- Build failures (mock, scratch, repoclosure)
- Authentication problems (COPR, Koji, Brew)
- Common errors and solutions

## Advanced Operations

For advanced operations, see:
- [Parallel Builds](references/parallel-builds.md) - Multiple package builds
- [Build System Config](references/build-system-config.md) - Custom chroots, skip checks
- [Package Manifest Structure](references/package-manifest-structure.md) - Custom release numbers, prerelease versions

## Best Practices

1. **Always lint before building** - Catch spec file issues early with `obal lint`

2. **Verify dependencies** - Run `obal repoclosure` after updates to ensure all deps are buildable

3. **Test locally first** - Use `obal mock` for fast iteration before remote builds

4. **Use scratch builds for testing** - Never release without scratch build verification

5. **Review diffs carefully** - Check `git diff` after `obal update` to catch unexpected changes

6. **Commit frequently** - Use `--commit` flag or commit manually after each successful update

7. **Monitor build URLs** - Always check build logs for warnings even on successful builds

8. **Understand your target** - Know whether you're building for COPR (upstream) or Brew (downstream)

9. **Respect release gates** - Never bypass safety checks for release builds without understanding why

10. **Document changes** - Use meaningful changelog entries with `obal changelog`

## When to Use This Skill

**Trigger this skill when the user:**
- Mentions obal, COPR, Koji, Brew, or packaging repositories (foreman-packaging, pulpcore-packaging, etc.)
- Works with package updates, RPM builds, spec files, or package manifests
- Asks about dependency verification, changelogs, or package testing
- Reviews or tests packaging pull requests

**Auto-activate:** If you detect `package_manifest.yaml` or packaging repository structure in the working directory.

## Additional Resources

### Setup & Configuration
- [Installation](references/installation.md) - Installing obal and dependencies
- [Authentication](references/authentication.md) - COPR, Koji, and Brew setup
- [Repository Structure](references/repository-structure.md) - Packaging repository anatomy
- [Package Manifest](references/package-manifest-structure.md) - Understanding package_manifest.yaml
- [Build System Config](references/build-system-config.md) - Configuring build targets

### Workflows
- [Scratch Build Workflow](references/scratch-build-workflow.md) - Testing packages
- [Release Workflow](references/release-workflow.md) - Publishing to production
- [Repoclosure Usage](references/repoclosure-usage.md) - Dependency verification
- [New OS Version](references/new-os-version-workflow.md) - Bootstrap builds
- [Parallel Builds](references/parallel-builds.md) - Multi-package builds

### Troubleshooting & Best Practices
- [Troubleshooting Guide](references/troubleshooting-guide.md) - Setup, builds, authentication
- [CI Best Practices](references/ci-best-practices.md) - General CI guidelines

## License

This skill is part of the packaging-skills project licensed under GPL-3.0. See [LICENSE](../../../../LICENSE) at the repository root.
