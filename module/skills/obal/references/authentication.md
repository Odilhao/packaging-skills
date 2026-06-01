# Authentication Setup

Different build systems require different authentication configurations.

## Table of Contents
- [COPR Authentication](#copr-authentication)
- [Koji Authentication](#koji-authentication)
- [Troubleshooting Authentication](#troubleshooting-authentication)

## COPR Authentication

COPR (Community Projects) is the default build system for upstream packages.

**Configuration file:** `~/.config/copr`

**Getting your API token:**
1. Visit https://copr.fedorainfracloud.org/api/
2. Log in with your FAS (Fedora Account System) credentials
3. Copy the configuration snippet

**Example config:**
```ini
[copr-cli]
login = your-username
username = your-username
token = your-api-token-here
copr_url = https://copr.fedorainfracloud.org
```

**Testing authentication:**
```bash
copr-cli whoami
copr-cli list
```

## Koji Authentication

**Configuration file:** `~/.koji/config`

**Example config:**
```ini
[koji]
server = https://koji.fedoraproject.org/kojihub
weburl = https://koji.fedoraproject.org/koji
topurl = https://kojipkgs.fedoraproject.org
cert = ~/.fedora.cert
ca = ~/.fedora-upload-ca.cert
serverca = ~/.fedora-server-ca.cert
```

**Testing authentication:**
```bash
koji hello
```

## Troubleshooting Authentication

**COPR Error:** `Error: Invalid API token`

**Solutions:**
- Regenerate API token at https://copr.fedorainfracloud.org/api/
- Check config file permissions: `chmod 600 ~/.config/copr`
- Verify token hasn't expired
