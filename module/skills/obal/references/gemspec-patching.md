# Gemspec Patching

How to strip unwanted gem dependencies from RPM packages without upgrading to a new upstream version.

## Table of Contents
- [When to Use](#when-to-use)
- [The %gemspec_remove_dep Macro](#the-gemspec_remove_dep-macro)
- [Pattern: Strip Dep and Exclude CLI](#pattern-strip-dep-and-exclude-cli)
- [Verification](#verification)
- [Existing Precedents](#existing-precedents)

## When to Use

Use `%gemspec_remove_dep` when:

1. **Upstream declares a dep only used by a CLI tool, not the library** — the gem's library code doesn't `require` the dependency, but the gemspec lists it because a bundled CLI script needs it. You can strip the dep and exclude the CLI binary.

2. **A newer upstream version drops the dep but requires a newer language runtime** — e.g., ruby2ruby 2.6.x drops `ruby_parser` but requires Ruby >= 3.2, while EL9 ships Ruby 3.0. You can't upgrade, so patch the current version.

3. **The dep conflicts with the target OS** — e.g., a gem depends on a version of `racc` that conflicts with the version bundled in `ruby-libs` on EL9.

**Before using:** Verify the library code genuinely doesn't use the dependency:
```bash
# Check if any lib/ files require the dependency
curl -sL "https://raw.githubusercontent.com/<org>/<repo>/refs/tags/v<version>/lib/<gem>.rb" | grep "require.*<dep>"

# Check the CLI tool
curl -sL "https://raw.githubusercontent.com/<org>/<repo>/refs/tags/v<version>/bin/<tool>" | grep "require.*<dep>"
```

## The %gemspec_remove_dep Macro

`%gemspec_remove_dep` is a Fedora/RHEL RPM macro that modifies the gemspec during `%prep` to remove a declared dependency. This prevents RPM auto-requires from generating a `Requires: rubygem(<dep>)` in the built RPM.

```spec
%prep
%setup -q -n %{gem_name}-%{version}

%gemspec_remove_dep -g <dep-name> "<version-constraint>"
```

**Arguments:**
- `-g <dep-name>` — the gem dependency to remove (matches `add_runtime_dependency` or `add_dependency` in the gemspec)
- `"<version-constraint>"` — the version string to match (e.g., `"~> 3.1"`)

## Pattern: Strip Dep and Exclude CLI

When stripping a dependency that a CLI tool needs, also exclude the CLI binary to avoid shipping a broken tool.

**Before (original spec):**
```spec
%prep
%setup -q -n %{gem_name}-%{version}

%install
mkdir -p %{buildroot}%{gem_dir}
cp -a .%{gem_dir}/* %{buildroot}%{gem_dir}/

mkdir -p %{buildroot}%{_bindir}
cp -a .%{_bindir}/* %{buildroot}%{_bindir}/

find %{buildroot}%{gem_instdir}/bin -type f | xargs chmod a+x

%files
%dir %{gem_instdir}
%{_bindir}/r2r_show
%{gem_instdir}/bin
%{gem_libdir}
```

**After (patched spec):**
```spec
%prep
%setup -q -n %{gem_name}-%{version}

# ruby2ruby 2.5.x declares ruby_parser as a runtime dep, but only the
# bin/r2r_show CLI uses it — the library itself does not.
# ruby_parser was removed from the repo with safemode 2.0 migration.
%gemspec_remove_dep -g ruby_parser "~> 3.1"

%install
mkdir -p %{buildroot}%{gem_dir}
cp -a .%{gem_dir}/* %{buildroot}%{gem_dir}/

%files
%dir %{gem_instdir}
%exclude %{gem_instdir}/bin
%{gem_libdir}
```

**Key changes:**
1. Added `%gemspec_remove_dep` in `%prep`
2. Removed `%{_bindir}` installation (no bindir mkdir, no cp, no chmod)
3. Changed `%{gem_instdir}/bin` from included to `%exclude`
4. Removed `%{_bindir}/r2r_show` from `%files`

**Use `obal bump-release`** (not `obal update`) since the upstream version isn't changing:
```bash
obal bump-release rubygem-ruby2ruby --changelog "Drop ruby_parser dependency and r2r_show binary"
```

## Verification

After building, verify the dependency is gone from the built RPM:

```bash
# Scratch build first
obal scratch rubygem-ruby2ruby

# Check auto-requires on the built RPM
rpm -qpR <scratch-build-url>/rubygem-ruby2ruby-*.noarch.rpm

# Should NOT contain rubygem(ruby_parser)
# Should still contain rubygem(sexp_processor) and other legitimate deps
```

Then run repoclosure:
```bash
obal repoclosure rubygem-ruby2ruby --check "<scratch-repo-url>" --dist el9
```

## Existing Precedents

`%gemspec_remove_dep` is widely used in foreman-packaging. Common reasons:

### Bundled in Ruby standard library (EL9)

Ruby 3.0 on EL9 bundles several gems into `ruby-libs`. Upstream gems declaring these as dependencies cause conflicts because there's no separate RPM to satisfy the auto-requires. Strip them conditionally:

```spec
# racc bundled in ruby-libs on EL9
%if 0%{?rhel} == 9
%gemspec_remove_dep -g racc "~> 1.5"
%endif
```

Packages using this pattern: `rubygem-actionpack` (racc), `rubygem-nokogiri` (racc), `rubygem-gettext` (racc)

### Default gems extracted in Ruby 3.4+

Ruby 3.4 extracts several gems from stdlib as defaults. Upstream gems add explicit dependencies on these extracted gems, but EL9's Ruby 3.0 bundles them implicitly — no separate RPM exists. Strip unconditionally:

```spec
%gemspec_remove_dep -g base64
%gemspec_remove_dep -g csv
%gemspec_remove_dep -g logger
%gemspec_remove_dep -g ostruct
%gemspec_remove_dep -g bigdecimal
%gemspec_remove_dep -g drb
%gemspec_remove_dep -g mutex_m
```

Packages using this pattern: `rubygem-activesupport` (base64, drb, mutex_m, bigdecimal, logger, securerandom, benchmark), `rubygem-hammer_cli` (base64, csv), `rubygem-jwt` (base64), `rubygem-net-ldap` (base64, ostruct), `rubygem-fog-vsphere` (base64, ostruct), `rubygem-dynflow` (csv), many others.

### Net protocol gems (EL9)

Ruby 3.1+ extracted `net-smtp`, `net-imap`, `net-pop` from stdlib. On EL9 (Ruby 3.0) these are still bundled:

```spec
%gemspec_remove_dep -g net-smtp
%gemspec_remove_dep -g net-imap
%gemspec_remove_dep -g net-pop
```

Packages: `rubygem-actionmailbox`, `rubygem-actionmailer`, `rubygem-mail`

### Dep removed from repo

When a dependency is removed from the packaging repo (e.g., replaced by another package), strip it from consumers that still declare it:

```spec
# ruby_parser replaced by prism in safemode 2.0 migration
%gemspec_remove_dep -g ruby_parser "~> 3.1"
```

Packages: `rubygem-ruby2ruby`

### Version constraint widening

Use `%gemspec_remove_dep` + `%gemspec_add_dep` to widen upstream version constraints that are too restrictive for the packaged versions:

```spec
# Upstream pins thor < 1.3, but we have thor 1.3+
%gemspec_remove_dep -g thor ['>= 1.0.1', '< 1.3']
%gemspec_add_dep -g thor ['>= 1.0.1', '< 2.0']
```

Packages: `rubygem-facter` (thor), `rubygem-foreman_concrete` (sentry-raven), `rubygem-http` (llhttp), `rubygem-netbox-client-ruby` (faraday, faraday_middleware), `rubygem-opennebula` (nokogiri)

### Other macros

- **`%gemspec_add_dep`** — adds or replaces a dependency (often paired with remove)
- **`%gemspec_remove_file`** — removes bundled files from the gemspec (e.g., vendored SQLite sources in `rubygem-sqlite3`)
