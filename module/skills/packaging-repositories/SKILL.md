---
name: "Packaging Repositories"
description: "Work with obal-managed packaging repositories: update RPM specs, manage git-annex sources, create package PRs. Use when updating packages, managing dependencies, working with git-annex sources, or handling Ruby gems and Python wheels in packaging repos."
---

# Packaging Repositories

## What This Skill Does

Guides you through working with obal-managed packaging repositories like [foreman-packaging](https://github.com/theforeman/foreman-packaging):

1. Updating package versions and RPM spec files
2. Managing source files with git-annex
3. Creating PRs for package updates
4. Understanding repository organization and structure
5. Working with Ruby gems, Python wheels, and other package types

---

## Prerequisites

- **obal skill** — required for package updates, building, and source management
- `git` and `git-annex` installed
- Standard CLI tools: `curl`, `jq`
- GitHub CLI (`gh`) for PR operations

---

## Repository Structure

All obal-managed repositories follow a consistent structure:

```
[repository]/
├── packages/
│   ├── [component-1]/
│   │   ├── [package-1]/
│   │   │   ├── [package-name].spec    # RPM spec file
│   │   │   └── [sources]              # Binaries (git-annex)
│   │   └── [package-2]/
│   ├── [component-2]/
│   └── dependencies/                  # (legacy, ignore)
├── package_manifest.yaml              # Package metadata
├── rel-eng/                           # Release engineering tools
├── .git/
└── .gitannex/                         # git-annex config
```

### Repository Types

#### foreman-packaging
The primary upstream packaging repository for Foreman, Katello, and related projects.

- **Component structure**: 
  - `packages/foreman/` — Core Foreman packages
  - `packages/katello/` — Katello (content management) packages
  - `packages/client/` — Foreman client packages
  - `packages/satellite/` — Red Hat Satellite-specific packages
  - `packages/plugins/` — Node.js and other plugin packages

- **Package types**: Ruby gems, Python packages, Node.js packages, Ansible collections, Go packages
- **Branch convention**: 
  - `rpm/develop`, `rpm/3.19`, `rpm/3.18`, `rpm/3.17` (and older releases)
  - `deb/develop`, `deb/3.19`, etc. (Debian equivalents)
  - Automated branches: `bump_rpm/[package]`, `bump_deb/[package]`

- **Example packages**:
  ```
  packages/foreman/foreman-assets/
  packages/katello/rubygem-katello/
  packages/katello/rubygem-foreman_rh_cloud/
  packages/client/ansible-collection-redhat-satellite/
  packages/plugins/react/
  packages/plugins/webpack/
  ```

---

## Branch Naming

All obal-managed repositories use consistent branch naming:

| Release | Branch |
|---------|--------|
| Nightly/Development | `rpm/develop` |
| Latest Release | `rpm/3.18` (example) |
| Previous Release | `rpm/3.17` |
| Patch Release | `rpm/3.17.1` (when needed) |

**Determine active branches**:
```bash
git branch -a | grep rpm/
```

---

## Quick Reference

### Updating Packages

**For package version updates and source management**, use the **obal skill**. It handles:
- Version bumps with `obal update <package> --version X.Y.Z`
- Source file management via `obal source`
- Changelog generation
- Lint and mock builds
- All git-annex operations

See the **obal skill** for the complete update workflow.

### Manual Spec File Edits

If you're only editing the spec file directly (e.g., fixing BuildRequires, changing Release without updating version):

```bash
# 1. Checkout correct branch
git checkout rpm/3.18
git pull

# 2. Create fix branch
git checkout -b fix-package-name-buildrequires

# 3. Navigate to package
cd packages/component/package-name

# 4. Edit spec file
vim package-name.spec
# - Change Release: if rebuilding (increment number)
# - Update BuildRequires/Requires if fixing dependencies
# - Add changelog entry with updated date

# 5. Stage and commit
git add package-name.spec
git commit -m "Fix BuildRequires for package-name"

# 6. Push and create PR
git push origin fix-package-name-buildrequires
gh pr create --base rpm/3.18
```

**Note**: This skill handles understanding repository structure and manual spec edits. For orchestrated package updates, see the **obal skill**.

### Check Current Version

```bash
grep "^Version:" packages/component/package-name/package-name.spec
```

### Compare Versions Across Branches

```bash
for branch in rpm/3.17 rpm/3.18 rpm/develop; do
  version=$(git show $branch:packages/component/package-name/package-name.spec 2>/dev/null | grep "^Version:")
  echo "$branch: $version"
done
```

---

## Package Types and Patterns

See [Package Types Guide](references/package-types-guide.md) for comprehensive reference:
- Ruby gems (standard and Foreman plugins with `# template: foreman_plugin`)
- Python packages (modern `%py3_build`/`%pyproject_build`, Rust extensions)
- Node.js, Ansible collections, Go packages, SCL (legacy)

---

## Spec File Anatomy

See [Spec File Reference](references/spec-file-reference.md) for:
- Minimal and template specs (Ruby gem, Python, Foreman plugin)
- Complete section reference (%global, %prep, %build, %install, %files, %changelog)
- All macros and their expansions

---

## Managing Source Files with git-annex

**Important**: Source file management in packaging repositories is handled by the **obal skill**, which integrates with git-annex correctly. Always use `obal source <package-name>` instead of managing git-annex directly.

**Why this matters**: When you run `git annex add` directly without registering the source URL:
1. git-annex creates a local-only content-addressed key (SHA256E)
2. The file becomes a symlink pointing to a hash with no known upstream location
3. On CI/Jenkins, the symlink target doesn't exist (not in CDN), build fails with `Bad file: No such file or directory`
4. Your change works locally but breaks for everyone else

**The obal approach**:
1. `obal source <package>` calls `git annex addurl <upstream-url> <file>`
2. Creates an annex key that knows both the local content AND the upstream URL
3. Registers the file with the CDN for CI systems to access
4. Symlink works everywhere (local + CI + remote clones)

### Understanding git-annex

Packaging repositories use git-annex to manage large binary files (gems, tarballs, wheels) without storing them directly in git. This keeps the repository size manageable.

### How obal Handles Sources

**Use the obal skill for source management**:
```bash
# Fetch sources for a package (handles all git-annex operations correctly)
obal source package-name

# For Rust vendor tarballs (remove old files first, then fetch)
rm packages/component/package-name/*.tar.gz 2>/dev/null || true
obal source package-name

# Verify files are symlinks, not plain files
ls -la packages/component/package-name/
# Should show: lrwxrwxrwx (symlink), NOT -rw-r--r-- (plain file)
```

See the **obal skill** for complete source management workflows, including:
- Ruby gems (gem fetch)
- Python wheels (pip download)
- Rust vendor tarballs
- Vendor tarball generation for new versions

### When to Use Direct git-annex Commands

Only for read-only operations or recovery:

```bash
# Check which files are annexed
git annex list | head -20

# Download files locally (read-only)
git annex get packages/component/package-name/*.gem

# Remove local copy but keep in annex (cleanup disk space)
git annex drop packages/component/package-name/*.gem
```

**Never use `git annex add` directly** — always use `obal source` instead.

---

## Changelog Format

### Entry Format

```
* Day Mon DD YYYY Full Name <email@domain.com> - VERSION-RELEASE
- Changelog message
```

### Common Message Patterns

```
- Release package-name 1.2.4                    # Version bump
- Update to 1.2.4                               # Also acceptable
- Rebuild for dependency update                 # Release bump only
- Refs upstream#12345 - Fix description         # Link to upstream
- Add missing BuildRequires: python3-setuptools # Spec fix
```

### Generate Proper Date

```bash
date "+%a %b %d %Y"
# Output: Tue Jun 02 2026
```

---

## Common Tasks

**Add new package**: Copy spec from similar package, edit, run `obal source <package>`, commit & PR  
**Bump release (rebuild)**: Edit spec Release field, add changelog entry  
**Update dependencies**: Edit BuildRequires/Requires, bump Release, add changelog entry  
**Detect package type**: Check `ls packages/*/rubygem-*`, `ls packages/*/python-*` for patterns

---

## Advanced Patterns

See [Advanced Patterns Reference](references/advanced-patterns.md) for comprehensive coverage:
- **Epoch versioning** — When/why to use, the permanent rule
- **Conditional dependencies** — Version-dependent requirements
- **Satellite-specific builds** — Downstream branching and dist tags
- **Vendor tarballs** — Rust-extension Python packages (cryptography, nh3)
- **Software collections** — Legacy multi-version support
- **Template variations** — foreman_plugin, smart_proxy_plugin, scl
- **Performance tuning** — Parallel builds, skip tests
- **Multi-package builds** — Subpackages and multiple RPMs per spec

---

## PR Guidelines

### Title Format

```
Update rubygem-example to 1.2.4
Add python-newpackage 1.0.0
Rebuild ruby-example for Foreman 3.18
```

### PR Description Template

```markdown
## Summary
Update rubygem-example to version 1.2.4

## Changes
- Version bump to 1.2.4
- No spec changes required

## Testing
- [ ] Built successfully in local mock build
- [ ] Tested with packaging workflow
- ([ ] Verified in COPR scratch build if doing pre-release)

## Upstream References
- Upstream release: https://github.com/example/example/releases/tag/v1.2.4
- Changelog: https://github.com/example/example/blob/v1.2.4/CHANGELOG.md
```

### Continuous Integration

Repositories run automated checks on PRs:
- **Spec lint** — rpmlint validation
- **Mock build** — test build in local environment
- **Dependency verification** — check BuildRequires/Requires are complete

All must pass before merging.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `git annex: not available` | `sudo dnf install git-annex` |
| File not available locally | `git annex get path/to/file.gem` |
| Merge conflict in annexed files | `git annex get --from origin [file]` then `git annex drop [old-file]` |
| Missing BuildRequires | `dnf provides "*/missing-file"` then add to spec |
| Macro undefined | Verify template line, check `rpm -q foreman-rpm-macros` |
| Source URL returns 404 | Verify version is correct, check for version skips |

---

## Integration with obal

For automated package updates and builds, use the **obal** skill:

```bash
# Update package (semi-automated)
obal update rubygem-example

# Mock build (local testing)
obal build rubygem-example

# Scratch build (COPR pre-release testing)
obal scratch build rubygem-example
```

See **obal** skill for full automation workflows.

---

## Related Skills

- **obal**: Automated packaging workflows and build orchestration
- **copr-cli**: Build inspection and log analysis
- **packaging-systems-thinker**: Dependency analysis and rebuild planning

---

## Resources

- [Fedora Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)
- [RPM Spec Reference](https://rpm.readthedocs.io/)
- [git-annex Documentation](https://git-annex.branchable.com/)
- [foreman-packaging](https://github.com/theforeman/foreman-packaging)
