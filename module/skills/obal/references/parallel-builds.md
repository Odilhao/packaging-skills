# Parallel Builds

Building multiple independent packages simultaneously to reduce CI time.

## Table of Contents
- [Parallel Builds](#parallel-builds)
  - [Table of Contents](#table-of-contents)
  - [When to Use Parallel Builds](#when-to-use-parallel-builds)
  - [Basic Parallel Build Pattern](#basic-parallel-build-pattern)
  - [Automated Individual Updates](#automated-individual-updates)
  - [Group Operations for Non-Version Tasks](#group-operations-for-non-version-tasks)
  - [Dependency Chain Builds (Sequential)](#dependency-chain-builds-sequential)
  - [Automated Package Updates](#automated-package-updates)
  - [Integration with update automation](#integration-with-update-automation)

## When to Use Parallel Builds

**Use parallel builds for:**
- Independent packages with no dependencies between them
- Multiple packages in different package groups
- PR validation (lint and scratch builds of changed packages)

**Do NOT use parallel builds for:**
- Packages with dependencies on each other
- Dependency chain builds (use sequential builds instead)

## Basic Parallel Build Pattern

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

## Automated Individual Updates

Real-world automation updates packages individually since each typically has a different upstream version.

**Pattern from automation:**

```python
# Pseudocode showing real automation pattern
for package in packages:
    # Each package checks its own upstream version
    upstream_version = fetch_latest_version(package)
    
    if upstream_version > current_version(package):
        # Update with specific version for this package
        run_command(f"obal update {package} --version {upstream_version} --commit")
        create_pr(package, upstream_version)
```

**Manual version update loop:**

```bash
# Update multiple packages to different versions
for pkg in rubygem-foreman_bootdisk rubygem-foreman_discovery rubygem-foreman_remote_execution; do
    # Each package updated to its own latest version
    echo "Processing $pkg..."
    obal update $pkg --version $(fetch_version $pkg) --commit
    obal lint $pkg
    obal scratch $pkg
done
```

## Group Operations for Non-Version Tasks

Group operations work well when you're NOT updating versions:

```bash
# These work well with groups (no version parameter)
obal lint foreman_core_packages
obal scratch foreman_core_packages  # Build current versions
```

**Why batch version updates are rare:**
- Each package has independent upstream versioning
- Gems, Python packages, Node.js modules all version independently
- Only coordinated releases (like Rails) might share versions
- For updates: use individual `obal update <package> --version X.Y.Z` commands

## Dependency Chain Builds (Sequential)

Build packages in dependency order when changes affect multiple layers.

**Example scenario:** Update Python base library that affects plugins.

```bash
# Step 1: Build base library first
obal scratch python-pulpcore -e build_package_archive_build_info=True
# Verify with scratch build repo
while read -r url dist; do
    obal repoclosure python-pulpcore --check "$url" --dist "$dist"
done < <(scripts/extract_copr_repos.py --bash python-pulpcore)

# Step 2: Build plugins that depend on base  
obal scratch python-pulp-rpm python-pulp-file -e build_package_archive_build_info=True
# Verify each against their scratch repos
while read -r url dist; do
    obal repoclosure python-pulp-rpm --check "$url" --dist "$dist"
done < <(scripts/extract_copr_repos.py --bash python-pulp-rpm)

while read -r url dist; do
    obal repoclosure python-pulp-file --check "$url" --dist "$dist"
done < <(scripts/extract_copr_repos.py --bash python-pulp-file)

# Step 3: Build higher-level packages
obal scratch pulpcore-selinux -e build_package_archive_build_info=True
while read -r url dist; do
    obal repoclosure pulpcore-selinux --check "$url" --dist "$dist"
done < <(scripts/extract_copr_repos.py --bash pulpcore-selinux)

# Step 4: Release in order after testing
obal release python-pulpcore
obal release python-pulp-rpm python-pulp-file
obal release pulpcore-selinux
```

**Key points:**
- Run repoclosure between layers to catch missing dependencies early
- Test each layer before proceeding to next

## Automated Package Updates

Automatically update packages when upstream releases new versions.

**Detection workflow:**

```python
# Pseudo-code
for package in packages:
    current_version = get_version_from_manifest(package)
    upstream_version = fetch_latest_upstream_version(package)

    if upstream_version > current_version:
        # Create update branch
        branch = f"bump-{package}-{upstream_version}"
        git.checkout("develop")
        git.create_branch(branch)

        # Update package
        obal_update(package, upstream_version, commit=True)

        # Verify changes
        obal_lint(package)

        # Create PR
        pr = create_pull_request(
            title=f"Bump {package} to {upstream_version}",
            body=generate_pr_body(package, current_version, upstream_version),
            branch=branch
        )

        # Trigger CI for scratch build
        trigger_ci(pr.number)
```

**GitHub Actions example:**

```yaml
name: Check Upstream Updates

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  check-updates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install obal
        run: pip install obal

      - name: Check for updates
        run: |
          # Note: obal has no --check-only flag
          # Update detection must be done by:
          # 1. Fetching package info from upstream sources
          # 2. Comparing with current spec file version
          # 3. Running create_update_prs.py with detected versions
          python scripts/detect_updates.py

      - name: Create PRs for updates
        run: python scripts/create_update_prs.py
```

## Integration with update automation

Tool monitors upstream releases and creates PRs:

```python
# Check for new upstream version
if new_version_available:
    # Create branch
    create_branch(f"bump-{package}-{version}")

    # Update package
    run_obal_update(package, version, commit=True)

    # Create PR with scratch build
    pr = create_pr(package, version)
    trigger_ci_with_scratch_build(pr)
```
