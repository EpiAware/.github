# .github

Organisation-level configuration and shared CI for
[EpiAware](https://github.com/EpiAware).

## Reusable CI workflows

`.github/workflows/` hosts reusable `workflow_call` workflows shared across
the EpiAware Julia packages, modelled on the
[`SciML/.github`](https://github.com/SciML/.github) pattern at smaller
scale. Consuming packages keep only thin caller workflows that invoke these
with their package-specific inputs and `secrets: inherit`.

See issue
[EpiAware/.github#8](https://github.com/EpiAware/.github/issues/8) for the
design and rationale.

### Available workflows

| Workflow | Purpose | Key inputs |
|---|---|---|
| `tests.yml` | Test matrix (Julia versions x OS) + skip_quality | `julia_versions`, `os`, `experimental_versions`, `skip_quality`, `quality_version`, `fail_fast` |
| `downgrade.yml` | Test against oldest compatible deps | `julia_version`, `mode`, `test_args` |
| `coverage.yml` | Single-run coverage + Codecov upload | `julia_version`, `test_args`, `coverage_directories`, `flags`, `fail_ci_if_error` |
| `documentation.yml` | Documenter build/deploy + PR preview comment | `julia_version`, `julia_num_threads` |
| `docs-preview-cleanup.yml` | Delete closed-PR previews from gh-pages | `git_user_name`, `git_user_email` |
| `format-check.yml` | Python + pinned JuliaFormatter + pre-commit | `juliaformatter_version`, `extra_args` |
| `tagbot.yml` | JuliaRegistries TagBot | `lookback` |
| `ad-backend.yml` | Per-backend AD gradient suite + coverage | `name`, `tag`, `flag`, `julia-version`, `test_project`, `coverage_directories` |
| `major-version-tag.yml` | Maintains the moving `@v1` tag (runs here) | — |

### Versioning

Callers pin to the moving major tag `@v1`. `major-version-tag.yml` moves
the `v1` tag to the latest `v1.x.y` release tag when one is published, so
consumers track fixes without pinning a SHA. To bootstrap, create the first
release: tag `v1.0.0` (`git tag v1.0.0 && git push origin v1.0.0`); the
workflow then creates/moves `v1`. Until `v1` exists, callers may pin
`@main`.

### Caller example

A consuming package's `.github/workflows/test.yaml`:

```yaml
name: Test
on:
  push:
    branches: [main]
  pull_request:
  merge_group:
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  test:
    uses: EpiAware/.github/.github/workflows/tests.yml@v1
    with:
      julia_versions: '["1", "lts", "pre"]'
    secrets: inherit
```

For one AD backend (one thin caller per backend, so each gets its own
check; the caller job id and the `name` input determine the check name,
e.g. `forwarddiff / ForwardDiff`):

```yaml
name: AD ForwardDiff
on:
  push:
    branches: [main]
  pull_request:
  merge_group:
jobs:
  forwarddiff:
    uses: EpiAware/.github/.github/workflows/ad-backend.yml@v1
    with:
      name: ForwardDiff
      tag: forwarddiff
      flag: ad-forwarddiff
    secrets: inherit
```

### What stays per-repo

Package-specific configuration is NOT centralised: the set of AD backends/
tags a package tests, `compat` bounds, the benchmark comparison/comment
scripts, docs content, and the `JULIA_NUM_THREADS` a docs build needs.
These live in each consuming package as caller inputs or local files.
