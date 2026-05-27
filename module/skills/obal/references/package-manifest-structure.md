# Package Manifest Structure

The `package_manifest.yaml` file is a standard **Ansible inventory** organized hierarchically with groups and hosts.

## Table of Contents
- [Basic Structure](#basic-structure)
- [Hierarchical Group Structure](#hierarchical-group-structure)
- [Group Operations](#group-operations)
- [Package Fields](#package-fields)
- [Build System Configuration Variables](#build-system-configuration-variables)
- [Listing All Packages](#listing-all-packages)
- [Advanced Package Operations](#advanced-package-operations)

## Basic Structure

**Real example from foreman-packaging:**

```yaml
# Top-level structure
all:
  vars:
    # Global variables apply to all packages
    copr_project_user: "@theforeman"
    build_package_build_system: copr
  children:
    packages: {}
    copr_projects: {}

# Package hierarchy
packages:
  children:
    foreman_packages: {}
    plugin_packages: {}

foreman_core_packages:
  children:
    rails_core_packages: {}
  hosts:
    # Some packages have source information
    foreman:
      source_location: https://ci.theforeman.org/job/foreman-develop-source-release
      source_system: jenkins
      
# Most packages have empty metadata - actual metadata is in .spec files
rails_core_packages:
  hosts:
    rubygem-actioncable: {}  # Empty - all metadata in .spec file
    rubygem-activesupport: {}
    rubygem-actionpack: {}
```

**Key points:**
- Most packages are defined as empty hosts (`{}`) - version, release, dependencies are in the .spec file
- Some packages (like `foreman`) include `source_location` and `source_system` for automated source fetching
- Groups are organized hierarchically using `children`
- Individual packages appear under `hosts`

## Hierarchical Group Structure

```
all/
├── foreman_packages/
│   ├── foreman_core_packages/
│   │   ├── foreman
│   │   └── foreman-installer
│   ├── foreman_ruby_packages/
│   │   ├── rubygem-foreman_bootdisk
│   │   └── rubygem-foreman_discovery
│   └── foreman_nodejs_packages/
│       ├── nodejs-webpack
│       └── nodejs-babel
└── katello_packages/
    ├── katello_ruby_packages/
    └── katello_python_packages/
```

## Group Operations

Since package_manifest.yaml is an Ansible inventory, obal can operate on entire groups:

**Build all packages in a group:**
```bash
obal scratch foreman_ruby_packages
```

**Lint all packages in a group:**
```bash
obal lint foreman_core_packages
```

**Common group operations:**
- Coordinated builds of package sets
- Linting multiple packages at once
- **Note:** Batch version updates are rare (each package typically has independent versions - use `obal update <package> --version X.Y.Z`)

**Advanced/Rare:** In git-annex repositories, you can fetch sources for all packages in a group with `obal source <group>` (e.g., after `git pull`), but most workflows don't need this as `obal update --version X` automatically handles source fetching.

**Finding available groups:**
```bash
# List all group names
ansible-inventory -i package_manifest.yaml --list | \
  python3 -c "import sys, json; data=json.load(sys.stdin); \
  groups=[g for g in data.keys() if g != '_meta']; print('\n'.join(sorted(groups)))"
```

## Package Fields

**Important:** Most packages in package_manifest.yaml are defined as empty hosts (`{}`). Package metadata (version, release, dependencies, etc.) is stored in the .spec file, not in package_manifest.yaml.

**Common patterns:**

```yaml
# Most packages - empty definition
rubygem-actioncable: {}
rubygem-activesupport: {}

# Some packages - with source information for automated fetching
foreman:
  source_location: https://ci.theforeman.org/job/foreman-develop-source-release
  source_system: jenkins
```

**Optional fields** (rarely used, most metadata is in .spec):
- `source_location`: URL for automated source fetching
- `source_system`: Source system type (`jenkins`, `github`, etc.)
- **Version, release, dependencies**: Always in .spec file, NOT in package_manifest.yaml

## Build System Configuration Variables

Variables controlling build behavior are set at the group level:

```yaml
all:
  vars:
    # Which build system to use
    build_package_build_system: copr  # Options: copr, koji
    
    # COPR-specific settings
    copr_project_user: "@theforeman"  # COPR project owner
    
    # Koji-specific settings
    build_package_koji_command: koji

# COPR project configuration
copr_projects:
  hosts:
    foreman-copr:
      copr_project_name: "foreman-nightly-staging"
      copr_project_chroots:
        - name: "rhel-9-x86_64"
```

## Listing All Packages

```bash
# List all packages (excludes COPR projects ending with -copr)
ansible-inventory -i package_manifest.yaml --list | \
  python3 -c "import sys, json; data=json.load(sys.stdin); \
  hosts=[h for h in data.get('_meta', {}).get('hostvars', {}).keys() if not h.endswith('-copr')]; \
  print('\n'.join(sorted(hosts)))"
```

## Advanced Package Operations

### Custom Release Numbers

Override the default release number (1) when updating:

```bash
# Set custom release number
obal update <package> --version 1.2.3 --release 2
```

**When to use:**
- Rebuilding package without version change
- Adding patches to existing version
- Fixing packaging bugs in released version

**Why it matters:** The release number distinguishes different builds of the same upstream version. Incrementing release (1→2→3) allows you to publish updated packages without changing the upstream version number. This is essential when:
- Backporting patches to a stable version
- Fixing RPM packaging issues (spec file bugs, dependency problems)
- Rebuilding against updated dependencies

For example, if you discover a bug in the spec file for `mypackage-1.2.3-1`, you can fix the spec and rebuild as `mypackage-1.2.3-2` without waiting for a new upstream release.

### Prerelease Versions

Tag packages with prerelease identifiers:

```bash
# Mark as release candidate
obal update <package> --version 1.2.3 --prerelease rc1
```

**Common prerelease tags:**
- `alpha1`, `alpha2` - Early testing
- `beta1`, `beta2` - Feature-complete testing
- `rc1`, `rc2` - Release candidates

**Why it matters:** Prerelease tags ensure test versions sort before stable releases in package managers, preventing users from accidentally installing unstable versions. 

For example:
- `mypackage-1.2.3-0.1.rc1` sorts **before** `mypackage-1.2.3-1` 
- Users with automatic updates won't accidentally get the release candidate
- The `0.1` release number ensures the final release (`-1`) supersedes all prereleases

The version format becomes: `<version>-0.<N>.<prerelease>` where `<N>` is incremented for each prerelease build. When the final version is released, it uses release number `1` and automatically supersedes all `0.x` prereleases.

**Verify version ordering:**

Use `rpmdev-vercmp` (if available) to confirm your prerelease versions sort correctly:

```bash
# Verify prerelease sorts before stable release
rpmdev-vercmp 1.2.3-0.1.rc1 1.2.3-1
# Output: 1.2.3-0.1.rc1 < 1.2.3-1
# Exit code: 12 (first version is older)

# Verify multiple prereleases sort correctly
rpmdev-vercmp 1.2.3-0.1.rc1 1.2.3-0.2.rc2
# Output: 1.2.3-0.1.rc1 < 1.2.3-0.2.rc2
# Exit code: 12 (first version is older)
```

**Exit codes:**
- `0` - versions are equal
- `11` - first version is newer (>)
- `12` - first version is older (<)

This helps catch version ordering issues before building packages, and the exit codes are useful for automation scripts.
