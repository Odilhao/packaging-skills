---
name: "Packaging Systems Thinking"
description: "See software packaging as interconnected systems: understand dependency graphs, version constraints, rebuild cascades, and distribution mechanics. Use when packaging software (especially RPM), managing dependencies, planning version bumps, debugging build failures, or understanding how upstream changes ripple through distributions."
---

# Packaging Systems Thinking

## What This Skill Does

Shifts your perspective from **individual packages** to **the packaging ecosystem**. You'll see how packages interconnect through dependencies, how version constraints create compatibility matrices, how changes cascade through rebuild chains, and how packaging decisions affect downstream consumers.

This mindset changes:
- **Dependency analysis** - See build vs runtime, direct vs transitive, required vs optional
- **Version planning** - Understand compatibility windows, epoch triggers, and breaking changes
- **Change assessment** - Map rebuild cascades before making changes
- **Distribution understanding** - See how packages flow from upstream through builds to users

---

## When to Use This Skill

Apply packaging systems thinking when:
- **Creating new packages** - Understanding what belongs together, how to split subpackages
- **Updating packages** - Assessing impact on dependents, planning rebuild chains
- **Debugging build failures** - Tracing through dependency trees to find root causes
- **Managing dependencies** - Deciding between bundling vs unbundling, version pinning
- **Planning releases** - Coordinating multi-package updates, avoiding broken states
- **Security patches** - Understanding which packages need rebuilds, backporting strategies
- **ABI/API changes** - Managing soname bumps, compatibility layers

**Don't use when:**
- Writing code inside a package (focus on code, not packaging)
- Simple single-package operations with no dependents
- Writing upstream application code (different concerns than distribution packaging)

---

## Core Principles

### 1. See Dependencies as a Directed Graph

**Shift from:**
"This package needs these libraries to build"

**To:**
"This package participates in a dependency graph with build-time, runtime, and test dependencies flowing in different directions"

**Example:**
```
Component view:
  python-requests needs urllib3, chardet, idna

Packaging systems view:
  python-requests
    ├── BuildRequires (build-time):
    │   ├── python3-devel          → Needed to compile extensions
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
        ├── python3-docker          → Inherits our version constraints
        ├── python3-boto3           → May conflict with our deps
        ├── ansible-core            → Critical consumer
        └── 847 more packages...    → Ecosystem impact

System behaviors:
  - Updating urllib3 to 2.x breaks python-requests
  - Updating python-requests upper bound requires testing 847 packages
  - Removing chardet requires all consumers to adapt
  - Each dependency has its own dependency tree (transitive closure)
```

**Questions to ask:**
- What are the build-time vs runtime dependencies?
- What version constraints exist and why?
- Who depends on this package (reverse dependencies)?
- What's the transitive closure of dependencies?
- Are there circular dependencies or dependency diamonds?

---

### 2. Understand Version Constraints as Compatibility Matrices

**Version numbers encode compatibility promises:**

```
[Semantic Versioning Interpretation]

MAJOR.MINOR.PATCH  (e.g., 2.31.0)
  │     │     │
  │     │     └── Patch: Bug fixes, no API changes
  │     │               Safe to update automatically
  │     │
  │     └──────── Minor: New features, backwards compatible
  │               Safe for same major version
  │
  └────────────── Major: Breaking changes
                  Requires testing, may need code changes

[RPM Version Comparison]

Epoch:Version-Release  (e.g., 1:2.31.0-3.fc40)
  │      │        │
  │      │        └── Release: Packaging changes (patches, spec fixes)
  │      │                     Same upstream, different package
  │      │
  │      └───────── Version: Upstream version
  │                 Usually follows upstream scheme
  │
  └───────────────── Epoch: Version reset override
                     Nuclear option for version ordering
                     1:1.0 > 9999.0 (epoch always wins)

[Version Constraint Examples]

Requires: libfoo >= 2.0     # Minimum version
Requires: libfoo < 3.0      # Maximum version (exclude)
Requires: libfoo >= 2.0, libfoo < 3.0  # Version window
Requires: libfoo = 2.31.0-3.fc40  # Exact version (fragile!)

[Compatibility Matrix]

                    libfoo 2.0  2.1  2.5  3.0
app-a (>= 2.0)          ✓      ✓    ✓    ✓
app-b (>= 2.1, < 3)     ✗      ✓    ✓    ✗
app-c (>= 2.5)          ✗      ✗    ✓    ✓
app-d (= 2.1)           ✗      ✓    ✗    ✗

System constraint: If all apps must be installable,
only libfoo 2.5 satisfies all constraints!
```

