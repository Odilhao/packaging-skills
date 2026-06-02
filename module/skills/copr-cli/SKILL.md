---
name: "COPR CLI"
description: "Inspect COPR build status, fetch logs, and classify build failures. Use when debugging build failures, monitoring builds, analyzing logs, or researching package build issues in Fedora or upstream packaging."
---

# COPR CLI

## What This Skill Does

Provides workflows for interacting with [COPR](https://copr.fedorainfracloud.org/) (Community Build System):

1. **Build Inspection**: Check build status and results
2. **Log Retrieval**: Fetch and analyze build logs
3. **Failure Classification**: Identify common failure patterns
4. **Build History**: Track package builds across versions and branches

---

## Prerequisites

- [`copr-cli`](https://docs.pagure.org/copr.copr/) installed and configured
- COPR credentials in `~/.config/copr` (optional, for private projects)
- `curl`, `jq` for log processing (optional but recommended)

---

## Quick Start

### Check Build Status

```bash
# View builds for a project
copr-cli list-builds OWNER/PROJECT

# Check specific build
copr-cli status BUILD_ID

# Example: foreman-nightly Katello builds (most recent first)
copr-cli list-builds theforeman/foreman-nightly
```

### Get Build Logs

```bash
# Download build (downloads SRPM and RPMs)
copr-cli download-build BUILD_ID

# Or fetch logs directly via curl
curl -L https://copr.fedorainfracloud.org/results/owner/project/fedora-XX-x86_64/BUILD_ID/build.log

# View build status and available artifacts
copr-cli status BUILD_ID
```

---

## Common Workflows

### 1. Investigate Failed Build

```bash
# Get build info
copr-cli status BUILD_ID

# Fetch logs
curl -O https://copr.fedorainfracloud.org/results/owner/project/fedora-XX-x86_64/BUILD_ID/build.log

# Analyze error
tail -100 build.log | grep -i error

# Check rootlog for setup errors
curl -O https://copr.fedorainfracloud.org/results/owner/project/fedora-XX-x86_64/BUILD_ID/root.log
```

### 2. Compare Builds Across Branches

```bash
# List recent builds on multiple branches
copr-cli list-builds theforeman/foreman-3.18 --latest 5
copr-cli list-builds theforeman/foreman-3.17 --latest 5

# Compare versions of same package
for branch in foreman-3.18 foreman-3.17; do
  echo "=== $branch ===" 
  copr-cli list-builds theforeman/$branch | grep "rubygem-example"
done
```

### 3. Find Success Pattern

```bash
# Get successful builds
copr-cli list-builds OWNER/PROJECT | grep succeeded

# Identify what changed between failed and success
copr-cli get-build PREVIOUS_BUILD_ID
copr-cli get-build CURRENT_FAILED_ID

# Compare changelogs
```

---

## Build Log Analysis

### Common Failure Patterns

#### Missing BuildRequires
```
error: Failed build dependencies:
	python3-devel is needed by...
```
**Fix**: Add missing package to `BuildRequires` in spec file

#### Dependency Version Mismatch
```
Error: Package requirement libfoo >= 2.0 but got libfoo-1.5
```
**Fix**: Adjust version constraints in `Requires` or `BuildRequires`

#### Source File Not Found
```
error: Source 'file-1.2.3.tar.gz' not found
```
**Fix**: Update Source URL in spec or use git-annex for binary files

#### Macro or Compilation Error
```
error: undefined macro %{python_sitelib}
```
**Fix**: Install macro package (e.g., `python-devel`, `foreman-rpm-macros`)

### Reading Build Logs

1. **root.log** — setup phase errors (dependencies, repos)
2. **build.log** — compilation/build phase
3. **build.log.gz** — full log output

Order of investigation:
1. Check `copr-cli status` for build state
2. Review `root.log` for dependency issues
3. Check `build.log` for compilation errors
4. Search logs for `error:` or `failed`

---

## COPR Project Workflows

### Monitoring Upstream Builds

```bash
# Watch foreman-nightly builds
copr-cli list-builds theforeman/foreman-nightly --latest 10

# Check specific Foreman/Katello release
copr-cli list-builds theforeman/foreman-3.18 | grep rubygem
```

### Scratch Builds for Testing

For packaging repositories, use scratch builds to test changes before releasing:

```bash
# Using obal with COPR backend (see obal skill)
obal scratch build PACKAGE_NAME

# Or manual COPR submission (requires --name for package name)
copr-cli build-package OWNER/PROJECT --name rubygem-example --pkgdir ./packages/foreman/rubygem-example/
```

---

## Integration with Packaging Workflows

### With git and Packaging Repos

```bash
# Update package, test via COPR
cd packages/foreman/rubygem-example

# 1. Modify spec
vim rubygem-example.spec

# 2. Trigger scratch build
copr-cli build-package theforeman/scratch-testing --pkgdir .

# 3. Monitor
copr-cli list-builds theforeman/scratch-testing --latest 1

# 4. If successful, commit and PR
git add .
git commit -m "Update rubygem-example to 1.2.4"
```

### Log Inspection Script

```bash
#!/bin/bash
BUILD_ID=$1
OWNER=${2:-theforeman}
PROJECT=${3:-foreman-nightly}

echo "=== Build $BUILD_ID Status ==="
copr-cli status $BUILD_ID

echo -e "\n=== Root Log (first errors) ==="
curl -s "https://copr.fedorainfracloud.org/results/$OWNER/$PROJECT/fedora-40-x86_64/$BUILD_ID/root.log" 2>/dev/null | grep -i error | head -5

echo -e "\n=== Build Log (last 50 lines) ==="
curl -s "https://copr.fedorainfracloud.org/results/$OWNER/$PROJECT/fedora-40-x86_64/$BUILD_ID/build.log" 2>/dev/null | tail -50
```

**Note**: Replace `fedora-40-x86_64` with the appropriate chroot (e.g., `fedora-41-x86_64`, `rhel-9-x86_64`)

---

## COPR Projects Reference

See [COPR Projects Reference](references/copr-projects.md) for detailed information on:
- Upstream Foreman & Katello projects
- Active projects and their branch mappings
- Available chroots and architectures
- Monitoring builds across releases
- Permissions and access

---

## Troubleshooting

### Issue: Build Not Starting
**Symptom**: Build stuck in "Pending" state
**Check**:
```bash
copr-cli status BUILD_ID
copr-cli list-builds OWNER/PROJECT --latest 5
```
**Solution**: COPR builders may be busy. Wait and check again later.

### Issue: "copr-cli not found"
```bash
# Install copr-cli
sudo dnf install copr-cli

# Or with pip
pip install copr-cli
```

### Issue: Cannot access project
**Symptom**: "Permission denied" or "Project not found"
```bash
# Configure credentials
mkdir -p ~/.config/copr
# Edit with your COPR token (from copr.fedorainfracloud.org)
```

### Issue: Source File Not Found in Log
```
error: Source 'rubygem-example-1.2.4.gem' not found
```
**Solution**: For binary files, use git-annex via obal (see packaging-repositories and obal skills)

### Issue: Build Timeout
**Symptom**: Build stuck for 4+ hours, eventually times out
**Check**:
```bash
copr-cli status BUILD_ID | grep time
```
**Solution**: Usually transient. Either:
- Re-trigger build: `git commit --amend --no-edit && git push --force` 
- Or wait for COPR builder backlog to clear
- If consistent timeout, check if mock build times out locally first

### Issue: Architecture-Specific Failure
**Symptom**: Build succeeds on x86_64 but fails on aarch64
```
Build succeeded on: x86_64
Build failed on:    aarch64
```
**Cause**: Usually architecture-specific code issue (pointer size, alignment, endianness)
**Debug**:
1. Check if upstream has reported the issue for your package version
2. Look for `#ifdef __aarch64__` or architecture conditionals in code
3. If minor version, might be known limitation

### Issue: Undefined Macro
```
error: undefined macro '%{foreman_bundlerd_file}'
```
**Cause**: Macro package (foreman-rpm-macros) not installed in build chroot
**Solution**: Usually COPR automatically installs required packages. File a COPR issue if macro is missing from project buildroot.

---

## Related Skills

- **obal**: Orchestrate full packaging workflows (includes COPR builds)
- **packaging-repositories**: Manage obal-based packaging repos
- **packaging-systems-thinker**: Dependency and rebuild analysis

---

## Resources

- [COPR Documentation](https://docs.pagure.org/copr.copr/)
- [COPR Web Interface](https://copr.fedorainfracloud.org/)
- [Fedora Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)
- [obal (Ansible Packaging Orchestrator)](https://github.com/theforeman/obal)

---

**Created**: 2026-06-02
**Type**: Tool/Platform
**Scope**: Upstream (Fedora, foreman-packaging, pulpcore-packaging)
