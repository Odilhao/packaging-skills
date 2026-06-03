# Common Packaging Scenarios

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
  - Obsoletes old version? (see Fedora Obsoletes guidelines below)
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
  - Active Fedora releases: https://docs.fedoraproject.org/en-US/releases/
  - EPEL versions: https://docs.fedoraproject.org/en-US/epel/
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

### Scenario 4: Package Removal

```markdown
Without packaging systems thinking:
→ Delete spec file and source
→ Remove from package_manifest.yaml
→ Remove from comps XML
→ Submit PR, done

With packaging systems thinking:
→ Check reverse dependencies FIRST, before writing any PR
  - Runtime reverse deps (what breaks for users):
    dnf repoquery --whatrequires '<capability>'
  - Build-time reverse deps (what breaks builds):
    dnf repoquery --whatrequires '<capability>' --source
  - Point repoquery at the target repo with --repofrompath if needed
  - These commands query actual built RPMs — they catch gem/pip auto-requires
    that come from upstream metadata (gemspec/pyproject), not from spec files.
    Grepping spec files for Requires misses auto-generated dependencies.
  - Are any co-packaged libraries still needed? (e.g., ruby2ruby needed
    ruby_parser even after safemode dropped it)
→ Update or remove dependents
  - Bump versions of packages whose upstream dropped the dependency
  - BEFORE bumping: verify the new version is installable on all target
    platforms. Check language version constraints in upstream metadata:
    - Ruby: `required_ruby_version` in gemspec (e.g., ruby2ruby 2.6.x
      needs Ruby >= 3.2, but EL9 ships Ruby 3.0 — can't upgrade)
    - Python: `python_requires` in pyproject.toml/setup.cfg
    - If the new version is incompatible, consider using %gemspec_remove_dep
      to strip the unwanted dependency from the current version instead of
      upgrading. Exclude any CLI binaries that need the stripped dep.
      See obal skill's gemspec-patching.md reference for the full pattern.
  - After updating, verify with rpm -qpR on scratch-built RPMs:
    rpm -qpR <scratch-build-url>.rpm | grep <removed-dep>
    If the removed dep still appears, the consumer needs a version bump too
→ Obsolete the package for clean upgrades
  - If the package was a runtime dependency shipped on a release branch,
    add it to the obsolete-packages spec (e.g., foreman-obsolete-packages)
  - Include both main and -doc subpackages
  - Set the Obsoletes version to the last shipped version
  - Packages that were only BuildRequires or never shipped don't need obsoleting
→ Delete from COPR/Koji
  - copr-cli delete-package to prevent stale builds
→ Remove from repo
  - Delete spec, source/annex link, package_manifest.yaml entry, comps XML entries
→ Coordinate merge order
  - New dep first → update consumers → consumer spec bumps → removal → obsolete entry
  - All PRs in the set should exist before any individual one merges
```

---

**Note on Obsoletes:** RPM Obsoletes handling (package renames, splits, merges, removals) is a complex topic with its own set of pitfalls — especially in distribution builds. When removing a package that was shipped as a runtime dependency, always add an Obsoletes entry to ensure clean upgrades for existing users. See the [Fedora Obsoletes guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/#_obsoletes) for the authoritative reference.
