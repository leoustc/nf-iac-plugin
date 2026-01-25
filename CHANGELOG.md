# Changelog

## 0.3.1
- Added `ram_factor` and moved `cpu_factor`/`ram_factor` to `iac {}`.
- Updated resource summary to include start time and runtime, and to use trace `%CPU` directly.
- Added persistent `pipeline-resource-usage-<run-id>.txt` output with the same columns as the final summary.
