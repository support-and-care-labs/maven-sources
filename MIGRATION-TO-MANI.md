# Migration from Google Repo to Mani

## Overview

This document tracks the migration from Google's `repo` tool to `mani` for managing the Maven multi-repository build.

## Why Mani?

**Problems with repo:**
- ❌ `.repo/manifests/` duplication is unavoidable
- ❌ When using `path='.'`, we still get two checkouts and two git clones
- ❌ Complex Python codebase, harder to customize
- ❌ Designed for Android's specific workflow

**Benefits of mani:**
- ✅ Simpler YAML configuration
- ✅ No special treatment of manifest repository
- ✅ Written in Go, single binary
- ✅ More flexible for our use case
- ✅ No hidden `.repo/` directory

## Starting Point

We're starting from the `mani-migration` branch which contains:
- ✅ Flattened aggregator structure (no `aggregator/` directory)
- ✅ Simplified module paths in POMs
- ✅ Updated `.gitignore` for repo-managed directories
- ✅ All aggregator POMs at root level

## Mani Configuration

### File: `mani.yaml`

The `mani.yaml` file defines all Maven repositories and where they should be cloned.

**Key differences from repo's `default.xml`:**
- Simple YAML format instead of XML
- No distinction between "manifest repo" and other repos
- All repos treated equally
- Shallow clones by default

**Structure:**
```yaml
projects:
  project-name:
    path: checkout/path
    url: https://github.com/org/repo.git
    clone: shallow  # or "recursive" for submodules
    branch: branch-name  # optional
```

## Usage

### Initial Setup

**Clone the root repository:**
```bash
git clone https://github.com/support-and-care-labs/maven-sources.git
cd maven-sources
```

**Sync all repositories:**
```bash
mani sync
```

This will clone all repositories defined in `mani.yaml` into their specified paths.

### Common Operations

**Update all repositories:**
```bash
mani sync
```

**Run commands across all repos:**
```bash
mani exec -- git status
mani exec -- git pull
```

**List all projects:**
```bash
mani list projects
```

**Get project info:**
```bash
mani list projects --tree
```

## Expected Structure After `mani sync`

```
maven-sources/
├── .git/                          ← Root repo (maven-sources.git)
├── mani.yaml                      ← Mani configuration
├── pom.xml                        ← Root aggregator POM
├── .gitignore                     ← Ignores all checked-out repos
│
├── core/
│   ├── pom.xml                    ← Core aggregator POM
│   ├── build-cache/               ← Cloned by mani
│   │   └── .git/
│   ├── maven/                     ← Cloned by mani
│   │   └── .git/
│   └── ...
│
├── plugins/
│   ├── pom.xml                    ← Plugins aggregator POM
│   ├── core/
│   │   ├── pom.xml                ← Plugins-core aggregator POM
│   │   ├── maven-clean-plugin/    ← Cloned by mani
│   │   │   └── .git/
│   │   └── ...
│   └── ...
│
└── site/, doxia/, misc/, plexus/, shared/, etc.
```

## Key Differences from Repo

### ✅ Advantages

1. **No hidden directories** - No `.repo/` overhead
2. **Single checkout** - Root repo is just a normal git clone
3. **Simpler** - One YAML file instead of XML + Python internals
4. **Flexible** - Easy to add custom scripts/tasks
5. **Better Git integration** - Root repo works like any git repo

### ⚠️ Considerations

1. **Manual root clone** - Must clone maven-sources first, then run `mani sync`
2. **No sync state** - Mani doesn't track what was last synced (simpler but less state)
3. **Different commands** - `repo sync` → `mani sync`, etc.

## Migration Steps

### For Existing Repo Users

If you have an existing repo checkout:

1. **Extract your root repository:**
   ```bash
   # From existing repo checkout
   cd .repo/manifests
   git remote -v  # Note the URL
   cd ../../
   ```

2. **Fresh start with mani:**
   ```bash
   cd /path/to/new/location
   git clone https://github.com/support-and-care-labs/maven-sources.git
   cd maven-sources
   mani sync
   ```

3. **Verify structure:**
   ```bash
   ls -la
   # Should see: pom.xml, core/, plugins/, etc.

   ls -la core/
   # Should see: pom.xml, maven/, build-cache/, etc.
   ```

### For New Users

Simply:
```bash
git clone https://github.com/support-and-care-labs/maven-sources.git
cd maven-sources
mani sync
```

## Maintenance

### Adding a New Repository

Edit `mani.yaml`:
```yaml
projects:
  plugins/tools/my-new-plugin:
    path: plugins/tools/my-new-plugin
    url: https://github.com/apache/maven-my-new-plugin.git
    clone: shallow
```

Update `plugins/tools/pom.xml`:
```xml
<modules>
  <module>my-new-plugin</module>
</modules>
```

Then sync:
```bash
mani sync
```

### Removing a Repository

1. Remove from `mani.yaml`
2. Remove `<module>` from aggregator POM
3. Remove the directory:
   ```bash
   rm -rf path/to/removed/repo
   ```

### Updating Repository URLs

Edit `mani.yaml` and run:
```bash
mani sync
```

## Testing

### Test Location
- `/tmp/mani-test` - Fresh test checkout

### Test Steps

1. Clone root repo:
   ```bash
   cd /tmp
   git clone https://github.com/support-and-care-labs/maven-sources.git mani-test
   cd mani-test
   ```

2. Verify mani.yaml exists:
   ```bash
   ls -la mani.yaml
   ```

3. Sync all repositories:
   ```bash
   mani sync
   ```

4. Verify structure:
   ```bash
   test -f pom.xml && echo "✅ Root pom.xml"
   test -f core/pom.xml && echo "✅ core/pom.xml"
   test -d core/maven && echo "✅ core/maven cloned"
   test -d plugins/core/maven-clean-plugin && echo "✅ plugins cloned"
   ```

5. Test Maven build:
   ```bash
   mvn clean install -DskipTests
   ```

## Comparison: Repo vs Mani

| Feature | Repo | Mani |
|---------|------|------|
| **Config File** | XML (`default.xml`) | YAML (`mani.yaml`) |
| **Root Repo** | Special (`.repo/manifests/`) | Normal git clone |
| **Sync Command** | `repo sync` | `mani sync` |
| **Hidden Dir** | `.repo/` (~256K overhead) | None |
| **Duplication** | Yes (manifests + path='.') | No |
| **Setup** | `repo init -u URL` | `git clone URL && mani sync` |
| **Branch State** | Detached HEAD | Normal branches |
| **Complexity** | High (Python) | Low (Go binary) |
| **Customization** | Difficult | Easy (YAML) |

## Status

- ✅ Created `mani.yaml` with all 100+ repositories
- ✅ Tested mani installation
- ⏳ Testing full sync in `/tmp/mani-test`
- ⏳ Verify Maven build works
- ⏳ Document edge cases

## Next Steps

1. Test in `/tmp/mani-test`
2. Verify Maven build
3. Compare disk usage vs repo
4. Document any issues
5. Update `.gitignore` if needed
6. Create final migration guide

## Notes

- Mani doesn't enforce shallow clones - developers can do full clones if needed
- Each repository is independent - can use different branches/tags
- Root repo (`maven-sources.git`) contains aggregator POMs and `mani.yaml`
- No special `.repo/` directory means cleaner structure

---

**Created:** 2025-12-03
**Branch:** mani-migration
**Files:** mani.yaml, MIGRATION-TO-MANI.md
