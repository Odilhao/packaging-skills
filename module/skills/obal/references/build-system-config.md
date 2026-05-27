# Build System Configuration

Configure which build system obal uses for package builds: COPR (default) or Koji.

## Table of Contents
- [Build System Configuration](#build-system-configuration)
  - [Table of Contents](#table-of-contents)
  - [COPR (Default)](#copr-default)
  - [Koji](#koji)
  - [Switching Between Build Systems](#switching-between-build-systems)
  - [Advanced Build Configuration](#advanced-build-configuration)
    - [Custom Copr Chroot](#custom-copr-chroot)
    - [Skip Build Checks](#skip-build-checks)

## COPR (Default)

COPR is the default build system and requires no special configuration in package_manifest.yaml.

**Default behavior:**
```bash
obal scratch mypackage   # Uses default COPR config from ~/.config/copr
obal release mypackage   # Publishes to default COPR project
```

**Custom COPR credentials:**

Use the `--copr-config` flag to specify an alternative config file:

```bash
obal scratch mypackage --copr-config /path/to/alternate-copr-config
```

This is useful when:
- Building to a different COPR project
- Using different COPR credentials
- Testing in a personal COPR project before releasing

**Example custom config:**
```ini
[copr-cli]
login = my-test-account
username = my-test-account
token = test-token-here
copr_url = https://copr.fedorainfracloud.org
```

The `--copr-config` command-line flag is ONLY for COPR builds to specify a custom credentials file. It does NOT configure Koji builds. Koji configuration is always done via package_manifest.yaml variables.

## Koji

Koji builds require configuration in package_manifest.yaml, NOT command-line flags.

**package_manifest.yaml configuration:**
```yaml
all:
  vars:
    build_package_build_system: koji
    build_package_koji_command: koji
```

**Then run normal commands:**
```bash
obal scratch mypackage   # Builds in Koji
obal release mypackage   # Releases to Koji
```

**Authentication:** Requires certificates in `~/.koji/config` (see [Authentication Setup](authentication.md)).

## Switching Between Build Systems

To build with Koji instead of COPR, modify the group variables in package_manifest.yaml:

```yaml
all:
  vars:
    build_package_build_system: koji
    build_package_koji_command: koji
```

Then run normal obal commands—they will automatically use Koji:

```bash
obal scratch mypackage   # Uses Koji because of package_manifest.yaml config
obal release mypackage   # Uses Koji
```

## Advanced Build Configuration

### Custom Copr Chroot

Specify target chroot for COPR builds:

```bash
# Target specific chroot/architecture
obal release <package> --copr-chroot rhel-10-x86_64
```

**Common chroots:**
- `rhel-8-x86_64` - RHEL 8 x86_64
- `rhel-9-x86_64` - RHEL 9 x86_64
- `rhel-10-x86_64` - RHEL 10 x86_64

**When to use:**
- Targeting specific distribution versions
- Testing compatibility with different OS versions
- Building for specific architectures

**Note:** This flag only works with COPR builds. Koji target selection is controlled through package_manifest.yaml variables.

### Skip Build Checks

**Use with extreme caution:**

```bash
# Skip koji whitelist check
obal scratch <package> --skip-koji-whitelist-check
```

**When to use:**
- Package not yet whitelisted in Koji
- Testing in development environment
- With explicit approval from maintainers

**Never use unless you understand the implications.** Skip flags bypass safety checks that prevent invalid builds. Using skip flags can result in:
- Builds that violate project policies
- Packages that cannot be imported or released
- Wasted build resources on invalid configurations

Always prefer fixing the underlying issue (e.g., adding package to whitelist) rather than bypassing checks.
