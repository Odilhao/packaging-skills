# Package Types Guide

Complete reference for different package types in foreman-packaging and their specific patterns.

## Ruby Gems (rubygem-*)

Ruby packages from rubygems.org. Most common in foreman-packaging.

### Structure

```
packages/katello/rubygem-example/
├── rubygem-example.spec
├── rubygem-example-1.2.3.gem (git-annex symlink)
└── sources (optional metadata)
```

### Basic Template

```spec
%global gem_name example

Name: rubygem-%{gem_name}
Version: 1.2.3
Release: 1%{?dist}
Summary: Example Ruby gem
License: MIT
URL: https://github.com/example/example
Source0: https://rubygems.org/gems/%{gem_name}-%{version}.gem

BuildRequires: rubygem-devel
Requires: ruby

%description
Description of the gem.

%prep
%setup -q -n %{gem_name}-%{version}

%build
gem build ../%{gem_name}-%{version}.gemspec

%install
mkdir -p %{buildroot}%{gem_dir}
cp -a .%{gem_dir}/* %{buildroot}%{gem_dir}/

%files
%{gem_dir}/gems/%{gem_name}-*
%{gem_dir}/specifications/%{gem_name}-*.gemspec
```

### Foreman Plugin Template

```spec
# template: foreman_plugin
%global gem_name foreman_example
%global foreman_version 3.17

Name: rubygem-%{gem_name}
Version: 1.2.3
Release: 1%{?foremandist}%{?dist}
Summary: Example Foreman plugin
License: GPLv3
URL: https://github.com/theforeman/%{gem_name}
Source0: https://rubygems.org/gems/%{gem_name}-%{version}.gem

BuildRequires: foreman-plugin >= %{foreman_version}
BuildRequires: foreman-assets >= %{foreman_version}
Requires: foreman >= %{foreman_version}

%description
Description of the plugin.

%prep
%setup -q -n %{gem_name}-%{version}

%build
gem build ../%{gem_name}-%{version}.gemspec
%gem_install

%install
mkdir -p %{buildroot}%{gem_dir}
cp -a .%{gem_dir}/* %{buildroot}%{gem_dir}/
%foreman_bundlerd_file
%foreman_precompile_plugin -s

%files
%dir %{gem_instdir}
%license %{gem_instdir}/LICENSE
%{gem_instdir}/app
%{gem_instdir}/config
%{gem_libdir}
%{gem_spec}
%{foreman_bundlerd_plugin}
%{foreman_assets_plugin}
%{foreman_webpack_plugin}

%posttrans
%{foreman_plugin_log}
```

### Key Macros

| Macro | Purpose |
|-------|---------|
| `%{gem_name}` | Ruby gem name (without rubygem- prefix) |
| `%{gem_dir}` | `/usr/share/gems` |
| `%{gem_instdir}` | Installed gem directory |
| `%{gem_libdir}` | Ruby code directory within gem |
| `%{gem_spec}` | .gemspec file |
| `%{gem_cache}` | Cached gem file |
| `%foreman_plugin_log` | Log plugin installation |
| `%foreman_bundlerd_file` | Generate bundler.d config |
| `%foreman_precompile_plugin` | Precompile assets |

### Smart Proxy Plugin Template

```spec
# template: smart_proxy_plugin
%global plugin_name dhcp
%global proxy_version 3.17

Name: foreman-proxy-plugin-%{plugin_name}
Version: 1.2.3
Release: 1%{?dist}
Summary: Smart Proxy %{plugin_name} plugin
```

---

## Python Packages (python-*, python3-*)

Python packages from PyPI. Growing in foreman-packaging.

### Modern Python (pyproject.toml)

```spec
Name: python-example
Version: 1.2.3
Release: 1%{?dist}
Summary: Example Python package
License: MIT
URL: https://github.com/example/example
Source0: https://files.pythonhosted.org/packages/.../example-%{version}.tar.gz

BuildArch: noarch
BuildRequires: python3-devel

%description
Description of the package.

%prep
%autosetup -n example-%{version}

%build
%pyproject_build

%install
%pyproject_install

%files
%{python3_sitelib}/example
```

### Python with Compiled Extensions (C/Rust)

For packages with compiled code, add BuildRequires:

```spec
BuildRequires: gcc
BuildRequires: python3-devel

# For Rust-based packages (e.g., cryptography)
BuildRequires: rust-toolset
BuildRequires: python3-maturin
```

### Python with Vendor Tarball (Rust)

Many Rust-powered Python packages use dual sources:

```spec
Source0: https://files.pythonhosted.org/packages/.../cryptography-43.0.1.tar.gz
Source1: https://downloads.theforeman.org/vendor/cryptography-43.0.1-vendor.tar.gz

%prep
%autosetup -p1 -n cryptography-%{version}
# Vendor tarball extracted during build
```

### Key Macros

| Macro | Expands To |
|-------|-----------|
| `%py3_build` | Build Python 3 package |
| `%py3_install` | Install Python 3 package |
| `%pyproject_build` | Build using pyproject.toml |
| `%pyproject_install` | Install using pyproject.toml |
| `%{python3_sitelib}` | `/usr/lib/python3.11/site-packages` |
| `%{python3_sitearch}` | `/usr/lib64/python3.11/site-packages` (with compiled code) |

