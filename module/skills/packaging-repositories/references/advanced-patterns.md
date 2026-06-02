# Advanced Packaging Patterns

Complex scenarios and advanced patterns found in foreman-packaging.

---

## Epoch Versioning

### When and Why

Use Epoch when the normal version ordering would be broken (e.g., switching from date-based to semver).

```spec
Epoch: 1
Version: 3.105.5
Release: 1%{?dist}
```

**Example problem:** Package versioning switches from `20230815` (date) to `1.0.0` (semver):
- Without Epoch: `1.0.0 < 20230815` (numeric comparison) → users can't upgrade
- With Epoch: `1:1.0.0 > 20230815` (epoch takes precedence) → upgrade works

### The Epoch Rule (Critical)

**Epoch is permanent. Once set, it can never be removed.**

Implications:
- Version comparisons will ALWAYS include epoch (`1:1.2.3` vs `0:2.0.0`)
- Downstream systems inherit the epoch permanently
- Removes complexity in version comparisons

### Usage in foreman-packaging

Example: `rubygem-pulpcore_client`
```spec
Epoch: 1
Version: 3.105.5
Release: 1%{?dist}
```

Reason: Prevents downgrade when pulpcore versioning scheme changed.

### When NOT to Use Epoch

- **Semantic versioning**: semver is designed for clear ordering
- **Switching to higher version number**: e.g., `2.9.9` → `3.0.0` (doesn't need epoch)
- **Minor version changes**: Just increment version normally

---

## Conditional Dependencies

### Requires with Conditions

Specify that a dependency is only needed in certain contexts:

```spec
Requires: (python3-requests if (ansible-core >= 1:2.14.7 and ansible-core < 1:2.16.14-3))
```

This means: "Only require python3-requests when ansible-core is between 2.14.7 and 2.16.14-3"

### Use Cases

1. **Version-specific dependencies:**
   ```spec
   Requires: (feature-package if some-tool >= 2.0)
   ```

2. **Optional compatibility layers:**
   ```spec
   BuildRequires: (compat-lib if redhat >= 8)
   ```

### How It Works

- DNF/RPM evaluates conditions during dependency resolution
- If condition is met, dependency is required
- If condition is not met, dependency is optional

---

## Satellite-Specific Builds (Downstream Branching)

### Detecting Downstream Builds

foreman-packaging maintains both upstream (Foreman) and downstream (Red Hat Satellite) builds.

Detect which context you're building in:

```spec
%global downstream_build ("%{?dist}" == ".el8sat" || "%{?dist}" == ".el9sat")

%if %{downstream_build}
# Red Hat Satellite-specific code path
BuildRequires: satellite-specific-package
Requires: satellite-feature
%else
# Upstream (Foreman) code path
BuildRequires: upstream-package
%endif
```

### Dist Tag Indicators

| Dist Tag | Context | Example |
|----------|---------|---------|
| `.fc40` | Fedora 40 (Upstream) | `1.el9.fc40` |
| `.el9` | RHEL 9 (Upstream) | `1.el9` |
| `.el8sat` | RHEL 8 with Satellite | `1.el8sat` |
| `.el9sat` | RHEL 9 with Satellite | `1.el9sat` |

### Components in packages/satellite/

Some packages are Satellite-specific and only exist in the `packages/satellite/` directory:

```
packages/satellite/
├── foreman_theme_satellite/
├── ansible-collection-redhat-satellite/
└── ansible-collection-redhat-satellite_operations/
```

These packages are only built for downstream (Satellite) contexts.

---

## Vendor Tarballs for Rust-Based Python Packages

### The Problem

Python packages with Rust extensions (e.g., cryptography, nh3) require:
1. Rust compiler + cargo
2. Rust dependencies (build-time only)

Building on-the-fly in mock/COPR is slow and complex.

### The Solution: Vendor Tarballs

Pre-build the Rust dependencies into a single "vendor tarball" and store it as Source1.

```spec
Source0: https://files.pythonhosted.org/packages/.../cryptography-43.0.1.tar.gz
Source1: https://downloads.theforeman.org/vendor/cryptography-43.0.1-vendor.tar.gz

%prep
%autosetup -p1 -n cryptography-%{version}
# Source1 extracted here, registered as vendor directory
```

### Generating a Vendor Tarball

```bash
# 1. Fetch the upstream source
pip download --no-binary :all: --no-deps cryptography==43.0.1

# 2. Extract and prepare
tar xzf cryptography-43.0.1.tar.gz
cd cryptography-43.0.1

# 3. Create vendor directory with all dependencies
mkdir -p vendor
cd vendor

# Fetch all Rust crates (dependencies)
cargo vendor --locked

# 4. Tar it up
tar czf ../cryptography-43.0.1-vendor.tar.gz .

# 5. Upload to CDN (theforeman.org/vendor/)
# Then reference in spec as Source1
```

### Key Points

- **Source0**: Original upstream tarball (PyPI)
- **Source1**: Pre-built vendor dependencies (CDN)
- Speeds up build significantly (no network download during build)
- Only needed for Rust-extension packages
- Always use `obal source` to manage git-annex symlinks for both

---

## Software Collections (Legacy)

For supporting multiple Python or Ruby versions (rare in modern packaging).

```spec
# template: scl
%global scl_name python37
%{?scl:%scl_package rubygem-%{gem_name}}

Name: %{?scl_prefix}rubygem-%{gem_name}
Version: 1.2.3
Release: 1%{?dist}

%scl_require rubygem-json
```

Creates packages like:
- `rh-python37-rubygem-example` (for RHEL Python 3.7)
- `rh-python39-rubygem-example` (for RHEL Python 3.9)

**Note**: Modern distributions prefer single-version approach. SCL patterns are legacy.

---

## Template Variations

### Available Templates in gem2rpm

foreman-packaging provides several spec templates for different package types:

| Template | Use For | Example |
|----------|---------|---------|
| `foreman_plugin` | Foreman UI plugins | foreman_rh_cloud |
| `smart_proxy_plugin` | Smart Proxy extensions | dhcp, puppetca |
| `scl` | Software collection versioning | python37 variant |
| `default.spec.erb` | Regular Ruby gems | most gems |

Template specified as comment in spec:
```spec
# template: foreman_plugin
```

---

## Advanced Branch Patterns

### Automated Bump Branches

Foreman uses automation to create version bump branches:

```
bump_rpm/rubygem-example       # Ruby gem version bump
bump_deb/python-example        # Python package version bump
revert-123                     # Revert for failed change
release-katello-4.19.0         # Tagged release branch
```

These branches are created automatically by CI/CD systems and contain:
- Updated spec file (Version, Release reset)
- Updated changelog entry
- Often includes automatic PR creation

### Branch Protection Rules

Main branches (`rpm/develop`, `rpm/3.18`, etc.) are protected:
- Require PR for any changes
- CI checks must pass
- Code review required before merge

---

## Performance Tuning

### Parallel Make Builds

Use parallel compilation when possible:

```spec
%build
make %{?_smp_mflags}
# or
cmake --build . --parallel %{?_smp_mflags}
```

`%{?_smp_mflags}` expands to `-j8` (or appropriate core count).

### Skip Tests in Builds

Some packages take too long to test; conditionally skip:

```spec
%global skip_tests 0

%check
%if ! %{skip_tests}
pytest
%endif
```

Can be disabled with `--define 'skip_tests 1'` when building.

---

## Distribution-Specific Build Configuration

### Different chroots, different requirements

```spec
# Only require systemd on newer systems
%if 0%{?rhel} >= 8 || 0%{?fedora} >= 30
BuildRequires: systemd-rpm-macros
%endif

# Use different package names on different distros
%if 0%{?fedora}
BuildRequires: python3-devel
%elif 0%{?rhel} >= 8
BuildRequires: python3-devel
%else
BuildRequires: python-devel
%endif
```

### Macro Reference

| Macro | Value | Example |
|-------|-------|---------|
| `%{?fedora}` | Fedora version or undefined | `40` in Fedora 40, undefined in RHEL |
| `%{?rhel}` | RHEL version or undefined | `9` in RHEL 9, undefined in Fedora |
| `%{?dist}` | Distribution tag | `.fc40`, `.el9`, `.el8sat` |

---

## Macro Definition Best Practices

### Global Macros

Use `%global` for macros that should be immutable:

```spec
%global gem_name          example
%global foreman_version   3.17
%global plugin_enabled    1
```

### Conditional Macros

Use `%define` for context-dependent definitions:

```spec
%define buildlevel alpha   # Can be overridden at build time
```

### Macro Naming

Follow conventions:
- `%{gem_name}` — Ruby gem name
- `%{python_version}` — Python version
- `%{version}` — Upstream version (built-in)
- `%{release}` — RPM release (built-in)

---

## Multi-Package Builds (Subpackages)

Large projects often split into multiple packages:

```spec
Name: example
Version: 1.0.0
Release: 1%{?dist}

# Main package
%description
Core example package

# -devel subpackage
%package devel
Summary: Development files for example

%description devel
Headers and static libraries for development

# -docs subpackage
%package doc
Summary: Documentation for example
BuildArch: noarch

%description doc
API reference and guides

# Install sections can differ per subpackage
%files
%{_bindir}/example

%files devel
%{_includedir}/example.h
%{_libdir}/libexample.a

%files doc
%{_docdir}/example/
```

Each subpackage becomes a separate RPM:
- `example-1.0.0-1.fc40.x86_64.rpm`
- `example-devel-1.0.0-1.fc40.x86_64.rpm`
- `example-doc-1.0.0-1.fc40.noarch.rpm`

---

## See Also

- [Package Types Guide](package-types-guide.md) — Ruby, Python, Node.js, Go packages
- [Spec File Reference](spec-file-reference.md) — Complete spec syntax
- [Fedora Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)
