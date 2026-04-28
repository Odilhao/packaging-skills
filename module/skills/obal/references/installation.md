# Installation and Prerequisites

This guide covers installing obal and verifying your environment is ready for packaging workflows.

## System Requirements

- **Python:** 3.6 or newer
- **Git:** For repository management
- **Mock:** For local RPM builds (optional but recommended)
- **Ansible:** 2.9 or newer (obal is an Ansible wrapper)
- **Operating System:** Linux (Fedora, RHEL, CentOS recommended)

## Installation from PyPI

```bash
pip install obal
```

## Installation from Source

```bash
git clone https://github.com/theforeman/obal.git
cd obal
pip install -e .
```

## Installing Dependencies

**Fedora/RHEL/CentOS:**
```bash
sudo dnf install mock rpm-build ansible python3-pip
```

**For mock builds, add your user to the mock group:**
```bash
sudo usermod -a -G mock $USER
newgrp mock
```

## Verifying Installation

```bash
obal --help
ansible --version
```

Both commands should execute without errors and display version information.