**When to use Epoch:**
```
BEFORE: package versions were 20230115, 20230401, 20230815 (date-based)
AFTER:  upstream switched to 1.0, 1.1, 1.2 (semver)

Problem: 1.2 < 20230115 (string comparison)
         Users can't upgrade!

Solution: Epoch bump
         1:1.2 > 20230815 (epoch wins)

Cost: Once set, epoch can never be removed
      Must increment for future resets
      Complicates version comparisons forever
```

**Questions to ask:**
- What compatibility promise does this version encode?
- What's the version constraint window for each dependency?
- Will this update break any reverse dependencies?
- Is an epoch bump necessary? (Usually no, find another way)

---

### 3. Map Rebuild Cascades Before Making Changes

**When you change a package, dependent packages may need rebuilding:**

```
[Rebuild Trigger Analysis]

Change Type                  | Rebuild Needed?
-----------------------------|------------------------------------------
Patch in source              | No (if ABI unchanged)
Build flag change            | No (if ABI unchanged)
Minor version bump           | Possibly (check ABI)
Major version bump           | Usually (API/ABI changes)
Soname bump (libfoo.so.1→2)  | Yes (all linked packages)
Python major version         | Yes (all Python packages)
Compiler major version       | Yes (all packages, often)

[Example: Library Soname Bump]

libcurl.so.4 → libcurl.so.5 (ABI break)

Direct rebuilds needed:
  ├── curl (CLI tool)
  ├── git (uses libcurl)
  ├── python3-pycurl (Python bindings)
  ├── php-curl (PHP bindings)
  ├── wget2 (uses libcurl)
  └── 127 more packages...

Each of those may trigger further rebuilds:
  git rebuild →
    ├── git-lfs (built against git)
    ├── tig (git dependency)
    └── Many git extensions...

[Cascade Planning]

Before bumping soname:

1. Query reverse dependencies:
   $ dnf repoquery --whatrequires 'libcurl.so.4()(64bit)'

2. Assess rebuild scope:
   - Direct deps: 132 packages
   - Transitive: ~400 packages
   - Build time: ~48 hours (parallel build system)
   - Human time: 2-3 days coordinating

3. Plan the update:
   Phase 1: Build libcurl (tagged, not pushed)
   Phase 2: Side-tag with rebuilt dependents
   Phase 3: Test critical packages (git, ansible)
   Phase 4: Push all together

4. Coordinate:
   - Notify package maintainers
   - File bugs for FTBFS (fails to build from source)
   - Plan for packages needing patches
```

**Questions to ask:**
- What type of change is this? (ABI-breaking, API-breaking, compatible)
- How many packages need rebuilding?
- Can this be done in a side-tag?
- What's the testing strategy for dependents?
- Who needs to be notified?

---

### 4. See Packages as Upstream-to-User Pipelines

**Software flows through multiple stages:**

```
[Package Lifecycle Pipeline]

┌──────────────┐
│   Upstream   │  Original source code
│   Source     │  (GitHub, releases, tarballs)
└──────┬───────┘
       │ wget/curl
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

[Information Flow at Each Stage]

Upstream → SRPM:
  - Which version to package?
  - What patches to apply?
  - How to build (configure flags)?
  - What dependencies?

SRPM → RPM:
  - What architecture?
  - Which compiler/toolchain?
  - What build environment (mock config)?
  - Reproducible builds?

RPM → Repository:
  - Which repository/tag?
  - Signing key?
  - Compose timing?
  - Koji vs COPR vs local?

Repository → System:
  - Which repos enabled?
  - Version locks?
  - Module streams?
  - Conflicts with local packages?
```

**Questions to ask:**
- Where in the pipeline is the issue?
- What information is needed at each stage?
- How does a change propagate through the pipeline?
- What testing happens at each stage?

---

### 5. Understand Packaging Decisions Have Trade-offs

**Common packaging trade-offs:**

