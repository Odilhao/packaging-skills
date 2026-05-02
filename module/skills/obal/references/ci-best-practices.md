# CI/CD Best Practices

General guidelines for effective packaging CI workflows.

## Table of Contents
- [1. Lint Early and Often](#1-lint-early-and-often)
- [2. Parallel Builds When Possible](#2-parallel-builds-when-possible)
- [3. Use Scratch Builds for Testing](#3-use-scratch-builds-for-testing)
- [4. Verify Dependencies After Updates](#4-verify-dependencies-after-updates)
- [5. Monitor Build Logs](#5-monitor-build-logs)
- [6. Archive Build Artifacts](#6-archive-build-artifacts)
- [7. Use Meaningful Commit Messages](#7-use-meaningful-commit-messages)
- [8. Version Verification](#8-version-verification)
- [9. Build Error Recovery](#9-build-error-recovery)
- [10. Common Patterns](#10-common-patterns)
- [11. Use Group Operations](#11-use-group-operations)
- [Summary Checklist](#summary-checklist)

## 1. Lint Early and Often

Run `obal lint` before any build operation:

```bash
# Good: Lint catches errors in 5 seconds
obal lint mypackage && obal scratch mypackage

# Bad: Wait 10 minutes for scratch build to fail on lint error
obal scratch mypackage
```

**Why:** Linting is fast (~5 seconds) and catches most spec file issues. Building is slow (5-15 minutes) and expensive.

## 2. Parallel Builds When Possible

Build independent packages in parallel to reduce CI time:

```bash
# Serial: 30 minutes total
obal scratch package1  # 10 min
obal scratch package2  # 10 min
obal scratch package3  # 10 min

# Parallel: 10 minutes total
obal scratch package1 &
obal scratch package2 &
obal scratch package3 &
wait
```

**Limitation:** Don't parallelize dependent packages—build in dependency order instead (see [Parallel Builds](parallel-builds.md)).

## 3. Use Scratch Builds for Testing

Never release without scratch build verification:

```bash
# CI workflow
obal lint mypackage
obal scratch mypackage
# Manual test of scratch build RPMs
# Only after testing passes:
obal release mypackage
```

**Why:** Scratch builds are temporary and safe for testing. Releases publish to production repositories.

## 4. Verify Dependencies After Updates

**CRITICAL:** Always run repoclosure after package updates.

```bash
obal update mypackage --version 2.3.4
obal repoclosure mypackage  # Verify dependencies
```

**Why:** Version updates often change dependencies. Repoclosure verifies all BuildRequires and Requires are available before release builds.

**See:** [Repoclosure Usage](repoclosure-usage.md) for detailed patterns.

## 5. Monitor Build Logs

Always check build logs even for successful builds:

```bash
obal scratch mypackage
# Open build URL (COPR/Koji web interface)
# Check for warnings in the RPM build logs
# Verify all sub-packages built
```

**What to look for:**
- Compiler warnings
- Deprecated API usage
- Missing optional dependencies
- Sub-packages that failed silently

## 6. Archive Build Artifacts

Save build logs and URLs for debugging:

```bash
# Create builds/ directory
mkdir -p builds

# Archive each build
obal scratch mypackage 2>&1 | tee builds/mypackage-$(date +%Y%m%d).log
```

**Benefits:**
- Debug failures that can't be reproduced locally
- Track build history
- Share build URLs with team
- Audit trail for releases

## 7. Use Meaningful Commit Messages

When using `--commit` flag, obal auto-generates changelog and commit message. Review and amend if needed:

```bash
obal update mypackage --version 1.2.3 --commit

# Obal commits with message like:
# "Update mypackage to 1.2.3"

# If more context needed, amend:
git commit --amend -m "Update mypackage to 1.2.3

Includes fix for CVE-2023-12345
Related: https://github.com/project/issues/123"
```

## 8. Version Verification

Verify version updates were applied correctly before building:

```bash
#!/bin/bash
PACKAGE=$1
EXPECTED_VERSION=$2

# Extract version from spec file
SPEC_FILE="packages/${PACKAGE}/${PACKAGE}.spec"
ACTUAL_VERSION=$(grep "^Version:" $SPEC_FILE | awk '{print $2}')

if [ "$ACTUAL_VERSION" != "$EXPECTED_VERSION" ]; then
    echo "ERROR: Version mismatch!"
    echo "Expected: $EXPECTED_VERSION"
    echo "Actual: $ACTUAL_VERSION"
    exit 1
fi

echo "Version verified: $ACTUAL_VERSION"
```

## 9. Build Error Recovery

**Retry with verbose output:**

```bash
# Failed build
obal scratch mypackage

# Retry with Ansible verbose mode
obal -v scratch mypackage

# Maximum verbosity
obal -vvv scratch mypackage
```

## 10. Common Patterns

**Scratch builds with metadata:**
```bash
obal scratch mypackage -e build_package_archive_build_info=True
```

**Repoclosure after scratch builds:**
```bash
while read -r url dist; do
    obal repoclosure mypackage --check "$url" --dist "$dist"
done < <(scripts/extract_copr_repos.py --bash mypackage)

# Alternative: use extra variables directly
obal repoclosure mypackage \
  --check https://download.copr.fedorainfracloud.org/results/@user/project/chroot \
  --dist el9
```

**Extra variables syntax:**
```bash
# Inline key=value
obal scratch mypackage -e build_package_build_system=copr
```

From file, `vars.yaml:`
```yaml
repoclosure_check_repos:
  - https://download.copr.fedorainfracloud.org/results/@user/project/chroot
repoclosure_target_dist: el9
```

```bash
obal scratch mypackage -e @vars.yaml
```

**See also:** [Scratch Build Workflow](scratch-build-workflow.md) for real-world CI patterns.

## 11. Use Group Operations

Automate repetitive tasks with group operations:

```bash
# Lint entire group
obal lint foreman_core_packages

# Scratch build entire group
obal scratch katello_packages
```

**When to use groups:**
- Linting related packages together
- Building packages in batch
- Testing multiple packages simultaneously

**When NOT to use groups:**
- Updating to specific versions (each package has its own version - use `obal update <package> --version X.Y.Z`)
- Dependency chain builds (use sequential builds instead)

## Summary Checklist

Before releasing any package:

- [ ] Lint passed
- [ ] Mock build succeeded (optional but recommended)
- [ ] Scratch build succeeded
- [ ] Repoclosure passed
- [ ] Build logs reviewed for warnings
- [ ] Test installation of scratch build RPMs
- [ ] User approval obtained for release
