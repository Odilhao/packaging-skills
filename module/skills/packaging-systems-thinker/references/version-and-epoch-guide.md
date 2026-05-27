# Version Comparison and Epoch Guide

Detailed reference for RPM version comparison rules, epoch usage, and version constraint strategies.

## RPM Version Comparison (EVR)

```
Epoch:Version-Release  (e.g., 1:2.31.0-3.fc40)

Comparison order:
  1. Epoch (integer, default 0 if absent) — always compared first
  2. Version (upstream version string)
  3. Release (packaging iteration)

Each field is compared segment by segment:
  - Numeric segments: compared as integers (9 < 10)
  - Alpha segments: compared lexicographically (a < b)
  - Tildes (~) sort BEFORE anything: 1.0~rc1 < 1.0
  - Carets (^) sort AFTER anything: 1.0^post1 > 1.0
```

## Semantic Versioning in RPM Context

```
MAJOR.MINOR.PATCH  (e.g., 2.31.0)
  │     │     │
  │     │     └── Patch: bug fixes, no API changes — safe to auto-update
  │     └──────── Minor: new features, backwards compatible — safe within major
  └────────────── Major: breaking changes — requires testing, may need code changes

Note: not all upstreams follow semver. Some use date-based (20230815),
single-number (45), or custom schemes. Check upstream's actual compatibility promise.
```

## When Epoch Is Necessary

```
Scenario: upstream switched versioning schemes

BEFORE: 20230115, 20230401, 20230815  (date-based)
AFTER:  1.0, 1.1, 1.2                (semver)

Problem: RPM says 1.2 < 20230115 → users can't upgrade via dnf

Solution: set Epoch: 1
  1:1.2 > 20230815  (epoch wins regardless of version string)

Cost:
  - Once set, epoch can NEVER be removed
  - Must increment epoch for future versioning resets
  - Complicates every version comparison forever
  - All reverse deps must use epoch in version constraints
  - Confusing for users (rpm -q shows epoch)
```

## Alternatives to Epoch

Before reaching for epoch, try:

1. **Version mangling in spec**: if upstream uses `v1.2`, strip the `v` prefix
2. **Pre-release markers**: use `~` tilde for pre-releases: `1.0~rc1 < 1.0`
3. **Post-release markers**: use `^` caret: `1.0^post1 > 1.0`
4. **Obsoletes**: if truly renaming, `Obsoletes: old-name < last-old-version`

## Compatibility Matrix Example

```
                    libfoo 2.0  2.1  2.5  3.0
app-a (>= 2.0)          ✓      ✓    ✓    ✓
app-b (>= 2.1, < 3)     ✗      ✓    ✓    ✗
app-c (>= 2.5)          ✗      ✗    ✓    ✓
app-d (= 2.1)           ✗      ✓    ✗    ✗

If all four apps must be installable simultaneously,
only libfoo 2.5 satisfies all constraints.

This is why exact version pins (app-d) are dangerous:
they reduce the solution space to a single point.
```
