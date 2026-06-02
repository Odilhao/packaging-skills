# COPR Projects Reference

## Upstream Foreman & Katello Projects

Theforeman organization maintains several COPR projects for Foreman and Katello development.

### Active Projects

| Project | Purpose | Branch Mapping | URL |
|---------|---------|---|---|
| `theforeman/foreman-nightly` | Development builds | `rpm/develop` | https://copr.fedorainfracloud.org/coprs/theforeman/foreman-nightly/ |
| `theforeman/foreman-3.18` | Foreman 3.18 release | `rpm/3.18` | https://copr.fedorainfracloud.org/coprs/theforeman/foreman-3.18/ |
| `theforeman/foreman-3.17` | Foreman 3.17 release | `rpm/3.17` | https://copr.fedorainfracloud.org/coprs/theforeman/foreman-3.17/ |
| `theforeman/pulpcore` | Pulp upstream | `rpm/develop` | https://copr.fedorainfracloud.org/coprs/theforeman/pulpcore/ |
| `theforeman/scratch-testing` | Test builds (non-release) | `rpm/develop` | https://copr.fedorainfracloud.org/coprs/theforeman/scratch-testing/ |

### Chroots Available

Each project typically builds for multiple Fedora and RHEL versions:

- `fedora-40-x86_64`, `fedora-40-aarch64`
- `fedora-41-x86_64`, `fedora-41-aarch64`
- `rhel-9-x86_64`, `rhel-9-aarch64`
- `rhel-8-x86_64`

Check specific project for supported chroots: https://copr.fedorainfracloud.org/coprs/theforeman/foreman-nightly/chroots/

### Common Project Operations

#### List Recent Builds

```bash
# Show recent 10 builds
copr-cli list-builds theforeman/foreman-nightly

# Monitor specific project
watch -n 10 "copr-cli list-builds theforeman/foreman-nightly | head -5"
```

#### Find Build for Specific Package

```bash
# List all builds mentioning a package name
copr-cli list-builds theforeman/foreman-nightly | grep rubygem

# More detailed: use COPR web API
curl -s "https://copr.fedorainfracloud.org/api_3/build/list?project_id=PROJECTID" | jq '.data[]'
```

#### Check Project Details

```bash
# View project metadata (if public)
copr-cli get theforeman/foreman-nightly
```

## Monitoring Builds

### Real-Time Build Status

For continuous monitoring during development:

```bash
# Watch latest build
BUILD_ID=$(copr-cli list-builds theforeman/foreman-nightly --limit 1 | tail -1 | awk '{print $1}')
watch -n 5 "copr-cli status $BUILD_ID"
```

### Compare Versions Across Releases

```bash
# Check package version in multiple projects
for proj in foreman-nightly foreman-3.18 foreman-3.17; do
  echo "=== theforeman/$proj ==="
  copr-cli list-builds theforeman/$proj | grep rubygem-example | head -1
done
```

## Scratch Builds vs Release Builds

### Scratch Builds (theforeman/scratch-testing)

- Temporary, non-public builds
- Good for testing changes before release
- Automatically cleaned up after 1-2 weeks
- Safe for experimenting

### Release Builds (theforeman/foreman-X.XX)

- Published to stable repositories
- Consumed by downstream (Satellite, other distributions)
- Permanent, versioned
- Require approval and testing before publishing

## Project Permissions

To build to a COPR project, your account needs:

1. Membership in the Foreman community
2. Permission in the specific project (usually auto-granted to members)
3. Valid COPR token in `~/.config/copr`

For access issues, contact the Foreman maintainers (Ondrej Gajdusek, Zach Huntington-Meath, etc.)

## See Also

- [COPR Documentation](https://docs.pagure.org/copr.copr/)
- [Theforeman COPR Organization](https://copr.fedorainfracloud.org/coprs/theforeman/)
