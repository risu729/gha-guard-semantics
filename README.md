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
