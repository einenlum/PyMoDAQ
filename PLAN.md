# PyMoDAQ Monorepo Migration Plan

## Problem Statement

PyMoDAQ is currently split into 4 separate repositories:
- `pymodaq_utils` (v0.0.x)
- `pymodaq_data` (v5.0.x)
- `pymodaq_gui` (v5.0.x)
- `pymodaq` (v5.0.x)

**Issues with current setup:**
- Managing 4 different versions is complex and error-prone
- Cross-package changes require coordinating multiple PRs across repos
- Duplicate CI/CD workflows in each repo
- Version compatibility matrix causes dependency hell
- Testing the full stack together is difficult

**Proposed Solution:**
- **One repository** containing all 4 packages
- **Four separate PyPI packages** (architectural separation maintained)
- **Single version number** across all packages
- **Unified CI/CD** for testing and releases

---

## Proposed Monorepo Structure

```
PyMoDAQ/
├── packages/
│   ├── pymodaq_utils/
│   │   ├── pyproject.toml
│   │   ├── src/pymodaq_utils/
│   │   ├── tests/
│   │   └── README.rst
│   ├── pymodaq_data/
│   │   ├── pyproject.toml
│   │   ├── src/pymodaq_data/
│   │   ├── tests/
│   │   └── README.rst
│   ├── pymodaq_gui/
│   │   ├── pyproject.toml
│   │   ├── src/pymodaq_gui/
│   │   ├── tests/
│   │   └── README.rst
│   └── pymodaq/
│       ├── pyproject.toml
│       ├── src/pymodaq/
│       ├── tests/
│       └── README.rst
├── .github/
│   └── workflows/
│       ├── tests.yml          # Unified testing workflow
│       ├── publish.yml        # Multi-package publishing
│       └── updater.yml
├── scripts/
│   ├── bump_version.py        # Helper to update versions
│   └── publish_all.sh         # Local publishing helper
├── pyproject.toml             # Root config (optional)
├── README.rst
├── CLAUDE.md
└── PLAN.md                    # This file
```

---

## Version Management Strategy

### Unified Versioning

All packages share the same version number.

**Example:**
- Single git tag: `v5.1.0`
- All 4 packages published as `5.1.0`
- Dependencies declared simply: `pymodaq_utils>=5.1.0`

## CI/CD Workflows

### Testing Workflow (`.github/workflows/tests.yml`)

```yaml
name: tests

on:
  pull_request:
  push:
    branches:
    - '*'
    - '!badges'

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  test-matrix:
    continue-on-error: true
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        python-version: ["3.9", "3.10", "3.11", "3.12"]
        qt-backend: [pyqt5, pyqt6, pyside6]
        package: [pymodaq_utils, pymodaq_data, pymodaq_gui, pymodaq]

    runs-on: ${{ matrix.os }}
    env:
      DISPLAY: ':99'
      QT_DEBUG_PLUGINS: 1

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies (Linux)
        if: runner.os == 'Linux'
        run: |
          sudo apt update
          sudo apt install -y libxkbcommon-x11-0 libxcb-icccm4 libxcb-image0 \
            libxcb-keysyms1 libxcb-cursor0 libxcb-randr0 libxcb-render-util0 \
            libxcb-xinerama0 libxcb-xfixes0 x11-utils libgl1 libegl1

      # Install in dependency order
      - name: Install packages
        run: |
          python -m pip install --upgrade pip
          pip install flake8 pytest pytest-cov pytest-qt pytest-xvfb pytest-xdist \
            setuptools wheel numpy h5py ${{ matrix.qt-backend }}
          pip install -e packages/pymodaq_utils
          pip install -e packages/pymodaq_data
          pip install -e packages/pymodaq_gui
          pip install -e packages/pymodaq

      - name: Lint
        run: |
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics --exclude=docs

      - name: Test ${{ matrix.package }}
        run: |
          cd packages/${{ matrix.package }}
          pytest -vv --cov=. -n 1
```

### Publishing Workflow (`.github/workflows/publish.yml`)

```yaml
name: Publish All Packages

on:
  release:
    types: [created]

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install hatch
        run: |
          python -m pip install --upgrade pip
          pip install hatch hatchling

      # Verify version from tags
      - name: Get version from tags
        run: |
          git fetch --prune --unshallow || true
          git fetch --depth=1 origin +refs/tags/*:refs/tags/* || true

      # Build and publish in dependency order
      - name: Publish pymodaq_utils
        env:
          HATCH_INDEX_USER: ${{ secrets.PYPI_USERNAME }}
          HATCH_INDEX_AUTH: ${{ secrets.PYPI_PASSWORD }}
        run: |
          cd packages/pymodaq_utils
          hatch version
          hatch build
          hatch publish

      - name: Publish pymodaq_data
        env:
          HATCH_INDEX_USER: ${{ secrets.PYPI_USERNAME }}
          HATCH_INDEX_AUTH: ${{ secrets.PYPI_PASSWORD }}
        run: |
          cd packages/pymodaq_data
          hatch version
          hatch build
          hatch publish

      - name: Publish pymodaq_gui
        env:
          HATCH_INDEX_USER: ${{ secrets.PYPI_USERNAME }}
          HATCH_INDEX_AUTH: ${{ secrets.PYPI_PASSWORD }}
        run: |
          cd packages/pymodaq_gui
          hatch version
          hatch build
          hatch publish

      - name: Publish pymodaq
        env:
          HATCH_INDEX_USER: ${{ secrets.PYPI_USERNAME }}
          HATCH_INDEX_AUTH: ${{ secrets.PYPI_PASSWORD }}
        run: |
          cd packages/pymodaq
          hatch version
          hatch build
          hatch publish
```

