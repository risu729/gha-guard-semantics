# GitHub Actions guard semantics

This repository tests how job-level conditions behave when a required-style
guard depends on a reusable workflow that fails, is skipped, or times out.

The experiment compares:

- `always() && contains(needs.*.result, ...)`
- `contains(needs.*.result, ...)` without `always()`
- `failure() || cancelled()`
- `always() && (failure() || cancelled())`
- `!cancelled() && contains(needs.*.result, ...)`
- `failure()` (the LayerOne baseline)
- `${{ !cancelled() && (needs.dependency.result == 'failure' || needs.dependency.result == 'cancelled') }}` (the exact proposed LayerOne condition, with its dependency named `dependency`)

Every guard exits with status 1 if GitHub schedules it. A skipped guard
therefore means its job-level condition did not cause it to run.

## Current results: 2026-10-08 (JST)

All workflow runner labels are explicitly `ubuntu-26.04`. The timeout and
manual-cancellation dependency logs report Ubuntu 26.04.1, image
`20260927.149.1`, and Actions runner **2.337.0**. Runs started on
2026-10-07 UTC / 2026-10-08 JST at commit
[`a7c8dd5`](https://github.com/risu729/gha-guard-semantics/commit/a7c8dd5ac4ad535c06cbef49326a7a27b0bfcd11).

Every scheduled guard deliberately exits 1. `failure` in the table is the
expected detection result, not an unexpected test failure.

| Dependency/run outcome | `always` + `needs` | `needs` only | `failure() \|\| cancelled()` | `always` + status functions | `!cancelled` + `needs` | `failure()` baseline | Exact proposed condition |
| --- | --- | --- | --- | --- | --- | --- | --- |
| [failure](https://github.com/risu729/gha-guard-semantics/actions/runs/37651736040) | failure | skipped | failure | failure | failure | failure | failure |
| [skipped](https://github.com/risu729/gha-guard-semantics/actions/runs/37651740830) | skipped | skipped | skipped | skipped | skipped | skipped | skipped |
| [success](https://github.com/risu729/gha-guard-semantics/actions/runs/37651745780) | skipped | skipped | skipped | skipped | skipped | skipped | skipped |
| [dependency timeout/cancellation](https://github.com/risu729/gha-guard-semantics/actions/runs/37651752433) | failure | skipped | skipped | skipped | failure | skipped | failure |
| [whole workflow manually cancelled](https://github.com/risu729/gha-guard-semantics/actions/runs/37651757258) | failure | cancelled | failure | failure | cancelled | cancelled | cancelled |

The timeout case has a one-minute dependency job timeout. Its report job
prints `dependency=cancelled`, while `failure()` and even
`failure() || cancelled()` are skipped. The exact proposed guard runs and
fails, detecting the cancelled reusable-workflow dependency.

The `manual-cancel` mode sleeps with a ten-minute timeout. This run was
manually cancelled using `gh run cancel` while the dependency was sleeping
(the log shows the sleep starting at 16:25:01 UTC and cancellation at
16:25:52 UTC). Both `!cancelled()` guards ended as `cancelled` with no
runner assigned, while the `always()` guards and `failure() || cancelled()`
started runners and failed. This distinguishes whole-run cancellation from
a dependency timing out; the top-level run conclusion is `cancelled` in
both cases and is insufficient to distinguish them.

The preferred guard for a single reusable-workflow dependency is therefore:

```yaml
if: ${{ !cancelled() && (needs.dependency.result == 'failure' || needs.dependency.result == 'cancelled') }}
```

Use `ci` instead of `dependency` when the caller job is named `ci`. The status
function also prevents GitHub from adding an implicit `success()` gate;
see [GitHub's status-check function documentation](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions#status-check-functions).

[Saved job results and runner assignments](results/2026-10-08.json) preserve
the observed matrix after Actions logs expire. The separate existing
`allowed-timeout` experiment is outside this five-case matrix.

To repeat, dispatch `experiment.yml` at the commit/branch containing these
workflows with `mode=failure`, `skipped`, `success`, `timeout`, and
`manual-cancel`. For the last case, wait until the dependency is running,
then cancel that run (not the timeout run):

```sh
gh workflow run experiment.yml --ref ci/guard-semantics-ubuntu-26.04 -f mode=manual-cancel
gh run cancel <manual-cancel-run-id>
```

## Ordinary job timeout

A direct dependency on an ordinary job (no reusable workflow and no
`continue-on-error`) has the same timeout behavior on `ubuntu-26.04` /
Actions runner 2.337.0. In the `direct-timeout` mode, the job sleeps for ten
minutes with a one-minute job timeout. The report printed
`dependency=cancelled`; `failure()` skipped its guard, while the explicit
`!cancelled()` plus dependency-result condition ran and failed as intended.
This problem is therefore not specific to reusable workflows.

[Run](https://github.com/risu729/gha-guard-semantics/actions/runs/37658329705) /
[saved results](results/2026-10-08-ordinary-timeout.json).

## Historical results: 2026-07-16

Observed on GitHub Actions on 2026-07-16:

| Dependency/run outcome | `always` + `needs` | `needs` only | status functions | `always` + status functions | `!cancelled` + `needs` |
| --- | --- | --- | --- | --- | --- |
| failure | failure | skipped | failure | failure | failure |
| skipped | skipped | skipped | skipped | skipped | skipped |
| success | skipped | skipped | skipped | skipped | skipped |
| dependency timeout/cancellation | failure | skipped | skipped | skipped | failure |
| whole workflow cancelled | failure | cancelled | failure | failure | cancelled |

Runs:

- [failure](https://github.com/risu729/gha-guard-semantics/actions/runs/29473604423)
- [skipped](https://github.com/risu729/gha-guard-semantics/actions/runs/29473606103)
- [success](https://github.com/risu729/gha-guard-semantics/actions/runs/29473607620)
- [dependency timeout/cancellation](https://github.com/risu729/gha-guard-semantics/actions/runs/29473609173)
- [whole workflow cancelled](https://github.com/risu729/gha-guard-semantics/actions/runs/29473705399)

The original experiment used the following guard to detect ordinary failure,
a cancelled dependency, and whole-workflow cancellation. It intentionally
runs even after the whole workflow is cancelled; use the cancellation-aware
condition above when that extra runner is undesirable:

```yaml
if: always() && (
  contains(needs.*.result, 'failure') ||
  contains(needs.*.result, 'cancelled')
)
```