```
[Bundling vs Unbundling]

Bundled dependencies (vendor/):
  ✓ Exact version control
  ✓ No external dep resolution
  ✓ Works offline after download
  ✗ Security updates require package rebuild
  ✗ Duplication wastes disk/memory
  ✗ Different packages may bundle conflicting versions
  ✗ Violates distro guidelines (usually)

Unbundled (system libraries):
  ✓ Single security update fixes all consumers
  ✓ Shared libraries = less memory
  ✓ Follows distro guidelines
  ✗ Dependency resolution complexity
  ✗ Version constraint conflicts
  ✗ Upstream may not test with system versions

[Subpackage Decisions]

When to split into subpackages:

my-app                    # Main package (always needed)
my-app-devel              # Headers, pkg-config (developers only)
my-app-doc                # Documentation (optional)
my-app-libs               # Shared libraries (may be shared)
my-app-python             # Python bindings (optional)
my-app-plugins-database   # Database plugins (optional feature)

Split when:
  - Large optional components
  - Different audiences (users vs developers)
  - Licensing differences
  - Reduces install size for most users

Don't split when:
  - Components always used together
  - Adds complexity without benefit
  - Small size difference

[Static vs Dynamic Linking]

Static linking:
  ✓ Self-contained binary
  ✓ No library version issues
  ✗ Security updates require full rebuild
  ✗ Larger binary size
  ✗ Can't share memory across processes

Dynamic linking:
  ✓ Shared library updates fix all users
  ✓ Smaller individual binaries
  ✓ Memory shared between processes
  ✗ Dependency management complexity
  ✗ ABI compatibility requirements
```

**Questions to ask:**
- Should this dependency be bundled or unbundled?
- Is a subpackage warranted for this component?
- What are the maintenance costs of each approach?
- What do distro guidelines say?

---

## RPM-Specific Concepts

### Spec File as System Definition

```spec
# A spec file defines the entire package system

Name:           python-requests
Version:        2.31.0
Release:        1%{?dist}
Summary:        HTTP library for Python
License:        Apache-2.0
URL:            https://requests.readthedocs.io/

# Source location (upstream relationship)
Source0:        %{pypi_source requests}

# Patches (downstream modifications)
Patch1:         requests-2.31.0-certs.patch

# Build environment requirements
BuildRequires:  python3-devel
BuildRequires:  python3-setuptools
BuildRequires:  python3-pytest

# Runtime requirements (with version constraints)
Requires:       python3-urllib3 >= 1.21.1
Requires:       python3-chardet >= 3.0.2
Requires:       python3-certifi

# What this package provides (for others to depend on)
Provides:       python3dist(requests) = %{version}

# Weak dependencies
Recommends:     python3-idna >= 2.5
Suggests:       python3-urllib3+socks

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

### Understanding Provides/Requires

```
[Automatic Dependencies]

RPM automatically detects:
  - Shared library requirements (ldd analysis)
  - Python imports (python-rpm-generators)
  - Perl modules (perl.req)
  - pkgconfig files

Manual dependencies for:
  - Feature-based requirements
  - Version constraints beyond auto-detection
  - Virtual provides

[Provides vs Requires]

Package A:
  Provides: webserver
  Provides: httpd = 2.4.57
  Provides: libhttpd.so.2()(64bit)  # Auto-generated

Package B:
  Requires: webserver              # Any package providing 'webserver'
  Requires: httpd >= 2.4           # Version constraint
  Requires: libhttpd.so.2()(64bit) # Specific library

Package C (alternative):
  Provides: webserver              # nginx also provides webserver
  Conflicts: httpd                 # Can't coexist with httpd

[Dependency Types]

Requires:       Hard runtime dependency
BuildRequires:  Build-time dependency
Recommends:     Soft dependency (installed by default)
Suggests:       Hint (not installed by default)
Supplements:    Reverse recommends
Enhances:       Reverse suggests
Conflicts:      Cannot be installed together
Obsoletes:      Replaces old package (auto-remove)
```

### Scriptlets and Ordering

```
[Scriptlet Execution Order]

Installation of NEW package:
  %pretrans -p <lua>        # Before transaction starts
  %pre                      # Before files installed
  [files installed]
  %post                     # After files installed
  %posttrans -p <lua>       # After transaction completes

Upgrade (old → new):
  %pretrans of NEW
  %pre of NEW
  [NEW files installed]
  %post of NEW
  %preun of OLD            # Before old files removed
  [OLD files removed]
  %postun of OLD           # After old files removed
  %posttrans of NEW

Removal:
  %preun
  [files removed]
  %postun
  %posttrans (never runs on removal)

[Common Scriptlet Uses]

%post
  # Register with system services
  %systemd_post myservice.service

  # Update shared library cache
  /sbin/ldconfig

%preun
  # Stop service before removal
  %systemd_preun myservice.service

%postun
  # Restart on upgrade, nothing on removal
  %systemd_postun_with_restart myservice.service

[Scriptlet Ordering with Dependencies]

Requires(pre):    shadow-utils    # Ensures user exists before %pre
Requires(post):   systemd         # Ensures systemd available for %post
OrderWithRequires: yes            # Scriptlet ordering follows deps
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
  - Clean dependency tree
  - Reproducible builds
  - No host contamination
  - Architecture emulation (with qemu)