---

## Version Management with hatch-vcs

Each package's `pyproject.toml` uses hatch-vcs to get version from git tags:

```toml
[tool.hatch.version]
source = "vcs"
fallback-version = "5.1.0"
```

**Updated Dependency Specifications:**

Since all packages are in the same repo and share versions, dependencies can be simplified:

**Option A - No version constraint (monorepo trust):**
```toml
dependencies = [
    "pymodaq_utils",
    "pymodaq_data",
    ...
]
```

**Option B - Version floor (safer for PyPI):**
```toml
dependencies = [
    "pymodaq_utils>=5.1.0",
    "pymodaq_data>=5.1.0",
    ...
]
```

---

## Migration Steps

### Phase 1: Preparation: DONE!

1. **Backup everything**
   - Create backups of all 4 repositories
   - Tag current state in all repos

2. **Choose migration approach:**
   - **Simple:** Copy files (loses individual repo history)
   - **Advanced:** Use `git subtree` or `git filter-repo` to preserve full history

3. **Decide on version:**
   - Recommended: `5.1.0` (next minor version)
   - Signals new unified development model

### Phase 2: Repository Restructuring: DONE!

1. **Create packages directory:**
   ```bash
   cd PyMoDAQ
   mkdir -p packages
   ```

2. **Move current PyMoDAQ code:**
   ```bash
   mkdir -p packages/pymodaq
   git mv src packages/pymodaq/
   git mv tests packages/pymodaq/
   git mv pyproject.toml packages/pymodaq/
   git mv README.rst packages/pymodaq/
   git commit -m "Restructure: Move pymodaq to packages/"
   ```

3. **Copy in dependency packages:**
   ```bash
   cp -r local_deps/pymodaq_utils packages/
   cp -r local_deps/pymodaq_data packages/
   cp -r local_deps/pymodaq_gui packages/

   # Remove git metadata from copied packages
   rm -rf packages/pymodaq_utils/.git
   rm -rf packages/pymodaq_data/.git
   rm -rf packages/pymodaq_gui/.git

   git add packages/
   git commit -m "Add pymodaq_utils, pymodaq_data, pymodaq_gui packages"
   ```

4. **Clean up local_deps:**
   ```bash
   git rm -rf local_deps/
   git commit -m "Remove local_deps (now in packages/)"
   ```

### Phase 3: Update Configuration Files: DONE!

1. **Update each `pyproject.toml`:**
   - Set unified version: `fallback-version = "5.1.0"`
   - Update dependency specs (remove version ranges or set `>=5.1.0`)
   - Update `[project.urls]` to point to monorepo
   - Ensure all use `hatch-vcs`

2. **Update root files:**
   - Update root `README.rst` to document monorepo structure

3. **Create unified CI workflows:**
   - Replace `.github/workflows/tests.yml` with monorepo version
   - Replace `.github/workflows/python-publish.yml` with multi-package version

### Phase 4: Version Alignment & Testing

1. **Align versions:**
   ```bash
   # Update all fallback-version to 5.1.0 in each pyproject.toml
   git add -A
   git commit -m "Align all packages to version 5.1.0"
   ```

2. **Test local installation:**
   ```bash
   # Create fresh virtual environment
   python -m venv test_env
   source test_env/bin/activate  # or test_env\Scripts\activate on Windows

   # Install in dependency order
   pip install -e packages/pymodaq_utils
   pip install -e packages/pymodaq_data
   pip install -e packages/pymodaq_gui
   pip install -e "packages/pymodaq[dev]"

   # Run tests
   pytest packages/pymodaq_utils/tests -vv
   pytest packages/pymodaq_data/tests -vv
   pytest packages/pymodaq_gui/tests -vv
   pytest packages/pymodaq/tests -vv
   ```

3. **Test builds:**
   ```bash
   cd packages/pymodaq_utils && hatch build && cd ../..
   cd packages/pymodaq_data && hatch build && cd ../..
   cd packages/pymodaq_gui && hatch build && cd ../..
   cd packages/pymodaq && hatch build && cd ../..
   ```

