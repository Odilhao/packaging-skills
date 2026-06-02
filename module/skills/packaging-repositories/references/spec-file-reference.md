# RPM Spec File Reference

## Spec File Structure

A complete RPM spec file follows this standard structure:

```spec
# Comments start with #

# === PREAMBLE (metadata) ===
Name:           package-name
Version:        1.2.3
Release:        1%{?foremandist}%{?dist}
Summary:        One-line summary

License:        GPLv3
URL:            https://github.com/example/package
Source0:        https://upstream.example.com/package-1.2.3.tar.gz
Patch0:         fix-build.patch

# === MACROS (global definitions) ===
%global gem_name  package_name
%global minversion 2.0

# === DEPENDENCIES ===
BuildArch:      noarch
BuildRequires:  gcc
BuildRequires:  python3-devel >= %{minversion}
Requires:       python3
Recommends:     python3-optional-feature
Conflicts:      old-package

# === DESCRIPTION ===
%description
Multi-line description of what this package does.
Can span multiple lines and paragraphs.

# === SUBPACKAGES ===
%package devel
Summary: Development files for %{name}
Requires: %{name}%{?_isa} = %{version}-%{release}

%description devel
Development headers and libraries for package-name.

# === SOURCE PREP ===
%prep
%autosetup -p1  # Extract sources and apply patches

# === BUILD ===
%build
make %{?_smp_mflags}

# === INSTALL ===
%install
make install DESTDIR=%{buildroot}

# === FILES SECTION ===
%files
%license LICENSE
%doc README.md
%{_bindir}/executable
%{_libdir}/*.so*

%files devel
%{_includedir}/*.h
%{_libdir}/*.a

# === SCRIPTS (pre/post operations) ===
%post
ldconfig

%preun
systemctl stop service-name

# === CHANGELOG ===
%changelog
* Mon Jan 01 2026 Your Name <email@example.com> - 1.2.3-1
- Release package-name 1.2.3
```

---

## Section Reference

### Preamble (Metadata)

| Field | Example | Purpose |
|-------|---------|---------|
| `Name:` | `rubygem-example` | Package name (must match filename) |
| `Version:` | `1.2.3` | Upstream version number |
| `Release:` | `1%{?foremandist}%{?dist}` | Package release number (auto-incremented on rebuilds) |
| `Summary:` | `Example Ruby gem` | One-line description (max 79 chars) |
| `License:` | `GPLv3` | SPDX license identifier |
| `URL:` | `https://github.com/...` | Upstream project URL |
| `Source0:` | `https://rubygems.org/gems/...` | Primary source URL (archive, gem, wheel, etc.) |
| `Source1:` | `vendor.tar.gz` | Secondary source (usually vendor tarball for Rust) |
| `Patch0:` | `fix-build.patch` | Patch file to apply |

### Macros and Globals

```spec
# Define macros used throughout spec
%global gem_name          example        # Will be referenced as %{gem_name}
%global python_version    3.9
%global foreman_version   3.17

# Conditional macros (evaluated at build time)
Release: 1%{?foremandist}%{?dist}
#        ↑ If foremandist is set, include it; if dist is set, include it
#        Results in: "1.fc40" (Fedora 40)
```

### BuildArch

```spec
BuildArch: noarch    # Package is architecture-independent (docs, Python, Ruby)
BuildArch: x86_64    # Package is x86-64 specific (native code)
# If omitted, auto-detects based on BuildRequires and build process
```

### Dependencies

```spec
# Build-time dependencies (needed to compile)
BuildRequires: gcc
BuildRequires: python3-devel >= 3.8
BuildRequires: python3-setuptools

# Runtime dependencies (needed to run)
Requires: python3 >= 3.8
Requires: lib-example >= 2.0, lib-example < 3.0  # Version window

# Weak dependencies
Recommends:  optional-feature    # Nice to have, can be missing
Suggests:    advanced-feature    # User might want this
Conflicts:   old-incompatible    # Cannot install alongside this
Obsoletes:   replaced-package    # This replaces the old package

# Virtual dependencies (provided by other packages)
Provides: python-module(requests)
```

### %description Sections

```spec
%description
Main description of the package.

%description subpackage
Description of a subpackage.
```

### %prep (Preparation)

Common prep directives:

```spec
%autosetup -n name-%{version}  # Extract sources, auto-apply patches
%setup -q -n name-%{version}   # Extract sources (manual)
%patch0 -p1                     # Apply patch0 manually
```

### %build (Build)

```spec
%py3_build              # Build Python 3 package
%py3_wheel              # Build Python wheel (modern)
%pyproject_build        # Build pyproject.toml (modern)
gem build               # Build Ruby gem
make %{?_smp_mflags}    # Make with parallel flags (auto-use cores)
```

### %install (Installation to buildroot)

```spec
%py3_install                    # Install Python package
%pyproject_install              # Install from pyproject.toml
make install DESTDIR=%{buildroot}
mkdir -p %{buildroot}%{_bindir}
cp program %{buildroot}%{_bindir}/
```

### %files Sections

