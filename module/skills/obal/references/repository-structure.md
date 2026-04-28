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
- `satellite-packaging` - Red Hat Satellite packages
- `candlepin-packaging` - Candlepin packages
