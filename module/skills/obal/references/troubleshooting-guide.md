# Troubleshooting Guide

Comprehensive troubleshooting for obal setup, build failures, authentication, and common errors.

## Table of Contents
- [Setup Issues](#setup-issues)
  - [Permission Errors](#permission-errors)
  - [Repository Detection Issues](#repository-detection-issues)
  - [Build System Configuration Issues](#build-system-configuration-issues)
- [Build Failures](#build-failures)
  - [Mock Build Fails](#mock-build-fails)
  - [Scratch Build Fails](#scratch-build-fails)
  - [Repoclosure Fails](#repoclosure-fails)
- [Authentication Issues](#authentication-issues)
  - [COPR Authentication](#copr-authentication)
  - [Brew/Koji Authentication](#brewkoji-authentication)
- [Common Errors](#common-errors)

## Setup Issues

### Permission Errors

**Error:** `PermissionError: [Errno 13] Permission denied: '/var/lib/mock'`

**Solution:** Add your user to the mock group:
```bash
sudo usermod -a -G mock $USER
newgrp mock
```

### Repository Detection Issues

**Error:** `ERROR! Unable to load package_manifest.yaml`

**Solutions:**
- Verify you're in the repository root: `pwd`
- Check file exists: `ls package_manifest.yaml`
- Verify YAML syntax: `ansible-inventory -i package_manifest.yaml --list`

**Error:** "Not in a git repository"

**Solution:** Change to the repository root directory. obal commands must be run from the repository root where `package_manifest.yaml` is located.

**Error:** "Package X not found in package_manifest.yaml"

**Solutions:**
- Look up the package: `ansible-inventory -i package_manifest.yaml --host mypackage`
- Check spelling of package name
- Verify package is defined under a group with `hosts:` section
- List all packages: `grep "^[a-zA-Z]" package_manifest.yaml`

### Build System Configuration Issues

**Error:** `Sources not found for package X`

**Context:** This is a rare edge case, typically only occurring in git-annex repositories after `git pull` or branch switches.

**Solution:** Manually fetch sources:
```bash
obal source mypackage
```

**Note:** Most workflows don't need this - `obal update --version X` automatically fetches sources.

**Error:** `git-annex: not found`

**Solution:** Install git-annex:
```bash
sudo dnf install git-annex
```

## Build Failures

### Mock Build Fails

**Symptoms:** Local mock build fails with errors.

**Troubleshooting steps:**
1. Check BuildRequires are correct and available
2. Verify Source0 URL is accessible
3. Review mock logs for specific errors (check `/var/lib/mock/<chroot>/result/`)
4. Try building the SRPM separately: `obal srpm <package>`

**Common causes:**
- Missing or incorrect BuildRequires in spec file
- Inaccessible source URL
- Spec file syntax errors
- Mock configuration issues

### Scratch Build Fails

**Symptoms:** COPR or Koji scratch build fails.

**Troubleshooting steps:**
1. Check build logs at the COPR/Brew URL (provided in obal output)
2. Verify dependencies are available in target repos
3. Run `obal repoclosure` to check missing deps
4. Compare working builds to identify what changed

**Common causes:**
- Missing dependencies in target repository
- Version conflicts with existing packages
- Build system configuration mismatch
- Network issues accessing sources

### Repoclosure Fails

**Symptoms:** Dependency verification fails.

**Troubleshooting steps:**
1. Identify which dependency is missing (shown in error output)
2. Check if the dependency needs to be built first
3. Verify repository configuration includes all needed repos
4. Update dependencies before the current package

**Common causes:**
- Dependency not yet built or released
- Wrong repository configuration
- Version mismatch between package and its dependencies

## Authentication Issues

### COPR Authentication

**Error:** `Error: Invalid API token`

**Solutions:**
- Regenerate API token at https://copr.fedorainfracloud.org/api/
- Check config file permissions: `chmod 600 ~/.config/copr`
- Verify token hasn't expired
- Verify config file exists: `~/.config/copr`
- May need API token for certain operations (configured in COPR settings)

### Brew/Koji Authentication

**Error:** `krb5.GSSError: Unspecified GSS failure`

**Solutions:**
- Obtain Kerberos ticket: `kinit username@REDHAT.COM`
- Verify ticket: `klist` (should show valid ticket)
- Check network access to Brew servers
- Renew expired ticket: `kinit -R`

**Configuration verification:**
- Check credentials are configured: `~/.koji/config` or `~/.brewkoji/config`
- Verify Kerberos ticket is valid and not expired

## Common Errors

**Error: "Ansible playbook failed"**

**Solution:** Run with `-v` for verbose output to see detailed error messages:
```bash
obal -v <action> <package>
```

This will show the full Ansible playbook execution and help identify which task failed and why.

**Error: Package definition issues**

If a package is defined incorrectly in `package_manifest.yaml`, you may see various errors. Validate the package definition:
```bash
ansible-inventory -i package_manifest.yaml --host <package-name>
```

This shows all variables and configuration for the package, helping identify misconfigurations.
