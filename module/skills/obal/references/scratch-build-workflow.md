# Scratch Build Workflow

Testing package changes with scratch builds before releasing to production.

## Table of Contents
- [What is a Scratch Build?](#what-is-a-scratch-build)
- [Basic Workflow](#basic-workflow)
- [Pattern 1: PR Validation](#pattern-1-pr-validation)
- [Standard CI Workflow](#standard-ci-workflow)
- [Monitoring Scratch Builds](#monitoring-scratch-builds)
- [Testing Scratch Build RPMs](#testing-scratch-build-rpms)
- [Build Artifact Archiving](#build-artifact-archiving)

## What is a Scratch Build?

A scratch build is a temporary test build that:
- Tests whether a package builds successfully
- Does not publish to production repositories
- Provides temporary RPMs for testing
- Can be discarded without affecting releases

## Basic Workflow

```bash
# 1. Update package
obal update mypackage --version 2.3.4

# 2. Lint first (fast error detection)
obal lint mypackage

# 3. Review changes
git diff

# 4. Scratch build
obal scratch mypackage
```

## Pattern 1: PR Validation

Validate pull requests before merging with parallel lint and scratch builds.

**Workflow:**
```bash
# 1. Checkout PR
gh pr checkout ${PR_NUMBER}

# 2. Detect changed packages
CHANGED_PKGS=$(git diff --name-only $REMOTE_NAME/$REMOTE_BRANCH_NAME | \
    grep "packages/" | cut -d'/' -f3 | sort -u)

# `-f3` for projects with `packages/in-subdrectories/package-in-subdirectory
# `-f2` for projects with `packages/package-directly-in-the-packages-directory`

# 3. Lint all changed packages in parallel
for pkg in $CHANGED_PKGS; do
    obal lint $pkg &
done
wait

# 4. Scratch build all changed packages in parallel
for pkg in $CHANGED_PKGS; do
    obal scratch $pkg &
done
wait

# 5. Run repoclosure sequentially
# After scratch builds, verify against the scratch repo (not released repos!)
for pkg in $CHANGED_PKGS; do
    # Extract repo URLs and run repoclosure for each chroot
    while read -r url dist; do
        echo "Running repoclosure for $pkg ($dist)..."
        obal repoclosure $pkg --check "$url" --dist "$dist"
    done < <(scripts/extract_copr_repos.py --bash $pkg)
done

# 6. Report results in PR comments
```

**Benefits:**
- Fast feedback on spec file issues
- Catches build failures before merge
- Verifies dependencies are satisfied
- Parallel execution reduces CI time

**When to use:** Every pull request touching packages.

## Standard CI Workflow

The standard sequence used across all packaging CI systems:

```
1. Detect Changes
2. Lint Spec Files (fast feedback)
3. Build Packages (scratch/mock)
4. Verify Dependencies (repoclosure)
5. Run Tests (if applicable)
6. Release (only on merge to main branch)
```

**Key principle:** Lint early to catch errors before expensive build operations.

**Example from theforeman/jenkins-jobs (rpm_copr_packaging.groovy):**

```groovy
stage('Detect Changes') {
    // Find modified packages via git diff
    changedPackages = sh(
        script: "git diff --name-only ${env.CHANGE_TARGET}...HEAD | grep 'packages/' | cut -d'/' -f2 | sort -u",
        returnStdout: true
    ).trim().split('\n')
}

stage('Lint') {
    parallel changedPackages.collectEntries { pkg ->
        ["Lint ${pkg}": {
            sh "obal lint ${pkg}"
        }]
    }
}

stage('Scratch Build') {
    withCredentials([file(credentialsId: 'theforeman-bot-copr', variable: 'copr_config')]) {
        parallel changedPackages.collectEntries { pkg ->
            ["Build ${pkg}": {
                sh """
                    obal scratch ${pkg} \\
                      -e build_package_build_system=copr \\
                      -e build_package_archive_build_info=True \\
                      -e build_package_copr_config=${copr_config}
                """
            }]
        }
    }
}

stage('Repoclosure') {
    changedPackages.each { pkg ->
        // copr_repos() helper from jenkins-jobs/lib/copr.groovy
        // Parses copr_build_info/${pkg} YAML and constructs repo URLs
        def repos = copr_repos(pkg)  // Returns [{url: '...', dist: 'el9'}, ...]
        
        repos.each { repo ->
            obal(
                action: "repoclosure",
                packages: pkg,
                extraVars: [
                    'repoclosure_check_repos': [repo['url']],  // Array of repo URLs
                    'repoclosure_target_dist': repo['dist']     // e.g., 'el9'
                ]
            )
        }
    }
}
```

**Bash equivalent using extract_copr_repos.py:**

The obal skill provides `extract_copr_repos.py` which replicates the Jenkins `copr_repos()` function:

```bash
for pkg in $CHANGED_PKGS; do
    while read -r url dist; do
        obal repoclosure $pkg --check "$url" --dist "$dist"
    done < <(scripts/extract_copr_repos.py --bash $pkg)
done
```

## Monitoring Scratch Builds

After submitting a scratch build, obal prints a build URL. **Always monitor the build:**

```bash
obal scratch mypackage
# Output: https://copr.fedorainfracloud.org/coprs/build/1234567/

# Click the URL and check:
# - Build logs for errors or warnings
# - All sub-packages were built
# - RPM download links are available
```

## Testing Scratch Build RPMs

Download and test RPMs from scratch builds before releasing:

```bash
# Download RPMs from COPR
copr-cli download-build 1234567

# Test installation
sudo dnf install ./mypackage-*.rpm

# Run smoke tests
mypackage --version
mypackage --help
```

## Build Artifact Archiving

Save build outputs and logs for debugging and auditing.

**Archive scratch build URLs:**

```bash
#!/bin/bash
PACKAGE=$1
ANSIBLE_LOG="builds/${PACKAGE}-$(date +%Y%m%d-%H%M%S).log"

# Capture (ansible playbook) output
obal scratch $PACKAGE 2>&1 | tee $ANSIBLE_LOG

# Extract build URL from ansible output
BUILD_URL=$(grep -oP 'https://copr.fedorainfracloud.org/coprs/build/\d+' $ANSIBLE_LOG)

# Save to artifacts
echo "$BUILD_URL" > builds/${PACKAGE}-latest-url.txt

# Archive the ansible playbook log
gzip $ANSIBLE_LOG
```

**Jenkins pipeline artifact collection:**

```groovy
stage('Build') {
    // Note: This captures ansible playbook output, not RPM build.log
    sh "obal scratch ${package} > ansible-output.log 2>&1"
    archiveArtifacts artifacts: 'ansible-output.log', fingerprint: true

    // Extract and store build URL from ansible output
    def buildUrl = sh(
        script: "grep -oP 'https://copr[^ ]+' ansible-output.log",
        returnStdout: true
    ).trim()

    currentBuild.description = "<a href='${buildUrl}'>Build</a>"
}
```

**Download RPMs from successful builds:**

```bash
# After successful scratch build
BUILD_ID=$(extract_build_id_from_url $BUILD_URL)

# Download RPMs
copr-cli download-build $BUILD_ID
```
