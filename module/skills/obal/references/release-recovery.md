# Release Recovery

How to recover when the nightly release pipeline fails or a bad package version is released.

## Table of Contents
- [Manual COPR Release](#manual-copr-release)
- [Dependency Ordering](#dependency-ordering)
- [COPR Build Cleanup](#copr-build-cleanup)
- [NVR Downgrade Handling](#nvr-downgrade-handling)
- [Pipeline Behavior](#pipeline-behavior)

## Manual COPR Release

When the nightly release pipeline (`foreman_packaging_rpm_copr_release`) fails, packages that were supposed to be released may be missing from COPR staging. You can manually release them:

```bash
cd /path/to/packaging-repo
obal source <package-name>
obal release <package-name>
```

**When this happens:**
- A dependency was merged and released in the wrong order
- The pipeline failed partway through and didn't reach all packages
- A package was merged but the pipeline hasn't run yet

## Dependency Ordering

When manually releasing packages, **release dependencies before consumers**. The nightly pipeline doesn't guarantee dependency ordering — it may try to build a consumer before its dependency is available in the repo.

**Example from safemode 2.0 migration:**
```bash
# ruby2ruby must be in COPR before foreman can build against it
obal release rubygem-ruby2ruby    # dep first
obal release foreman              # consumer second
```

**General rule:** If package A requires package B, release B first. Check with:
```bash
rpm -qpR <package-A-scratch-build>.rpm | grep <package-B-capability>
```

## COPR Build Cleanup

When a bad version is released to COPR, **delete it before releasing the fix**. COPR keeps all builds and dnf picks the highest NVR (Name-Version-Release). If the bad version has a higher NVR than the fix, dnf will keep installing the bad version.

**Delete via COPR web UI:**
1. Go to `https://copr.fedorainfracloud.org/coprs/@theforeman/foreman-nightly-staging/builds/`
2. Find the bad build
3. Click delete

**Delete via CLI:**
```bash
copr-cli delete-build <build-id>
```

**Example:** ruby2ruby 2.6.1-1 was released but requires Ruby >= 3.2 (incompatible with EL9). The fix is 2.5.2-2 with `%gemspec_remove_dep`. Since `2.6.1-1 > 2.5.2-2` by NVR, the 2.6.1 build must be deleted first.

## NVR Downgrade Handling

When reverting a version bump that was already released, the new NVR may be lower than the released NVR. RPM considers `2.5.2-2 < 2.6.1-1`, so users won't get the downgrade automatically.

**Options (in order of preference):**

1. **Delete old COPR build** — preferred for nightly/staging repos. Remove the bad build so dnf only sees the correct version. No permanent packaging burden.

2. **Add Epoch** — `Epoch: 1` makes any version win over epoch-0 packages. But Epoch is permanent and can never be removed. Use only as last resort.

3. **Wait for next version bump** — if the upstream will release a new version soon that fixes the issue, skip the downgrade and wait.

**CI limitation:** The PR CI check uses `rpmdev-vercmp` to compare the PR branch version against the base branch. It doesn't handle version downgrades — it sees a version change with release != 1 and fails:
```
Version updated but release was not reset back to 1 for <package>
```
This is expected for revert PRs. Maintainers can override the CI failure.

## Pipeline Behavior

The nightly release pipeline (`foreman_packaging_rpm_copr_release` in `foreman-jenkins-jobs`) runs after merges to rpm/develop. It:

1. Detects changed packages by comparing git hashes
2. Builds SRPMs for each changed package
3. Submits them to COPR as release builds
4. Waits for builds to complete

**Key limitation:** The pipeline builds all changed packages but does **not** order them by dependency. If package A depends on package B, and both changed:
- B may not be available in the repo when A's build starts
- A's build fails with unresolved dependencies
- Fix: manually release B first, then re-trigger the pipeline (or manually release A)

**Re-triggering:** The pipeline runs on merge events. To re-trigger without a new merge, ask a maintainer to replay the Jenkins job, or manually release the remaining packages with `obal release`.