[Koji/COPR Tags]

Tag = Collection of builds with policy

f40-candidate          # New builds land here
f40-pending            # Passed CI, awaiting push
f40-updates            # Pushed to stable repo
f40-updates-testing    # In testing repo

Build flow:
  Build → candidate → testing → updates
                         ↓
                    (karma/time)

[Side Tags for Coordinated Builds]

# Create side tag for rebuild chain
$ fedpkg request-side-tag

# Build multiple packages
$ koji build f40-build-side-12345 libfoo.src.rpm
$ koji build f40-build-side-12345 app-using-libfoo.src.rpm

# Merge when ready
$ koji merge-side-tag f40-build-side-12345 f40-candidate
```

---

## Practical Techniques

### Technique 1: Analyze Dependency Trees

```bash
# What does this package require?
$ dnf repoquery --requires python-requests
python3-certifi
python3-chardet
python3-idna
python3-urllib3

# What requires this package?
$ dnf repoquery --whatrequires python-requests
ansible-core
python3-docker
python3-keystoneauth1
...

# Full dependency tree (recursive)
$ dnf repoquery --requires --resolve --recursive python-requests

# Build dependency tree
$ rpm -qp --requires package.src.rpm

# Check for dependency loops
$ rpmdep --cycles packages.txt
```

### Technique 2: Version Impact Analysis

```bash
# What packages have constraints on this version?
$ dnf repoquery --whatrequires 'python3dist(urllib3) >= 1.21'

# Check if update would break anything
$ dnf repoquery --whatrequires 'python3dist(urllib3) < 2.0'

# Simulate upgrade
$ dnf upgrade --assumeno python3-urllib3

# Check provides/requires match
$ rpm -qp --provides package.rpm
$ rpm -qp --requires dependent.rpm
```

### Technique 3: Rebuild Chain Planning

```bash
# Find all packages linking to library
$ dnf repoquery --whatrequires 'libcurl.so.4()(64bit)'

# Find build dependencies (what needs this to build)
$ koji list-buildroot --newest f40-build libcurl

# Estimate rebuild scope
$ fedora-rebuild-helper estimate libcurl

# Create side tag for rebuild
$ fedpkg request-side-tag --base-tag f40-build-side
```

### Technique 4: Debug Build Failures

```bash
# Reproduce build locally
$ fedpkg clone package && cd package
$ fedpkg mockbuild

# Check build logs for dependency issues
$ grep -i "nothing provides\|conflicts\|requires" build.log

# Enter failed build environment
$ mock --shell

# Compare successful vs failed buildrep
$ diff <(rpm -qa --root /old-buildroot) <(rpm -qa --root /new-buildroot)
```

---

## Common Packaging Scenarios

### Scenario 1: New Package

```markdown
Without packaging systems thinking:
→ Copy spec from similar package
→ Adjust name/version
→ Add obvious dependencies
→ Build, fix errors until it works
→ Done

With packaging systems thinking:
→ Analyze upstream's dependencies
  - What does it actually need?
  - What versions are compatible?
  - What's bundled that shouldn't be?
→ Map to existing distro packages
  - Which deps exist?
  - Which need packaging first?
  - Circular dependency issues?
→ Design package structure
  - Main package vs subpackages?
  - -devel package needed?
  - Documentation package?
→ Define compatibility window
  - Minimum versions for deps
  - Maximum versions (if API breaks)
→ Consider maintenance burden
  - How active is upstream?
  - Who maintains in distro?
  - Security response process?
→ Build and test in clean environment
→ Review against packaging guidelines
→ Plan for future updates
```

### Scenario 2: Major Version Update

```markdown
Without packaging systems thinking:
→ Update Version: in spec
→ Download new source
→ Build, fix obvious issues
→ Submit

With packaging systems thinking:
→ Read upstream changelog
  - API changes?
  - Removed features?
  - New dependencies?
→ Check downstream impact
  - Reverse dependencies?
  - Rebuild requirements?
  - Compatibility breaks?
→ Assess version constraints
  - Do consumers pin our version?
  - Will they build against new version?
→ Plan transition
  - Side tag for coordinated rebuild?
  - Parallel installable (soname bump)?
  - Compatibility package needed?
→ Update spec with proper changes
  - Patch removals (may be upstreamed)
  - New dependencies
  - Obsoletes old version?
→ Build all affected packages
→ Test critical consumers
→ Coordinate push timing
```

### Scenario 3: Security Fix

```markdown
Without packaging systems thinking:
→ Cherry-pick upstream fix
→ Bump release number
→ Build and push quickly

