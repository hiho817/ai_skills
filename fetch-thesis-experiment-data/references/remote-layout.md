# Remote analysis workspace reference

## Source hierarchy

| Remote path | Role | Thesis authority |
|---|---|---|
| `thesis_exp/physical_exp/` | Curated, independently reproducible physical experiments and aggregate reports | Official and authoritative |
| `thesis_exp/simulation/` | Curated simulation experiments | Official only for simulation tasks |
| `experiments/` | Complete dated experiment archive, historical analyses, scripts, and reports | Reference only |
| `tools/corgi_analysis/` | Shared bag, VICON, plotting, and metric utilities | Method reference |

## Official physical-experiment layout

```text
thesis_exp/physical_exp/
├── README.md
├── analyze_report.py
├── common/corgi_analysis/
├── experiments/<experiment-id>/
│   ├── bags/
│   ├── vicon/
│   ├── analyze.py
│   ├── analyze_impl.py
│   └── results/<experiment-id>/
│       ├── metrics.json
│       └── figures and reports
└── results/
    ├── analysis_report.md
    ├── contact_state_exp.md
    └── aggregate PDF figures
```

The single-experiment `analyze.py` dispatches to `analyze_impl.py`. The aggregate `analyze_report.py` discovers `experiments/*/results/*/metrics.json` and writes the official summary report.

## Common metric fields

- Identity: `exp_id`, `group`, `bag`, `T_END`, `data_start`, `data_end`.
- Position: axis RMSE, `RMSE_3D_cm` or `RMSE_2D_cm`, maximum error, and final EKF/VICON position.
- Velocity: axis RMSE, `RMSE_3D`, analysis window, and peak velocity.
- Attitude: roll, pitch, and yaw RMSE in degrees.
- Bias: accelerometer bias `ba` and gyroscope bias `bw` summaries.
- Odometry and fusion: `odom_pos`, `fusion_bv`.
- LiDAR and calibration: message rate, residuals, and `T_CO`.
- Contact state: per-leg `N`, `TP`, `TN`, `FP`, `FN`, `acc`, and valid time range.
- Statistical eligibility: `exclude_stats`.

Read the actual JSON before using a field because older or specialized experiments may omit sections or use 2D instead of 3D position metrics.

## Interpretation safeguards

- Preserve units exactly: position reports commonly use centimetres, velocity uses metres per second, and attitude uses degrees.
- Treat a group summary such as `mean ± sample standard deviation` differently from a pooled statistic across all time samples.
- Record contact-state thresholds and VICON ground-truth height when reporting accuracy, FP, or FN rates.
- Follow experiment-specific inclusion and exclusion notes in the official reports.
- If two official reports disagree, report both source paths and values, then request a decision or regenerate only with approval.

## Suggested traceability fields

Record these in each local experiment note:

- Retrieval date.
- Remote host and absolute source path.
- Remote Git commit SHA and whether the working tree was dirty.
- Experiment ID, group, and bag ID.
- Metric name, value, unit, and evaluation window.
- Inclusion/exclusion status and aggregation method.
- Local figure path and remote/local SHA-256 hash.
- For derivative figures, transformation tool, reproducible procedure, and parameters.
- For promoted results, original reference-source path, approval context, and final authoritative path.
