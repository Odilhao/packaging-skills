# Repository Structure

A packaging repository is a specialized Git repository designed for managing RPM packages. It contains four required components.

## The Four Components

### 1. package_manifest.yaml (Ansible Inventory)

The `package_manifest.yaml` file is an **Ansible inventory file** that defines:
- All packages in the repository
- Package metadata
- Package groups for batch operations
- Build system configuration variables

**Key point:** This is NOT a custom obal format—it's a standard Ansible inventory using YAML syntax with the same structure as any Ansible inventory (groups, hosts, variables).

### 2. packages/ Directory

Contains subdirectories for each package:
```
packages/
├── rubygem-foreman_bootdisk/
│   ├── rubygem-foreman_bootdisk.spec
│   └── foreman_bootdisk-0.1.0.gem
├── rubygem-foreman_discovery/
│   ├── rubygem-foreman_discovery.spec
│   └── foreman_discovery-1.2.0.gem
└── python-pulp-rpm/
    ├── python-pulp-rpm.spec
    └── pulp-rpm-3.18.0.tar.gz
```

### 3. .spec Files

Each package directory contains an RPM spec file defining:
- Package metadata (Name, Version, Release, Summary)
- Build and runtime dependencies
- Build instructions (%prep, %build, %install)
- File lists (%files)
- Changelog entries

### 4. Source Files

Source tarballs, gems, or other upstream artifacts. Some repositories use git-annex to manage large source files externally.

## Verification Commands

Check if you're in a packaging repository:

```bash
# Check for package_manifest.yaml
ls package_manifest.yaml

# Identify repository
basename $(pwd)

# List all packages
ansible-inventory -i package_manifest.yaml --list | \
  python3 -c "import sys, json; data=json.load(sys.stdin); \
  hosts=[h for h in data.get('_meta', {}).get('hostvars', {}).keys() if not h.endswith('-copr')]; print('\n'.join(sorted(hosts)))"
```

## Example Repository Structures

**foreman-packaging:**
```
foreman-packaging/
├── package_manifest.yaml      # Defines ~180 Ruby gems + core packages
├── packages/
│   ├── foreman/               # Core Foreman package
│   ├── rubygem-*/             # Foreman plugins and dependencies
│   └── nodejs-*/              # Node.js packages
├── .git/
└── README.md
```

**pulpcore-packaging:**
```
pulpcore-packaging/
├── package_manifest.yaml      # Defines Pulp packages
├── packages/
│   ├── python-pulpcore/
│   ├── python-pulp-rpm/
│   └── python-pulp-file/
└── .git/
```

## Common Packaging Repositories

- `foreman-packaging` - Foreman and Katello packages
- `pulpcore-packaging` - Pulp packages
- `candlepin-packaging` - Candlepin packages

## Branch Naming Conventions

**Packaging repositories (foreman-packaging, pulpcore-packaging, candlepin-packaging):**
- `rpm/X.Y` - Release branch (e.g., `rpm/3.18`)
- `rpm/develop` - Development/stream branch

Most updates happen on `rpm/develop`; branched `rpm/X.Y` branches see fewer changes.

**Core project release branches:**
- Foreman: `X.Y-stable` (e.g., `3.18-stable`)
- Katello: `KATELLO-X.Y` (e.g., `KATELLO-4.20`)
- foreman-installer: `X.Y-stable` (e.g., `3.18-stable`)
- foreman-selinux: `X.Y-stable` (e.g., `3.18-stable`)
- smart-proxy: `X.Y-stable` (e.g., `3.18-stable`)

## git-annex Workflow

Packaging repos use git-annex to manage large binary files. Always use `obal source` — never `git annex add` directly.

```bash
# CRITICAL: delete existing tarballs first — obal source silently skips if file exists
rm -f packages/<package-name>/*.tar.gz packages/<package-name>/*.tar.xz
obal source <package-name>

# Verify every source file is a symlink, not a plain file
# Expected (correct):  lrwxrwxrwx  ... package-1.0.tar.gz -> .git/annex/objects/...
# Wrong (plain file):  -rw-r--r--  ... package-1.0.tar.gz
# Note: package directory structure varies — some repos nest packages in subdirectories
ls -la packages/<package-name>/
```

**Auto-generated PR fix:** GitHub Actions automation commits tarballs as plain `100644` files, not `120000` git-annex symlinks. `obal source` silently skips files that already exist locally. Fix:

```bash
git rm --cached packages/<package-name>/<tarball-file>
rm packages/<package-name>/<tarball-file>
obal source <package-name>
```

## Typical Update Flow

```
1. Upstream gem/tarball released
         ↓
2. Update packaging repo on correct branch (rpm/X.Y or rpm/develop):
   obal update <package-name> --version <new-version>
   # This downloads the new source and updates the spec Version
         ↓
3. Open PR, review, merge
         ↓
4. After merge, COPR build is triggered automatically
```

For post-merge dependency alignment fixes (correcting stale Requires bounds without a version change), use `obal bump-release <package-name>` to increment Release, then fix the spec.
