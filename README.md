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

Every input carries a default, so a caller can omit any input it does not
need to override.
The defaults are EpiAware's house values (CensoredDistributions.jl's
current settings), so a package matching that baseline writes a near-empty
caller and overrides only where it genuinely differs.

| Workflow | Purpose | Key inputs (default) |
|---|---|---|
| `tests.yml` | Test matrix (Julia versions x OS) + skip_quality | `julia_versions` (`["1","lts","pre"]`), `os` (ubuntu/windows/macOS), `experimental_versions` (`["pre"]`), `skip_quality`, `quality_version`, `fail_fast` |
| `downgrade.yml` | Test against oldest compatible deps | `julia_version` (`1.10`), `mode` (`deps`), `test_args` |
| `coverage.yml` | Single-run coverage + Codecov upload | `julia_version`, `test_args`, `coverage_directories` (`src`), `flags` (`unit`), `fail_ci_if_error` |
| `documentation.yml` | Documenter build/deploy + PR preview comment | `julia_version`, `julia_num_threads` |
| `docs-preview-cleanup.yml` | Delete closed-PR previews from gh-pages | `git_user_name`, `git_user_email` (both default to the derived `github-actions[bot]` identity) |
| `cancel-on-close.yml` | Cancel in-progress/queued runs on a PR's head branch when the PR is closed or merged | `head_ref` (defaults to the event's `pull_request.head.ref`) |
| `format-check.yml` | Python + pinned JuliaFormatter + pre-commit | `juliaformatter_version` (`2.5.5`), `extra_args` |
| `tagbot.yml` | JuliaRegistries TagBot | `lookback` (`3`) |
| `ad.yml` | AD gradient suite, internally matrixed over a backend list | `backends` (default: the six EpiAware backends below), `julia-version`, `test_project` (`test/ad`), `coverage_directories` (`src,ext`), `fail_fast` (`false`) |
| `ad-backend.yml` | Single-backend AD runner (one check per caller job) | `name`, `tag`, `flag`, `julia-version`, `test_project`, `coverage_directories` |
| `downstream.yml` | Reverse-dependency tests (opt-in), internally matrixed over a downstream list | `downstreams` (`[]`), `julia_version`, `os`, `coverage` |
| `release-nudge.yml` | Opens/refreshes a single issue when `main` has unreleased changes | `julia_version`, `registry` (`General`), `branch` (`main`), `label` (`release-nudge`), `stale_days` (`14`); optional secret `DOCUMENTER_KEY` |
| `major-version-tag.yml` | Maintains the moving `@v1` tag (runs here) | — |

`ad.yml` matrixes over its `backends` input internally
(`strategy.matrix.backend: ${{ fromJSON(inputs.backends) }}`), so a package
calls it ONCE instead of one caller per backend. The default `backends`
list is the EpiAware AD set: ForwardDiff, ReverseDiff (tape), Mooncake
forward, Mooncake reverse, Enzyme forward, Enzyme reverse. Override
`backends` with a JSON array of `{name, tag, flag}` objects only to test a
different set. `ad-backend.yml` (one backend per call, its own check name)
remains for packages that need per-backend checks rather than a matrix.

### Fast-failing and runner efficiency

