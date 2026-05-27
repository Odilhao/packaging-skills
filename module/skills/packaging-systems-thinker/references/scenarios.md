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

---

**Note on Obsoletes:** RPM Obsoletes handling (package renames, splits, merges) is a complex topic with its own set of pitfalls — especially in distribution builds. See the [Fedora Obsoletes guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/#_obsoletes) for the authoritative reference.
