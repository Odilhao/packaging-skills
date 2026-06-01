---
name: "Packaging Systems Thinking"
description: "See software packaging as interconnected systems: understand dependency graphs, version constraints, rebuild cascades, and distribution mechanics. Use when packaging software (especially RPM), managing dependencies, planning version bumps, debugging build failures, or understanding how upstream changes ripple through distributions."
---

# Packaging Systems Thinking

Shifts your perspective from **individual packages** to **the packaging ecosystem** — how packages interconnect through dependencies, how version constraints create compatibility matrices, how changes cascade through rebuild chains, and how packaging decisions affect downstream consumers.

## When to Use

- Creating or updating packages, debugging build failures, managing dependencies
- Planning releases, security patches, ABI/API changes, rebuild cascades
- **Don't use** for writing code inside a package or simple single-package operations with no dependents

---

## Core Principles

### 1. Dependencies as a Directed Graph

Every package participates in a dependency graph with build-time, runtime, and test dependencies flowing in different directions.

```
python-requests
  ├── BuildRequires (build-time):
  │   ├── python3-devel          → Compile extensions
  │   ├── python3-setuptools     → Build system
  │   └── python3-pytest         → Run tests during build
  │
  ├── Requires (runtime):
  │   ├── python3-urllib3 >= 1.21.1, < 2.0  → Version window
  │   ├── python3-chardet >= 3.0.2, < 5     → Compatibility range
  │   ├── python3-idna >= 2.5, < 4          → Upper bound for safety
  │   └── python3-certifi >= 2017.4.17      → No upper bound (stable API)
  │
  ├── Weak Dependencies:
  │   ├── Recommends: python3-urllib3[socks]  → Optional SOCKS support
  │   └── Suggests: python3-cryptography      → Enhanced security
  │
  └── Reverse Dependencies (what depends on us):
      ├── ansible-core, python3-docker, python3-boto3, 847+ more
      └── Each has its own dep tree (transitive closure)
```

**Key questions:**
- What are the build-time vs runtime deps?
- What version constraints exist and why?
- Who depends on this package (reverse deps)?
- What's the transitive closure?
- Are there circular deps or dependency diamonds?

---

### 2. Version Constraints as Compatibility Matrices

```
Epoch:Version-Release  (e.g., 1:2.31.0-3.fc40)
  │      │        │
  │      │        └── Release: packaging changes (same upstream, different package)
  │      └───────── Version: upstream version
  └───────────────── Epoch: version reset override (1:1.0 > 9999.0, epoch always wins)

Requires: libfoo >= 2.0              # Minimum version
Requires: libfoo >= 2.0, libfoo < 3  # Version window

                    libfoo 2.0  2.1  2.5  3.0
app-a (>= 2.0)          ✓      ✓    ✓    ✓
app-b (>= 2.1, < 3)     ✗      ✓    ✓    ✗
app-c (>= 2.5)          ✗      ✗    ✓    ✓
→ If all must be installable, only libfoo 2.5 satisfies all constraints.
```

**When to use Epoch:**
```
Problem: upstream switched from date-based (20230815) to semver (1.2)
         1.2 < 20230815 → users can't upgrade
Solution: 1:1.2 > 20230815 (epoch wins)
Cost: once set, epoch can never be removed — complicates comparisons forever
```

**Key questions:** What compatibility promise does this version encode? Will this update break reverse deps? Is epoch necessary? (Usually no — see [Version and Epoch Guide](references/version-and-epoch-guide.md) for alternatives.)

---

### 3. Rebuild Cascades

```
Change Type                  | Rebuild Needed?
-----------------------------|------------------------------------------
Patch / build flag change    | No (if ABI unchanged)
Minor version bump           | Possibly (check ABI)
Major version bump           | Usually (API/ABI changes)
Soname bump (libfoo.so.1→2)  | Yes — all linked packages
Python major version         | Yes — all Python packages

Example: libcurl.so.4 → .so.5
  Direct: curl, git, python3-pycurl, php-curl, wget2 + 127 more
  Transitive: git → git-lfs, tig, extensions... (~400 total)

Planning:
  1. dnf repoquery --whatrequires 'libcurl.so.4()(64bit)'
  2. Assess scope: 132 direct, ~400 transitive, ~48h build time
  3. Build in side-tag, test critical consumers, push together
```

