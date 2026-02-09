# Changelog

## 1.0.2
- Replace AWS S3 CLI usage in `nxf_work.sh` with `rclone` (auto-install on workers).
- `nxf_work.sh` now writes rclone config once and uses `bucket:bucket/path` remote paths.
- Pipeline resource usage output includes `%DISK` and preserves full precision for all `%` columns.
- Added visualization notebook (`visualization.ipynb`).

## 0.3.5
- Added visualization notebook (`visualization.ipynb`).
- Pipeline resource usage output now includes `%DISK` and preserves full precision for all `%` columns.
- `nxf_work.sh` now uses `rclone` for S3 transfers (auto-installs if missing).
- `nxf_work.sh` no longer uses AWS S3 CLI; all object-store transfers go through `rclone`.

## 0.3.3
- Job dir naming is now `.nextflow/iac/<run_timestamp>/<task_id>/` (jobId stays `<aa>/<hash>`).
- Added richer resource and trace metrics, including CPU time, estimated cycles, LCPU, peak RSS/VMEM, and disk ratio.
- Added per-run resource and trace usage files and a concise end-of-run summary table.

## 0.3.1
- Added `ram_factor` and moved `cpu_factor`/`ram_factor` to `iac {}`.
- Updated resource summary to include start time and runtime, and to use trace `%CPU` directly.
- Added persistent `pipeline-resource-usage-<run-id>.txt` output with the same columns as the final summary.