```spec
%files                      # Main package files
%license LICENSE           # Mark as license file
%doc README.md              # Documentation (not compressed in package)
%{_bindir}/executable       # Binary executables
%{_libdir}/library.so*      # Libraries (expands to .so, .so.1, etc.)
%{_libexecdir}/helper       # Non-user-visible helper binaries
%{python3_sitelib}/module   # Python site-packages

%files devel                # Development subpackage
%{_includedir}/*.h          # Headers
%{_libdir}/*.a              # Static libraries
%{_libdir}/pkgconfig/*.pc   # pkg-config files

%exclude %{python3_sitelib}/tests  # Exclude test files

# Directory ownership
%dir %{_sysconfdir}/config     # Mark directory as owned by package
```

### Special Macros

| Macro | Expands To | Example |
|-------|-----------|---------|
| `%{_bindir}` | `/usr/bin` | `/usr/bin/executable` |
| `%{_libdir}` | `/usr/lib64` (on 64-bit) | `/usr/lib64/libfoo.so` |
| `%{_includedir}` | `/usr/include` | `/usr/include/foo.h` |
| `%{_sysconfdir}` | `/etc` | `/etc/config.conf` |
| `%{_libexecdir}` | `/usr/libexec` | `/usr/libexec/helper` |
| `%{python3_sitelib}` | `/usr/lib*/python3.*/site-packages` | `/usr/lib/python3.11/site-packages/module` |
| `%{gem_dir}` | `/usr/share/gems` | `/usr/share/gems/gems/gem-1.0.0` |
| `%{?_isa}` | Architecture specifier or empty | `libfoo.so()(64bit)` or empty |

### Scripts

```spec
%prep       # Before build (extract sources, patch)
%build      # Compile/build
%install    # Install to buildroot
%check      # Run tests (optional)
%pre        # Before package installation
%post       # After package installation
%preun      # Before package uninstall
%postun     # After package uninstall
%posttrans  # After transaction completes
```

Example:

```spec
%post
ldconfig            # Update library cache
systemctl daemon-reload

%preun
systemctl stop myservice || true
```

### Changelog

```spec
%changelog
* Mon Jun 02 2026 Your Name <email@example.com> - 1.2.3-1
- Release package-name 1.2.3

* Wed May 29 2026 Your Name <email@example.com> - 1.2.2-1
- Update dependency to 2.0
- Fix crash on invalid input (refs #12345)
```

Date format: `Day Mon DD YYYY` (e.g., `Mon Jun 02 2026`)

---

## Common Patterns

### Python Package (Modern)

```spec
BuildRequires: python3-devel
BuildRequires: python3-build

%build
%pyproject_build

%install
%pyproject_install

%files
%{python3_sitelib}/mymodule
```

### Ruby Gem

```spec
%global gem_name example

BuildRequires: rubygem-devel

%build
gem build ../gem_name-%{version}.gemspec

%install
mkdir -p %{buildroot}%{gem_dir}
cp -a .%{gem_dir}/* %{buildroot}%{gem_dir}/

%files
%{gem_dir}/gems/*
%{gem_dir}/specifications/*
```

### Foreman Plugin

```spec
# template: foreman_plugin
%global foreman_version 3.17

BuildRequires: foreman-plugin >= %{foreman_version}
BuildRequires: foreman-assets >= %{foreman_version}

%install
%foreman_bundlerd_file
%foreman_precompile_plugin -s

%files
%{foreman_bundlerd_plugin}
%{foreman_assets_plugin}

%posttrans
%{foreman_plugin_log}
```

---

## Version and Release Numbering

### Version Field

- **Format**: `X.Y.Z` or upstream version convention
- **Changes when**: Upstream releases new version
- **Example**: `1.2.3`, `20230815` (date-based), `4.19.0-rc1`

### Release Field

- **Format**: `NUMBER%{?MACRO}%{?DIST}`
- **Changes when**: RPM packaging changes (new rebuild, patch, spec fix), version bumps reset to 1
- **Example**: `1%{?foremandist}%{?dist}` → expands to `1.fc40` (Fedora 40) or `1.el9` (RHEL 9)

**Release Numbering**:
- Start at `1` for new upstream version
- Increment `Release:` for rebuilds (not version changes)
- Reset to `1` when `Version:` bumps

Example progression:
```
1.2.3-1  (initial package)
1.2.3-2  (rebuild for dependency)
1.2.3-3  (spec fix, security patch)
1.2.4-1  (version bump, reset to 1)
```

---

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Wrong `Name:` (doesn't match filename) | Build fails | Name must match spec filename prefix |
| Missing `%{?dist}` in Release | Version conflicts | Always include `%{?dist}` for distro tagging |
| Files listed but not installed | Empty RPM | Add to `%install` and `%files` |
| BuildRequires missing | Build fails | Review upstream dependencies (setup.cfg, .gemspec, pyproject.toml) |
| Wrong macro for directory | Files in wrong location | Use correct `%{_bindir}`, `%{_libdir}`, etc. |
| Changelog entry without date | rpmlint fails | Always include date in proper format |

---

## References

- [Fedora Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)
- [RPM Spec File Reference](https://rpm.readthedocs.io/)
- [Red Hat Package Development](https://access.redhat.com/documentation/en-us/red_hat_software_collections/3/html/packaging_guide/)
