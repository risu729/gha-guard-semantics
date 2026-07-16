# GitHub Actions guard semantics

This repository tests how job-level conditions behave when a required-style
guard depends on a reusable workflow that fails, is skipped, or times out.

The experiment compares:

- `always() && contains(needs.*.result, ...)`
- `contains(needs.*.result, ...)` without `always()`
- `failure() || cancelled()`
- `always() && (failure() || cancelled())`
- `!cancelled() && contains(needs.*.result, ...)`

Every guard exits with status 1 if GitHub schedules it. A skipped guard
therefore means its job-level condition did not cause it to run.

## Results

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

The only condition that detected ordinary failure, a cancelled dependency,
and whole-workflow cancellation was:

```yaml
if: always() && (
  contains(needs.*.result, 'failure') ||
  contains(needs.*.result, 'cancelled')
)
```
