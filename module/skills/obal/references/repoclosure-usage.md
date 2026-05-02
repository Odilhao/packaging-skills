# Repoclosure Usage

Dependency verification to ensure all package dependencies are satisfied.

## Table of Contents
- [What is Repoclosure?](#what-is-repoclosure)
- [Critical Requirement: --dist Flag](#critical-requirement---dist-flag)
- [Basic Usage (Without --check)](#basic-usage-without---check)
- [Configuration](#configuration)
- [Local Scratch Builds (Mock)](#local-scratch-builds-mock)
- [Remote COPR Scratch Builds](#remote-copr-scratch-builds)
- [Real CI Pattern (Automated)](#real-ci-pattern-automated)
- [Dependency Chain Verification](#dependency-chain-verification)
- [Common Repoclosure Failures](#common-repoclosure-failures)
- [When to Run Repoclosure](#when-to-run-repoclosure)
- [Why Repoclosure Matters](#why-repoclosure-matters)

## What is Repoclosure?

Repoclosure checks whether:
- All `BuildRequires` dependencies can be satisfied
- All `Requires` (runtime dependencies) can be satisfied
- Dependencies are available from specified repositories

## Critical Requirement: --dist Flag

**CRITICAL:** When using the `--check` flag with repoclosure, you MUST specify `--dist`.

```bash
# WRONG - will fail
obal repoclosure mypackage --check /path/to/repo

# RIGHT - specifies distribution
obal repoclosure mypackage --check /path/to/repo --dist <target-dist>  # e.g., el9, el8, el10
```

**Why --dist is required with --check:** The `--check` flag tells repoclosure to verify packages from a custom repository URL (like a COPR scratch build or local mock repository). Without `--dist`, repoclosure cannot determine which distribution's dependency metadata to use. Different OS versions (el8 vs el9 vs el10) have different available packages in their base repositories. The `--dist` flag specifies which OS version's repository metadata to check against, ensuring repoclosure validates dependencies correctly for your target platform.

## Basic Usage (Without --check)

Verify dependencies from default repositories:

```bash
obal repoclosure mypackage
```

This checks dependencies against the repositories configured in `package_manifest.yaml` under `repoclosure_target_repos`. 

**When to use bare repoclosure (without --check):**
- Testing packages already **released** to production repositories
- Verifying dependencies for packages in staging/stable repos
- Checking released package compatibility

**When to use repoclosure with --check:**
- After **scratch builds** (COPR) - to verify the newly built RPMs
- After **mock builds** - to verify locally built RPMs
- Testing packages **before** they're released

## Configuration

The target distribution is configured in `package_manifest.yaml`:

```yaml
all:
  vars:
    repoclosure_target_repos:
      el9:
        - "el9-foreman-client-{{ foreman_version }}-staging"
      el8:
        - "el8-foreman-client-{{ foreman_version }}-staging"
```

Some package groups may also specify a default:
```yaml
my_package_group:
  vars:
    repoclosure_target_dist: <dist>  # e.g., el9  # Default when not specified via --dist
```

## Local Scratch Builds (Mock)

After building locally with mock, results are in `mock_builds/results/<dist>`:

```bash
# 1. Mock build
obal mock mypackage

# 2. Repoclosure with local build
obal repoclosure mypackage --check ./mock_builds/results/el9 --dist <target-dist>  # e.g., el9, el8, el10
```

## Remote COPR Scratch Builds

### Method 1: Variables File

```bash
# 1. Scratch build with metadata archiving
obal scratch mypackage -e build_package_archive_build_info=True

# 2. Create vars.yaml with COPR URL
# vars.yaml:
repoclosure_check_repos:
  - https://download.copr.fedorainfracloud.org/results/@theforeman/katello-nightly-staging-scratch-<uuid>/rhel-9-x86_64
repoclosure_target_dist: <dist>  # e.g., el9

# 3. Run repoclosure
obal repoclosure mypackage -e @vars.yaml
```

### Method 2: CLI Flags

```bash
obal repoclosure mypackage \
  --check https://download.copr.fedorainfracloud.org/results/@theforeman/katello-nightly-staging-scratch-<uuid>/rhel-9-x86_64/ \
  --dist <target-dist>  # e.g., el9, el8, el10
```

## Real CI Pattern (Automated)

Use the `extract_copr_repos.py` helper script (see [details below](#helper-script-extract_copr_repospy)) to automatically extract repository URLs from scratch build metadata.

```bash
# 1. Scratch build with metadata archiving
obal scratch mypackage -e build_package_archive_build_info=True

# 2. Extract and run repoclosure for each chroot
while read -r url dist; do
    echo "Running repoclosure for $dist..."
    obal repoclosure mypackage --check "$url" --dist "$dist"
done < <(scripts/extract_copr_repos.py --bash mypackage)
```

**Single chroot shorthand:**

```bash
read -r url dist < <(scripts/extract_copr_repos.py --bash mypackage)
obal repoclosure mypackage --check "$url" --dist "$dist"
```

### Helper Script: extract_copr_repos.py

The `scripts/extract_copr_repos.py` script replicates the Jenkins `copr_repos()` function from foreman-jenkins-jobs, parsing YAML build metadata to extract repository URLs and distribution information.

**Features:**
- Handles multiple chroots in a single build
- Converts chroot names to distribution format (e.g., `rhel-9-x86_64` → `el9`)
- Supports JSON or bash-friendly output formats
- Error handling for missing or malformed files

**Usage:**

```bash
# JSON output (default)
scripts/extract_copr_repos.py mypackage
# Returns: [{"url": "https://...", "dist": "el9", "chroot": "rhel-9-x86_64"}, ...]

# Bash-friendly output (one repo per line: URL DIST)
scripts/extract_copr_repos.py --bash mypackage
# Returns: https://... el9

# Quiet mode (suppress errors if file not found)
scripts/extract_copr_repos.py --quiet mypackage
```

**Chroot to dist conversions:**
- `rhel-8-x86_64` → `el8`
- `centos-stream-9-x86_64` → `el9`
- `opensuse-leap-15.4-x86_64` → `leap154`

### Understanding build_package_archive_build_info

When you pass `-e build_package_archive_build_info=True` to `obal scratch`:

1. obal creates a `copr_build_info/` directory
2. For each package, it creates a metadata file: `copr_build_info/<package-name>`
3. The file contains COPR build information including the build URL
4. CI scripts can parse this file to get the repository URL for repoclosure

**Example metadata file content:**

```
changed: true
msg: All items completed
results:
- ansible_loop_var: chroot
  build_urls: ['https://copr.fedorainfracloud.org/coprs/build/<build-id>']
  builds: ['<build-id>']
  changed: true
  chroot: rhel-9-x86_64
  failed: false
  invocation:
    module_args: {chroot: rhel-9-x86_64, config_file: null, force: false, project: <project>-nightly-staging-scratch-<uuid>,
      srpm: /tmp/ansible.<random>/<package-name>-<version>-<release>.src.rpm,
      user: '@<copr-user>', wait: false}
  output: "Uploading package /tmp/ansible.<random>/<package-name>-<version>-<release>.src.rpm\nBuild
    was added to <project>-nightly-staging-scratch-<uuid>:\n
    \ https://copr.fedorainfracloud.org/coprs/build/<build-id>\nCreated builds: <build-id>\n"
skipped: false
```

## Dependency Chain Verification

When updating packages with dependencies, verify each layer:

```bash
# Step 1: Build base library
obal scratch python-pulpcore
obal repoclosure python-pulpcore  # Verify base deps

# Step 2: Build plugins that depend on base
obal scratch python-pulp-rpm python-pulp-file
obal repoclosure python-pulp-rpm python-pulp-file

# Step 3: Build higher-level packages
obal scratch pulpcore-selinux
obal repoclosure pulpcore-selinux
```

## Common Repoclosure Failures

### Missing Dependency

**Error:**
```
Error: Nothing provides python3-foo >= 2.0
```

**Solutions:**
1. Build or update the missing dependency first
2. Check if the dependency is available in the target repository
3. Verify the version requirement is correct in the spec file

### Wrong Distribution

**Error:**
```
Error: Cannot find repository metadata for el8
```

**Solution:** Verify `--dist` matches your target distribution:
```bash
obal repoclosure mypackage --check <repo> --dist <target-dist>  # e.g., el9, el8, el10
```

## When to Run Repoclosure

**After every package update:**
```bash
obal update mypackage --version 2.3.4
obal repoclosure mypackage  # Verify new version's dependencies
```

**Before releasing:**
```bash
obal lint mypackage
obal scratch mypackage
obal repoclosure mypackage  # Final dependency check
obal release mypackage  # Only after repoclosure passes
```

**In dependency chain builds:**
```bash
# Build base, verify deps, then build dependents
obal scratch base-package
obal repoclosure base-package
obal scratch dependent-package
obal repoclosure dependent-package
```

## Why Repoclosure Matters

Version updates often change dependencies. Repoclosure catches:
- Missing dependencies before production builds
- Version mismatches between packages
- Broken dependency chains
- Repository configuration issues

**Always run repoclosure before releasing** to ensure users can install your packages.