**Key questions:** ABI-breaking, API-breaking, or compatible? How many rebuilds? Can this use a side-tag? Who needs notification?

See [Rebuild Cascade Planning](references/rebuild-cascade-planning.md) for the full planning workflow with side-tag examples.

---

### 4. Packages as Upstream-to-User Pipelines

```
┌──────────────┐
│   Upstream   │  Original source code
│   Source     │  (GitHub, releases, tarballs)
└──────┬───────┘
       │ wget/curl/spectool
       ▼
┌──────────────┐
│   Source     │  Source + patches + spec file
│   Package    │  (SRPM in RPM world)
│   (SRPM)     │
└──────┬───────┘
       │ rpmbuild / mock / koji
       ▼
┌──────────────┐
│   Binary     │  Compiled for specific arch
│   Package    │  (x86_64, aarch64, noarch)
│   (RPM)      │
└──────┬───────┘
       │ createrepo / composing
       ▼
┌──────────────┐
│   Repository │  Signed, available for download
│              │  (Fedora, EPEL, CentOS Stream)
└──────┬───────┘
       │ dnf install
       ▼
┌──────────────┐
│   Installed  │  Files on filesystem
│   System     │  Running software
└──────────────┘
```

**Key questions:**
- Where in the pipeline is the issue?
- What information is needed at each stage?
- How does a change propagate through the pipeline?
- What testing happens at each stage?

---

### 5. Packaging Trade-offs

| Decision | Pros | Cons |
|----------|------|------|
| **Bundled deps** | Exact version control, works offline | Security updates need rebuild, violates distro guidelines |
| **Unbundled deps** | Single security fix for all consumers, shared libs | Dependency resolution complexity, version conflicts |
| **Subpackages** | Smaller installs, different audiences | Added complexity |
| **Static linking** | Self-contained binary | Security updates need full rebuild, larger binary |
| **Dynamic linking** | Shared updates, smaller binaries | ABI compatibility requirements |

Split into subpackages when: large optional components, different audiences (users vs devs), licensing differences. Don't split when components are always used together.

---

## RPM-Specific Concepts

### Spec File Essentials

```spec
Name:           python-requests
Version:        2.31.0
Release:        1%{?dist}
Summary:        HTTP library for Python
License:        Apache-2.0
URL:            https://requests.readthedocs.io/
Source0:        %{pypi_source requests}
Patch1:         requests-2.31.0-certs.patch

BuildRequires:  python3-devel
BuildRequires:  python3-setuptools
BuildRequires:  python3-pytest

Requires:       python3-urllib3 >= 1.21.1
Requires:       python3-chardet >= 3.0.2
Requires:       python3-certifi
Provides:       python3dist(requests) = %{version}
Recommends:     python3-idna >= 2.5

%description
Requests is an elegant and simple HTTP library for Python.

%prep
%autosetup -n requests-%{version} -p1

%build
%py3_build

%install
%py3_install

%check
%pytest

%files
%license LICENSE
%doc README.md
%{python3_sitelib}/requests/
%{python3_sitelib}/requests-%{version}.dist-info/
```

### Provides/Requires Mechanics

```
[Automatic Dependencies]
RPM auto-detects: shared libraries (ldd), Python imports, Perl modules, pkgconfig files.
Manual deps for: feature-based requirements, version constraints, virtual provides.

[Provides vs Requires]
Package A:
  Provides: webserver, httpd = 2.4.57, libhttpd.so.2()(64bit)
Package B:
  Requires: webserver           # Any provider
  Requires: httpd >= 2.4        # Version constraint
Package C (nginx):
  Provides: webserver           # Alternative provider
  Conflicts: httpd              # Cannot coexist

[Dependency Types]
Requires       Hard runtime         BuildRequires  Build-time
Recommends     Soft (default on)    Suggests       Hint (default off)
Supplements    Reverse recommends   Enhances       Reverse suggests
Conflicts      Cannot coexist       Obsoletes      Replaces old package
```

See [RPM Dependency Mechanics](references/rpm-dependency-mechanics.md) for rich/boolean dependencies, automatic detection details, and ordering deps.

### Scriptlets and Ordering

