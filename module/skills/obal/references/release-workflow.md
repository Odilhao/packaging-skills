# Release Workflow

Complete pipeline from package update to production release.

## Table of Contents
- [Release Pipeline Stages](#release-pipeline-stages)
- [Example](#example)
- [Full Release Workflow Example](#full-release-workflow-example)
- [Multi-Distribution Releases](#multi-distribution-releases)
- [Integration with Foreman's installer testing](#integration-with-foremans-installer-testing)
- [Integration with automated testing](#integration-with-automated-testing)
- [Recovery from Failed Release](#recovery-from-failed-release)

## Release Pipeline Stages

### Stage 1: Update

```bash
# Update packages
obal update ${PACKAGE} --version ${NEW_VERSION} --commit
```

### Stage 2: Validation

```bash
# Fast checks first
obal lint ${PACKAGE}

# Local build
obal mock ${PACKAGE}

# Verify dependencies
obal repoclosure ${PACKAGE}
```

### Stage 3: Testing

```bash
# Scratch build in target environment
obal scratch ${PACKAGE}

# Manual testing or automated tests on scratch build RPMs
# Download RPMs from COPR/Koji
# Run integration tests
```

### Stage 4: Release (With Approval)

**CRITICAL:** Never release without explicit user approval. See [Safety Guardrails](../SKILL.md#safety-guardrails).

```bash
# After testing passes and approval obtained:
obal release ${PACKAGE}
```

### Stage 5: Verification

* Verify release is available
* Check repository metadata updated
* Run smoke tests against released packages

## Example

```yaml
- name: Update Package
  command: obal update {{ package_name }} --version {{ version }}

- name: Validate
  block:
    - name: Lint
      command: obal lint {{ package_name }}

    - name: Repoclosure
      command: obal repoclosure {{ package_name }}

    - name: Mock Build
      command: obal mock {{ package_name }}

- name: Scratch Build
  command: obal scratch {{ package_name }}
  register: scratch_result

- name: Manual Approval Gate
  pause:
    prompt: "Approve release of {{ package_name }}?"

- name: Release
  command: obal release {{ package_name }}
  when: approved
```

## Full Release Workflow Example

```bash
# 1. Update package (auto-generates changelog and commits to new branch)
obal update mypackage --version 2.3.4 --commit

# 2. Lint
obal lint mypackage

# 3. Verify deps
obal repoclosure mypackage

# 4. Local test
obal mock mypackage

# 5. Scratch test
obal scratch mypackage

# 6. Get explicit user approval for release
# Then: obal release mypackage
```

## Multi-Distribution Releases

Build packages for multiple target distributions (EL8, EL9).

**Method 1: Multiple COPR chroots**

```bash
# Scratch build for all distributions
obal scratch mypackage --copr-chroot rhel-8-x86_64
obal scratch mypackage --copr-chroot rhel-9-x86_64

# Release to all chroots
obal release mypackage --copr-chroot rhel-8-x86_64
obal release mypackage --copr-chroot rhel-9-x86_64
```

**Method 2: CI loop over distributions**

```bash
DISTRIBUTIONS="rhel-8 rhel-9"

for dist in $DISTRIBUTIONS; do
    echo "Building for $dist"

    # Update package manifest for target distribution
    # (some repos have dist-specific package_manifest.yaml)

    obal lint mypackage
    obal scratch mypackage --copr-chroot ${dist}-x86_64

    # Verify build succeeded
    if [ $? -eq 0 ]; then
        echo "$dist build successful"
    else
        echo "$dist build failed"
        exit 1
    fi
done
```

**Example from jenkins-jobs project:**

```groovy
def distributions = ['rhel-8', 'rhel-9']

distributions.each { dist ->
    stage("Build ${dist}") {
        parallel {
            stage("Scratch ${dist}") {
                sh "obal scratch ${package} --copr-chroot ${dist}-x86_64"
            }
        }
    }
}
```

## Integration with Foreman's installer testing

```bash
# After scratch build completes
BUILD_ID=$(extract_build_id)

# Test installation
docker run -it centos:8 bash -c "
  dnf config-manager --add-repo ${COPR_REPO_URL}
  dnf install -y mypackage
  mypackage --version
"
```

## Integration with automated testing

```yaml
- name: Build package
  command: obal scratch {{ package }}
  register: build_result

- name: Extract build URL
  set_fact:
    build_url: "{{ build_result.stdout | regex_search('https://copr[^ ]+') }}"

- name: Run installation test
  include_role:
    name: package_install_test
  vars:
    test_package: "{{ package }}"
    test_repo_url: "{{ build_url }}"
```

## Recovery from Failed Release

If a release fails partway through:

```bash
# Check what was released
copr-cli list-builds @theforeman/nightly

# Or for Koji:
koji list-builds --pattern=mypackage*
```

**Prevention:** Always use scratch builds to test before releasing.