Every job sets a `timeout-minutes` so a hung run ("runner lost
communication") is killed and its slot freed instead of sitting until the
6h ceiling: 60 for the test and AD matrices, 45 for coverage / downgrade /
docs, 20 for the format check, 10-15 for the housekeeping jobs.

`tests.yml` defaults `fail_fast: false`. The matrix mixes the required
merge-gate legs (Julia 1/lts on ubuntu) with non-required legs (macOS,
windows, and pre-release Julia `pre`, where Mooncake/AD are
expected-flaky). With fail-fast on, a non-required leg failing would cancel
the required ubuntu legs, so a PR could never confirm essential-green and
the merge queue would stall (CD #766). Pre-release legs are additionally
`continue-on-error` (via `experimental_versions`, default `["pre"]`) so
they never block; fail-fast false is what stops them *cancelling* siblings.
`ad.yml` defaults `fail_fast: false` so a single backend break still
reports the full per-backend picture; set `fail_fast: true` on the caller
to free runners faster during a backlog at the cost of that full coverage.

Two complementary mechanisms free runners from stale runs:

1. **Concurrency groups (push-supersede).** Cancelling a superseded run on
a new push is set on the *caller* workflows, not in the reusables: inside a
reusable, `github.workflow`/`github.ref` resolve to the caller's context, so
duplicating a `concurrency:` group here would collide with the caller's own
group. Each consuming caller must carry
`concurrency: { group: ${{ github.workflow }}-${{ github.ref }},
cancel-in-progress: true }` (see the caller examples below). This cancels a
prior run only when a NEW push lands on the same ref; it does nothing when a
PR is closed or merged without a further push.

2. **Cancel on close/merge.** `cancel-on-close.yml` covers that gap. When a
PR is closed or merged, GitHub stops scheduling new runs but leaves any
in-progress and queued runs going until they finish or hit the 6h ceiling.
The reusable cancels them by listing in-progress/queued runs on the closed
PR's head branch and `gh run cancel`-ing each. It uses only the runner's
preinstalled `gh` CLI with the default `GITHUB_TOKEN` (no third-party
action), and needs `actions: write`. Callers trigger it on
`pull_request: types: [closed]`, the same trigger as
`docs-preview-cleanup.yml`; the cancel run executes in the base-branch
context, not on the head ref, so it never cancels itself.
### Downstream (reverse-dependency) tests

`downstream.yml` catches the case where a change to a base package breaks
a package that depends on it, mirroring SciML's `Downstream.yml`. For each
entry in the `downstreams` JSON list it checks out the downstream,
`Pkg.develop`s this PR's version of the base package into the downstream
env, then runs the downstream test suite (with the entry's `group` set as
the `GROUP` env). A failed dependency resolve is read as a deliberate
breaking change and the job exits green, as in SciML; real test failures
still red the run unless the entry sets `allow_fail: true`.

It is heavy, so it is opt-in: callers trigger it on `workflow_dispatch` or
a `downstream`/`reverse-deps` label, not on every push. With an empty
`downstreams` list (the default) the job is skipped, so a package can wire
the caller before it has any registered downstreams. Each entry takes
`repo` (`owner/name`), optional `group`, and optional `allow_fail`:

```yaml
downstreams: >-
  [{"repo":"EpiAware/EpiAwarePackageTools.jl","group":"All"}]
```

### Release nudge

`release-nudge.yml` runs on a schedule (the caller sets the cron) and on
`workflow_dispatch`. It compares `Project.toml`'s version and the
commits on `branch` since the latest GitHub release/tag (and,
best-effort, the version registered in the `registry` input's Julia
registry) against `main`. When there are unreleased commits it opens
(or refreshes) a single issue labelled `label`, telling a maintainer
the released vs `Project.toml` version, how many commits are unreleased
with a compare link and a short recent-commit list, whether a version
bump is still needed or only registration is outstanding, and the
concrete `/version`/`/register` steps. When nothing is unreleased it
closes any open nudge issue instead and stays silent.

An open nudge issue is never edited in place: it embeds its computed
state (`Project.toml` version + unreleased commit count) in an HTML
comment, and a run that finds the state has changed, or finds the
issue has sat open for at least `stale_days` days regardless, closes
it with a short "superseded" comment and opens a fresh one. This keeps
the issue honest without an unbounded edit history.

The issue also warns when the repository has no `DOCUMENTER_KEY`
secret. Without it TagBot tags with the Actions app token, GitHub
raises no push event for that tag, and the release publishes no
versioned docs. Only the secret's presence is read, never its value,
and the check relies on the caller passing `secrets: inherit` — a
caller that forwards nothing gets the warning whether or not the key
exists.

The issue body is built so it can never contain a literal `@`: any
repo-derived text that could carry an unexpected mention (commit
subject lines, and the release tag name itself) has every `@` replaced
before it reaches the body, and the process to trigger registration is
described by name (the Register workflow, the `/register` slash
command) rather than by writing out the registry bot's handle
anywhere.

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
    secrets: inherit
```

The caller omits `julia_versions` / `os` entirely and inherits the default
matrix. A package that needs a different grid passes only the input it
changes.

All AD backends in one call (the default `backends` list runs the six
EpiAware backends as a matrix, each a `ad / <name>` check, e.g.
`ad / ForwardDiff`):

```yaml
name: AD
on:
  push:
    branches: [main]
  pull_request:
  merge_group:
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  ad:
    uses: EpiAware/.github/.github/workflows/ad.yml@v1
    secrets: inherit
```

Cancel a PR's runs when it is closed or merged. Add this once per package;
it covers every workflow on that PR's branch, so one caller is enough:

```yaml
name: Cancel on Close
on:
  pull_request:
    types: [closed]
jobs:
  cancel:
    uses: EpiAware/.github/.github/workflows/cancel-on-close.yml@v1
    secrets: inherit
    permissions:
      actions: write
      contents: read
```

### What stays per-repo

Package-specific configuration is NOT centralised: the set of AD backends/
tags a package tests, `compat` bounds, the benchmark comparison/comment
scripts, docs content, and the `JULIA_NUM_THREADS` a docs build needs.
These live in each consuming package as caller inputs or local files.