---

## Node.js Packages (npm-*, *-webpack, *-react, etc.)

JavaScript/Node packages from npm registry.

### Example: React Package

```spec
Name: react
Version: 18.2.0
Release: 1%{?dist}
Summary: React JavaScript library
License: MIT
Source0: https://registry.npmjs.org/react/-/react-%{version}.tgz

BuildArch: noarch
BuildRequires: npm
BuildRequires: nodejs

%prep
%autosetup -n package

%build
npm install
npm run build

%install
mkdir -p %{buildroot}%{_jsdir}/react
cp -r dist %{buildroot}%{_jsdir}/react/

%files
%{_jsdir}/react/
```

### Special Consideration

- Node.js packages in foreman-packaging are often consumed by Foreman's webpack build
- Dependencies tracked in foreman-packaging's package_manifest.yaml
- Not separately installed as RPMs in most cases; bundled into Foreman

---

## Ansible Collections (ansible-collection-*)

Community and vendor Ansible content collections.

### Example: Satellite Collection

```spec
Name: ansible-collection-redhat-satellite
Version: 5.0.0
Release: 1%{?dist}
Summary: Red Hat Satellite Ansible Collection
License: GPL-3.0-or-later
Source0: https://galaxy.ansible.com/download/redhat-satellite-%{version}.tar.gz

BuildArch: noarch
BuildRequires: ansible-core >= 2.14
Requires: ansible-core >= 2.14

%prep
%autosetup -n redhat-satellite-%{version}

%install
mkdir -p %{buildroot}%{_ansible_collections_path}/redhat/satellite
cp -r . %{buildroot}%{_ansible_collections_path}/redhat/satellite/

%files
%{_ansible_collections_path}/redhat/satellite/
```

### Installation Paths

```
%{_ansible_collections_path}        # /usr/share/ansible/collections/
ansible_collections/
├── redhat/
│   ├── satellite/
│   └── rhel/
└── community/
    └── general/
```

---

## Go Packages (yggdrasil-*, *-agent, etc.)

Native Go binaries and daemons.

### Example: Go Binary

```spec
Name: yggdrasil-worker-forwarder
Version: 0.1.0
Release: 1%{?dist}
Summary: Yggdrasil worker forwarder service
License: MIT
Source0: https://github.com/theforeman/yggdrasil/archive/refs/tags/v%{version}.tar.gz

BuildRequires: golang >= 1.20
BuildRequires: systemd-devel

%prep
%autosetup -n yggdrasil-%{version}

%build
%gobuild -o yggdrasil-worker-forwarder ./cmd/worker-forwarder

%install
install -D -m 0755 yggdrasil-worker-forwarder %{buildroot}%{_bindir}/
install -D -m 0644 worker-forwarder.service %{buildroot}%{_unitdir}/

%files
%{_bindir}/yggdrasil-worker-forwarder
%{_unitdir}/worker-forwarder.service
```

### Key Macros

| Macro | Purpose |
|-------|---------|
| `%gobuild` | Compile Go binary with proper flags |
| `%gotest` | Run Go tests |
| `%{go_mod}` | Go module path (if using modules) |

---

## Software Collections (SCL) — Legacy Multi-Version Support

For supporting multiple Python or Ruby versions simultaneously (rare in modern foreman-packaging).

```spec
# template: scl
%global scl_name python37
%{?scl:%scl_package rubygem-%{gem_name}}

Name: %{?scl_prefix}rubygem-%{gem_name}
Version: 1.2.3
Release: 1%{?dist}
Summary: %{summary}

%scl_require rubygem-json

%description
%{summary}

# ... rest of spec uses scl macros
```

---

## Comparison Table

| Type | Package Naming | Build Tool | Output Location | Example |
|------|---|---|---|---|
| Ruby Gem | `rubygem-*` | gem build | `%{gem_dir}/` | rubygem-katello |
| Python | `python-*` | %py3_build | `%{python3_sitelib}/` | python-cryptography |
| Node.js | `npm-*` or bare | npm | `%{_jsdir}/` or bundled | react, webpack |
| Ansible Collection | `ansible-collection-*` | none | `%{_ansible_collections_path}/` | ansible-collection-redhat-satellite |
| Go Binary | `*-worker`, `*-agent` | %gobuild | `%{_bindir}/` | yggdrasil-worker-forwarder |

---

## Common Patterns Across Types

### Changelog Format (All Types)

```
* Mon Jun 02 2026 Your Name <email@example.com> - 1.2.3-1
- Release package-name 1.2.3
```

### BuildArch

- `noarch` — Ruby, Python (pure), Node.js, Ansible collections
- `x86_64` or arch-specific — Python with compiled extensions, Go binaries

### Testing

Many packages include `%check` section (optional but recommended):

```spec
%check
%pytest         # Python packages
rspec           # Ruby gems
npm test        # Node.js packages
%gotest         # Go packages
```

---

## See Also

- [Spec File Reference](spec-file-reference.md) — Detailed spec anatomy
- [Advanced Patterns](advanced-patterns.md) — Epoch, conditional deps, Satellite-specific builds
- [foreman-packaging](https://github.com/theforeman/foreman-packaging) — Real examples
