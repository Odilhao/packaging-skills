# Rebuild Cascade Planning

Detailed reference for assessing, planning, and executing rebuild cascades.

## Rebuild Trigger Analysis

| Change Type | Rebuild Needed? | Reason |
|------------|----------------|--------|
| Patch in source (no ABI change) | No | Binary interface unchanged |
| Build flag change (no ABI change) | No | Same symbols exported |
| Minor version bump | Possibly | Check if ABI changed |
| Major version bump | Usually | API/ABI changes likely |
| Soname bump (libfoo.so.1→2) | **Yes — all linked packages** | Old symbol version gone |
| Python major version (3.x→3.y) | **Yes — all Python packages** | Bytecode/ABI tag changes |
| Compiler major version | **Yes — often all packages** | ABI may shift |

## Example: Library Soname Bump

```
libcurl.so.4 → libcurl.so.5 (ABI break)

Direct rebuilds needed:
  ├── curl (CLI tool)
  ├── git (uses libcurl for HTTP)
  ├── python3-pycurl (Python bindings)
  ├── php-curl (PHP bindings)
  ├── wget2 (uses libcurl)
  └── 127 more packages...

Each may trigger further rebuilds (transitive cascade):
  git rebuild →
    ├── git-lfs (built against git)
    ├── tig (git dependency)
    └── Many git extensions...

Scope assessment:
  - Direct deps: 132 packages
  - Transitive: ~400 packages
  - Build time: ~48 hours (parallel build system)
  - Human time: 2-3 days coordinating
```

## Cascade Planning Steps

### 1. Query reverse dependencies

```bash
# Runtime reverse deps
dnf repoquery --whatrequires 'libcurl.so.4()(64bit)'

# Build-time reverse deps (what needs this to build)
koji list-buildroot --newest f40-build libcurl

# Full reverse dep tree
dnf repoquery --whatrequires --recursive python3-requests
```

### 2. Assess scope and priority

- **Critical consumers** (must work): ansible-core, git, systemd bindings
- **High-impact**: widely-installed packages
- **Low-impact**: leaf packages with few/no reverse deps

### 3. Use side tags for coordinated rebuilds

```bash
# Create a side tag (isolates builds from main tag)
koji add-side-tag f40-updates-candidate --suffix=libcurl-soname-bump

# Build library first
koji build f40-side-tag-12345 libcurl-8.0.0-1.fc40.src.rpm

# Build dependents against the new library
koji build f40-side-tag-12345 git-2.42.0-1.fc40.src.rpm
koji build f40-side-tag-12345 python3-pycurl-7.45.0-1.fc40.src.rpm

# When all pass: merge side tag into main
koji merge-side-tag f40-side-tag-12345
```

### 4. Coordinate the push

- Notify affected package maintainers before starting
- File FTBFS (Fails To Build From Source) bugs for packages needing patches
- Test critical consumers before merging the side tag
- Push all rebuilt packages together to avoid broken intermediate states