```
Installation:   %pretrans → %pre → [files] → %post → %posttrans
Upgrade:        %pretrans(NEW) → %pre(NEW) → [NEW files] → %post(NEW)
                  → %preun(OLD) → [OLD removed] → %postun(OLD) → %posttrans(NEW)
Removal:        %preun → [files removed] → %postun

Common uses:
  %post:   %systemd_post myservice.service; /sbin/ldconfig
  %preun:  %systemd_preun myservice.service
  %postun: %systemd_postun_with_restart myservice.service

Ordering deps:
  Requires(pre):  shadow-utils   # User exists before %pre
  Requires(post): systemd        # systemd available for %post
```

### Build System Integration

```
[Build Pipeline]

Local development:
  rpmbuild -ba package.spec       # Build SRPM and RPM locally

Isolated builds:
  mock -r fedora-40-x86_64 package.src.rpm   # Clean chroot build

Production builds:
  koji build f40 package.src.rpm  # Fedora build system
  copr-cli build @theforeman/foreman package.src.rpm  # COPR build

[Mock Chroot Benefits]

Creates isolated build environment:
  - Clean dependency tree (only declared deps available)
  - Reproducible builds (no host contamination)
  - Architecture emulation (with qemu for cross-arch)
  - Catches undeclared BuildRequires
```

---

## Practical Techniques

These commands query whichever repos are enabled on the host (`/etc/yum.repos.d/`). To query a specific target (e.g., a COPR repo or a mock chroot), use `--repo=<repoid>` or `--repofrompath=<id>,<url>`.

### Analyze Dependency Trees

```bash
# What does this package require?
dnf repoquery --requires python-requests

# What requires this package?
dnf repoquery --whatrequires python-requests

# Full dependency tree (recursive)
dnf repoquery --requires --resolve --recursive python-requests

# Build dependency tree
rpm -qp --requires package.src.rpm
```

### Version Impact Analysis

```bash
# What packages have constraints on this version?
dnf repoquery --whatrequires 'python3dist(urllib3) >= 1.21'

# Check if update would break anything
dnf repoquery --whatrequires 'python3dist(urllib3) < 2.0'

# Simulate upgrade
dnf upgrade --assumeno python3-urllib3

# Check provides/requires match
rpm -qp --provides package.rpm
rpm -qp --requires dependent.rpm
```

### Rebuild Chain Planning

```bash
# Find all packages linking to a library
dnf repoquery --whatrequires 'libcurl.so.4()(64bit)'

# Find build dependencies (what needs this to build)
koji list-buildroot --newest f40-build libcurl
```

### Debug Build Failures

```bash
# Look for dependency issues in build logs
grep -i "nothing provides\|conflicts\|requires" build.log

# List what's installed in the mock chroot
mock --chroot -- rpm -qa | sort

# Run a specific command inside the chroot
mock --chroot -- rpm -q --requires python3-requests
```

See [Common Packaging Scenarios](references/scenarios.md) for before/after examples of new packages, major version updates, and security fixes.

---

## Anti-Patterns

1. **Epoch abuse** — Find the actual versioning problem; epoch is a last resort
2. **Bundling everything** — Work with distro versions, file bugs for incompatibilities
3. **Exact version pins** — `Requires: libfoo = 1.2.3-4.fc40` breaks on updates; use ranges
4. **Ignoring rebuild cascades** — Use side tags, coordinate rebuilds, test critical consumers
5. **Spec file archaeology** — Learn current guidelines; understand each line, don't cargo-cult
6. **Missing weak deps** — Use Recommends/Suggests for optional functionality

---

For Foreman/Katello repository structure, branch conventions, and git-annex workflows, see the [Repository Structure Guide](../obal/references/repository-structure.md).

## References

**Documentation:**
- [Fedora Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)
- [RPM Packaging Guide](https://rpm-packaging-guide.github.io/)

**Deep-dive references:**
- [RPM Dependency Mechanics](references/rpm-dependency-mechanics.md) — automatic deps, rich/boolean deps, virtual provides
- [Version and Epoch Guide](references/version-and-epoch-guide.md) — EVR comparison, epoch alternatives, version strategies
- [Rebuild Cascade Planning](references/rebuild-cascade-planning.md) — side-tag workflow, scope assessment, coordination
- [Common Packaging Scenarios](references/scenarios.md) — before/after examples for new packages, updates, security fixes

**Tools:** `rpmbuild`, `mock`, `koji`/`copr-cli`, `rpmlint`, `rpmdev-*`, `dnf repoquery`, `fedpkg`
