# New OS Version Workflow

Building packages for a new operating system version requires a multi-phase approach to handle bootstrap dependencies.

## Table of Contents
- [The Bootstrap Problem](#the-bootstrap-problem)
- [Solution: Bootstrap Mode](#solution-bootstrap-mode)
- [Four-Phase Tier-Based Builds](#four-phase-tier-based-builds)
- [Key Commands](#key-commands)
- [Platform Compatibility Issues](#platform-compatibility-issues)
- [Monitoring Tier Builds](#monitoring-tier-builds)
- [Rebuild Strategy](#rebuild-strategy)
- [Full Bootstrap Example](#full-bootstrap-example)
- [Common Pitfalls](#common-pitfalls)

## The Bootstrap Problem

When building for a new OS version, you face a circular dependency:
- Building packages requires the `foreman-build` package
- The `foreman-build` package comes from the `foreman` package itself
- You can't build `foreman` without `foreman-build`

## Solution: Bootstrap Mode

The solution is to temporarily enable bootstrap mode, which builds a minimal `foreman` package containing only the `foreman-build` sub-package.

### Phase 1: Enable Bootstrap

```bash
# Configure COPR project for bootstrap
obal copr-project foreman-copr -e bootstrap=true
```

This modifies the COPR project configuration to enable the `%bcond_with bootstrap` conditional in the spec file.

### Phase 2: Build Minimal Foreman

```bash
# Build foreman with bootstrap enabled
# This creates ONLY the foreman-build sub-package
obal release foreman
```

The RPM conditional in the spec file (`%if %{with bootstrap}`) limits the build to just `foreman-build`, skipping all other sub-packages.

### Phase 3: Disable Bootstrap

```bash
# Configure COPR project back to normal
obal copr-project foreman-copr -e bootstrap=false
```

Now you can build all other packages normally using the `foreman-build` package that was just created.

## Four-Phase Tier-Based Builds

After bootstrap, build packages in dependency tiers to avoid repoclosure failures.

### Tier 1: Base Dependencies

```bash
# Build foundational packages first
obal release --nowait ruby_core_packages_tier1

# Monitor builds
copr list-builds | awk '!/succeeded/' | xargs copr watch-build
```

### Tier 2: Second-Level Dependencies

```bash
# Build packages that depend on tier 1
obal release --nowait ruby_core_packages_tier2
```

### Tier 3: Parallel Independent Groups

```bash
# Multiple independent package groups can build in parallel
obal release --nowait rails_core_packages hammer_core_packages foreman_nodejs_packages
```

### Tier 4: Top-Level Packages

```bash
# Finally build the main packages
obal release --nowait foreman foreman_installer_packages katello
```

## Key Commands

### obal copr-project

Configure COPR project settings like bootstrap mode:

```bash
# Enable bootstrap
obal copr-project <project-name> -e bootstrap=true

# Disable bootstrap
obal copr-project <project-name> -e bootstrap=false
```

### obal release --nowait

Non-blocking release builds that allow parallel builds:

```bash
# Start multiple builds without waiting
obal release --nowait package1 package2 package3

# Monitor progress separately
```

### obal release --copr-rebuild

Force rebuild of an existing package:

```bash
# Rebuild without version change
obal release --copr-rebuild mypackage
```

Useful when:
- Rebuilding against updated dependencies
- Applying spec file fixes without version bump
- Testing build system changes

## Platform Compatibility Issues

When building for a new OS version, watch for these common issues:

### Ruby 3.2 Gem Compatibility

**Problem:** Many gems don't build with Ruby 3.2 due to deprecated APIs.

**Solutions:**
- Backport compatibility patches from upstream
- Use bundled versions of problematic gems
- Update gem to newer upstream version with fix

## Monitoring Tier Builds

Watch for non-succeeded builds and debug:

```bash
# List all non-succeeded builds
copr list-builds | awk '!/succeeded/'

# Watch specific build
copr watch-build BUILD_ID

# Download build logs
copr download-build BUILD_ID
```

## Rebuild Strategy

If a package fails:

```bash
# 1. Review build logs
copr download-build FAILED_BUILD_ID

# 2. Fix issue (patch, dependency, etc.)
# Edit spec file or add patches

# 3. Rebuild without version bump
obal release --copr-rebuild mypackage

# 4. Verify fix
copr watch-build NEW_BUILD_ID
```

## Full Bootstrap Example

```bash
# Step 1: Bootstrap foreman-build
obal copr-project foreman-copr -e bootstrap=true
obal release foreman
# Wait for build
obal copr-project foreman-copr -e bootstrap=false

# Step 2: Build tiers
obal release --nowait ruby_core_packages_tier1
# Monitor, wait for completion
obal release --nowait ruby_core_packages_tier2
# Monitor, wait for completion
obal release --nowait rails_core_packages hammer_core_packages
# Monitor, wait for completion
obal release --nowait foreman foreman_installer_packages

# Step 3: Final verification
copr list-builds | awk '!/succeeded/' | wc -l  # Should be 0
```

## Common Pitfalls

1. **Building tiers out of order** - Dependency failures cascade
2. **Not monitoring builds** - Silent failures delay the process
3. **Skipping repoclosure between tiers** - Broken dependencies discovered too late
4. **Not handling platform-specific issues** - Builds fail repeatedly on new OS-specific problems
