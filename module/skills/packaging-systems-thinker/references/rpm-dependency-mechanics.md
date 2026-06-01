# RPM Dependency Mechanics

Detailed reference for RPM dependency types, automatic dependency generation, and virtual provides.

## Automatic Dependency Detection

RPM automatically detects dependencies at build time using generators:

```
Shared libraries:  ldd analysis finds libfoo.so.2 → Requires: libfoo.so.2()(64bit)
Python imports:    python-rpm-generators scans imports → Requires: python3dist(requests)
Perl modules:      perl.req scans use/require statements
pkgconfig:         Scans .pc files → Requires: pkgconfig(libcurl)
```

Manual dependencies are needed for:
- Feature-based requirements (`Requires: webserver` — any provider)
- Version constraints beyond what auto-detection provides
- Virtual provides for capability-based dependency resolution

## Provides/Requires in Practice

```
Package A (httpd):
  Provides: webserver
  Provides: httpd = 2.4.57
  Provides: httpd(x86-64) = 2.4.57
  Provides: mod_ssl
  Provides: libhttpd.so.2()(64bit)  # Auto-generated

Package B (app needing a web server):
  Requires: webserver              # Any package providing 'webserver' — httpd or nginx
  Requires: httpd >= 2.4           # Specific implementation with version constraint
  Requires: libhttpd.so.2()(64bit) # Specific shared library

Package C (nginx — alternative provider):
  Provides: webserver              # Also satisfies 'webserver' requirement
  Conflicts: httpd                 # Cannot be installed alongside httpd
```

## Full Dependency Type Reference

| Type | Direction | Behavior |
|------|-----------|----------|
| `Requires` | Hard runtime | Must be installed for package to function |
| `BuildRequires` | Build-time | Must be installed to build the package |
| `Recommends` | Soft runtime | Installed by default, can be excluded with `--setopt=install_weak_deps=False` |
| `Suggests` | Hint | Not installed by default, shown as available |
| `Supplements` | Reverse recommends | "If X is installed, also install me" |
| `Enhances` | Reverse suggests | "If X is installed, I could be useful" |
| `Conflicts` | Negative | Cannot be installed at the same time |
| `Obsoletes` | Replacement | Auto-removes old package on upgrade |
| `OrderWithRequires` | Ordering | Controls scriptlet execution order |

## Dependency Ordering for Scriptlets

```spec
# Ensures the shadow-utils package (providing useradd) is installed
# BEFORE our %pre scriptlet runs to create a system user
Requires(pre):    shadow-utils

# Ensures systemd is available when our %post scriptlet runs
# to enable/start the service
Requires(post):   systemd
```

## Version Constraint Patterns

```spec
# Minimum version — upstream requires at least this
Requires: python3-urllib3 >= 1.21.1

# Version window — compatible range
Requires: python3-urllib3 >= 1.21.1
Requires: python3-urllib3 < 2.0

# Exact match (fragile — avoid unless required)
Requires: python3-foo = 1.2.3

# No constraint — any version acceptable (stable API)
Requires: python3-certifi
```

## Rich/Boolean Dependencies (RPM 4.13+)

```spec
# OR dependency — either provider satisfies
Requires: (python3-tomli or python3-tomllib)

# Conditional — only require if another package is installed
Requires: (python3-typing-extensions if python3 < 3.11)

# Complex boolean
Requires: (pkgA >= 2.0 with pkgB) or (pkgC >= 1.0)
```