### Phase 5: First Release

1. **Tag the repository:**
   ```bash
   git tag -a v5.1.0 -m "First monorepo release"
   git push origin v5.1.0
   ```

2. **Create GitHub Release:**
   - Go to GitHub → Releases → Create new release
   - Choose tag `v5.1.0`
   - Title: "v5.1.0 - Monorepo Migration"
   - Describe the change and benefits
   - Publish release → triggers automatic publishing to PyPI

3. **Verify on PyPI:**
   - Check all 4 packages published successfully
   - Verify version numbers are correct
   - Test installation: `pip install pymodaq==5.1.0`

### Phase 6: Cleanup & Communication

1. **Archive old repositories:**
   - Add README to each old repo pointing to monorepo
   - Archive repositories on GitHub (Settings → Archive)
   - Don't delete (preserve history and links)

2. **Update documentation:**
   - Update website/docs with new repo structure
   - Update contributor guide
   - Update installation instructions

3. **Announce changes:**
   - Post announcement in issues/discussions
   - Email contributors/users
   - Update any external docs/tutorials

---

## Benefits

✅ **Single version number** across all packages
✅ **One PR** for cross-package changes
✅ **Unified CI** - test the whole stack together
✅ **Simpler dependency management** - no more version range conflicts
✅ **Easier onboarding** - clone once, see everything
✅ **Atomic releases** - all packages always compatible
✅ **Still 4 separate PyPI packages** - users install what they need
✅ **Better testing** - full integration testing in CI

---

## Developer Workflow After Migration

### Setup

```bash
# Clone monorepo
git clone https://github.com/PyMoDAQ/PyMoDAQ
cd PyMoDAQ

# Install all packages in editable mode
pip install -e packages/pymodaq_utils
pip install -e packages/pymodaq_data
pip install -e packages/pymodaq_gui
pip install -e "packages/pymodaq[dev]"
```

### Development

```bash
# Make changes across multiple packages
vim packages/pymodaq_utils/src/pymodaq_utils/config.py
vim packages/pymodaq_data/src/pymodaq_data/data.py

# Commit everything together
git add .
git commit -m "Add new config option and data structure"

# Test everything
pytest

# Push one PR
git push origin feature/my-feature
```

### Releasing

```bash
# Tag with new version
git tag v5.2.0
git push --tags

# Create GitHub release
# → GitHub Actions automatically publishes all 4 packages to PyPI
```

---

## Risks & Mitigation

### Risk: Breaking existing installations

**Mitigation:**
- Keep old repos archived (not deleted)
- Old versions remain on PyPI
- Clear communication about migration
- Users can pin to pre-monorepo versions if needed

### Risk: Git history confusion

**Mitigation:**
- Tag all repos before migration
- Consider using `git subtree` to preserve full history
- Document the migration clearly

### Risk: CI/CD issues

**Mitigation:**
- Test workflows thoroughly before first release
- Use TestPyPI for dry run
- Manual fallback: `hatch publish` locally if needed

### Risk: Contributor confusion

**Mitigation:**
- Update CONTRIBUTING.md
- Clear documentation in README
- Update CLAUDE.md for Claude Code users

---

## Alternative: Preserve Full Git History

If preserving individual package histories is important:

```bash
# Use git filter-repo (advanced)
# This creates a monorepo with full history from all 4 repos
# See: https://github.com/newren/git-filter-repo

# Or use git subtree merge strategy
git remote add pymodaq_utils https://github.com/PyMoDAQ/pymodaq_utils
git fetch pymodaq_utils
git merge -s ours --no-commit --allow-unrelated-histories pymodaq_utils/main
git read-tree --prefix=packages/pymodaq_utils/ -u pymodaq_utils/main
git commit -m "Merge pymodaq_utils history"

# Repeat for other packages
```

This is more complex but preserves `git log` and `git blame` history.

---

## Timeline Estimate

- **Preparation:** 1-2 hours
- **Restructuring:** 2-4 hours
- **Testing:** 2-4 hours
- **First release:** 1 hour
- **Documentation:** 2-3 hours
- **Total:** 1-2 days of focused work

---

## Next Steps

1. **Get stakeholder approval** (project owner)
2. **Choose version number** (recommend 5.1.0)
3. **Execute Phase 1-2** (backup & restructure)
4. **Execute Phase 3-4** (config updates & testing)
5. **Execute Phase 5** (first release)
6. **Execute Phase 6** (cleanup & communication)

---

## Questions to Resolve

- [ ] Confirm unified versioning approach (recommended: yes)
- [ ] Decide on first monorepo version (recommended: 5.1.0)
- [ ] Preserve git history or fresh start? (recommend: fresh start for simplicity)
- [ ] Communication plan for users/contributors?
- [ ] Timeline for migration?

---

*This plan transforms PyMoDAQ from a multi-repo nightmare into a single-repo dream while keeping the architectural separation the project owner wants.*