With packaging systems thinking:
→ Understand the vulnerability
  - What's the attack vector?
  - What versions affected?
  - What's the fix scope?
→ Assess rebuild requirements
  - Is this library statically linked anywhere?
  - Bundled in other packages?
  - Header-only changes affecting ABI?
→ Check multiple streams
  - Active Fedora releases
  - EPEL versions
  - RHEL (if applicable)
→ Plan coordinated fix
  - Same CVE number all branches
  - Same changelog text
  - Proper advisory text
→ Verify fix works
  - Build in clean environment
  - Reproducer (if available)
  - Regression tests
→ Push with urgency appropriate to severity
  - Critical: skip testing, push immediately
  - Important: expedited testing
  - Moderate: normal flow
```

---

## Anti-Patterns to Avoid

### 1. Epoch Abuse
**Problem:** Using epoch to "fix" version comparison issues
**Solution:** Find the actual versioning problem. Epoch should be last resort.

### 2. Bundling Everything
**Problem:** Vendoring all dependencies to avoid conflicts
**Solution:** Work with distro versions, file bugs for incompatibilities

### 3. Exact Version Pins
**Problem:** `Requires: libfoo = 1.2.3-4.fc40` breaks on updates
**Solution:** Use version ranges that express actual compatibility

### 4. Ignoring Rebuild Cascades
**Problem:** Pushing ABI-breaking change without rebuilding dependents
**Solution:** Use side tags, coordinate rebuilds, test critical consumers

### 5. Spec File Archaeology
**Problem:** Copying ancient spec patterns without understanding
**Solution:** Learn current guidelines, use modern macros, understand each line

### 6. Missing Weak Dependencies
**Problem:** Hard requiring optional features
**Solution:** Use Recommends/Suggests for optional functionality

---

## Foreman/Katello Packaging Repositories

### Repository Layout

| Repository | Branch Pattern | Purpose |
|-----------|---------------|---------|
| `theforeman/foreman-packaging` | `rpm/X.Y` | Foreman and Katello RPM packages |
| `theforeman/pulpcore-packaging` | `main` | Pulp server packages |

### Branch Naming Conventions

**foreman-packaging:**
- `rpm/X.Y` - Foreman X.Y release branch (e.g., `rpm/3.18`)
- `rpm/develop` - Development/stream branch

**Core project release branches:**
- Foreman: `X.Y-stable` (e.g., `3.18-stable`)
- Katello: `KATELLO-X.Y` (e.g., `KATELLO-4.20`)
- foreman-installer: `X.Y-stable` (e.g., `3.18-stable`)

### git-annex Workflow for Gems/Tarballs

Packaging repos use git-annex to manage large binary files:

```bash
# Download new gem
curl -LO https://rubygems.org/gems/package-1.2.3.gem

# Add to git-annex
git annex add package-1.2.3.gem

# Remove old gem
git rm package-1.2.2.gem

# Commit
git add package-1.2.3.gem
git commit -m "Update package to 1.2.3"
```

### Typical Update Flow

```
1. Upstream gem/tarball released
         ↓
2. Update foreman-packaging (rpm/X.Y branch)
   - Download new source
   - Update spec file (Version, changelog)
   - git-annex add/rm
         ↓
3. PR merged, build triggered in Koji/COPR
         ↓
4. Tag into release candidate tag
```

---

## Related Mindsets

Combine packaging systems thinking with:
- **Software Systems Thinking** - Understand how packaged software systems interact
- **Meticulous** - Thorough dependency analysis, complete testing
- **Security Thinking** - Supply chain security, vulnerability response
- **Maintenance Mindset** - Long-term supportability, upgrade paths

---

## Resources

### Documentation
- [Fedora Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)
- [RPM Packaging Guide](https://rpm-packaging-guide.github.io/)
- [Maximum RPM](http://ftp.rpm.org/max-rpm/)
- [Fedora Release Engineering](https://docs.fedoraproject.org/en-US/fesco/)

### Tools
- `rpmbuild` - Build RPMs from spec files
- `mock` - Build in clean chroot
- `koji` / `copr-cli` - Build system integration
- `rpmlint` - Package linting
- `rpmdev-*` - RPM development tools
- `dnf repoquery` - Query package metadata
- `fedpkg` - Fedora packaging workflow

### Concepts
- Dependency graphs
- Semantic versioning
- ABI/API compatibility
- Soname versioning
- Reproducible builds
- Supply chain security

---

**Remember:** Every package exists within an ecosystem. Changes ripple through dependency graphs, rebuild chains, and user systems. Think beyond the single spec file to understand the full impact of your packaging decisions.
