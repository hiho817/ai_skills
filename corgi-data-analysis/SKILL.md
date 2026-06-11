---
name: corgi-data-analysis
description: >
  Use when analyzing CORGI quadruped robot experiment data end-to-end.
  Triggers on: inner EKF analysis, outer fusion node, odom_mapping, fusion/bv,
  body velocity, ekf/ba, ekf/bw, bias convergence, contact detection, foot contact,
  G1 G2 G3 G4 foot markers, gmo/contact_state, stance swing phase, gait analysis,
  lidar_odom quality, FAST-LIO2, LiDAR input, corgi experiment, 四足機器人分析,
  觸地偵測, 接觸狀態, 腳部標記, EKF 狀態分析, 融合節點分析, 里程計分析,
  VICON comparison, walk experiment analysis, trot experiment, experiment report,
  分析資料, 分析數據, 分析實驗, 幫我分析, analyze data, analyze experiment,
  analyze bag, analyze results, re-analyze, 重新分析, 看一下資料, 看資料.
  Output: structured Markdown analysis report.
argument-hint: "experiment path OR stage (contact | inner-ekf | outer-fusion | lidar | report)"
---

# CORGI Experiment Data Analysis

## Overview

Full-stack analysis of a CORGI walking / trotting experiment from raw ROS2 bag
and VICON CSV to a final **Markdown report**. Covers four analysis pillars:

| # | Pillar | Key signals | Reference |
|---|--------|------------|-----------|
| 1 | Contact Detection | VICON G1–G4, `/gmo/contact_state` | [contact-detection.md](./references/contact-detection.md) |
| 2 | Inner EKF | `/ekf`, `/ekf/ba`, `/ekf/bw`, `/ekf/orientation` | [inner-ekf.md](./references/inner-ekf.md) |
| 3 | Outer Fusion Node | `/odom_mapping`, `/fusion/bv` | [outer-fusion.md](./references/outer-fusion.md) |
| 4 | LiDAR Input | `/lidar_odom` | [lidar-input.md](./references/lidar-input.md) |

The final output follows the template in [report-template.md](./references/report-template.md).

> **語言規定**: 分析報告（analysis_report.md）一律使用**繁體中文（#tw-zh）**撰寫。
> 圖表標題、欄位名稱可保留英文，但段落說明、觀察與結論必須用繁體中文。

---

## Bag Topics Quick Reference

| Topic | Type | Description |
|-------|------|-------------|
| `/ekf` | `nav_msgs/msg/Odometry` | Inner EKF: position + velocity + covariance |
| `/ekf/orientation` | `geometry_msgs/msg/Quaternion` | Inner EKF attitude (higher rate) |
| `/ekf/ba` | `geometry_msgs/msg/Vector3` | Accelerometer bias (x, y, z) |
| `/ekf/bw` | `geometry_msgs/msg/Vector3` | Gyroscope bias (x, y, z) |
| `/odom_mapping` | `nav_msgs/msg/Odometry` | Outer fusion: fused odom (odom→base_link) |
| `/fusion/bv` | `geometry_msgs/msg/Vector3Stamped` | Outer fusion: body-frame velocity |
| `/lidar_odom` | `nav_msgs/msg/Odometry` | FAST-LIO2 (camera_init→base_link) |
| `/gmo/contact_state` | `corgi_msgs/msg/GMOContactStateStamped` | Estimated foot contact from GMO |
| `/trigger` | `corgi_msgs/msg/TriggerStamped` | Time-sync reference with VICON |
| `/imu_raw` | `corgi_msgs/msg/ImuStamped` | Raw IMU (optional input check) |
| `/motor/state` | `corgi_msgs/msg/MotorStateStamped` | Joint states (optional input check) |

---

## Step-by-Step Procedure

### Step 0 — Locate Experiment Assets

Confirm the following paths exist before starting:
- **ROS2 bag**: `experiments/<date>/<trial>/bags/<bag_name>/`
- **VICON CSV**: `experiments/<date>/<trial>/vicon/<name>.csv`
- **Results folder**: `experiments/<date>/<trial>/results/` (create if missing)

Also identify the **walking phase window** `[t_walk_start, t_walk_end]` (when robot
is on the ground) and the **bag start timestamp** `t0` (used to zero all time axes).

### Step 1 — Contact Detection

Use VICON G1–G4 foot markers (world-frame Z-height) to determine ground-truth
contact state. Compare with `/gmo/contact_state` from the bag.

See **[contact-detection.md](./references/contact-detection.md)** for the height
threshold algorithm, contact event extraction, and comparison metrics.

**Key outputs:**
- Per-leg contact timeline plot (VICON vs GMO side-by-side)
- Gait pattern diagram (stance/swing per leg vs time)
- Contact detection metrics: latency, false-positive rate, false-negative rate

### Step 2 — Inner EKF Analysis

Analyse all state variables published by the inner EKF node.

See **[inner-ekf.md](./references/inner-ekf.md)** for loading helpers and metric
definitions.

**Sub-analyses:**

| Sub | Signal | Compare against | Metric |
|-----|--------|----------------|--------|
| 2.1 Position | `/ekf` pose.position | VICON centroid (O1–O4) | RMSE XYZ, final drift |
| 2.2 Velocity | `/ekf` twist.linear | VICON SG-filter velocity | RMSE vx vy vz |
| 2.3 Attitude | `/ekf/orientation` (+ `/ekf` pose quat) | VICON body RPY | RMSE roll/pitch/yaw |
| 2.4 Accel bias | `/ekf/ba` x y z | — | Convergence time, steady-state value |
| 2.5 Gyro bias | `/ekf/bw` x y z | — | Convergence time, steady-state value |

### Step 3 — Outer Fusion Node Analysis

Analyse the outer EKF that fuses leg-odometry + LiDAR.

See **[outer-fusion.md](./references/outer-fusion.md)**.

**Sub-analyses:**

| Sub | Signal | Compare against | Metric |
|-----|--------|----------------|--------|
| 3.1 odom_mapping position | `/odom_mapping` pose | VICON centroid + inner `/ekf` | RMSE, max error |
| 3.2 odom_mapping yaw | `/odom_mapping` orientation | VICON yaw | RMSE |
| 3.3 Body velocity (bv) | `/fusion/bv` | VICON body-frame velocity | RMSE vx vy vz |

### Step 4 — LiDAR Input Quality

Assess `/lidar_odom` reliability as a fusion input.

See **[lidar-input.md](./references/lidar-input.md)**.

**Key checks:**
- Message rate (nominal vs actual)
- Trajectory smoothness (jump detection)
- Phase-gated validity: valid only during `[t_walk_start, t_walk_end]` when LiDAR
  has clear FOV (robot on ground, not lifted)
- XY trajectory overlay with VICON and `/ekf` in odom frame (requires
  T_{odom←camera_init} transform from fusion node log)

### Step 5 — Generate Markdown Report

Compile all plots and metrics into a report following
**[report-template.md](./references/report-template.md)**.

Save as: `experiments/<date>/<trial>/results/analysis_report.md`

---

## Common Pitfalls

- **Time zero**: always subtract `min(timestamp)` from raw nanosecond timestamps
  before plotting. Do NOT use wall-clock time directly.
- **LiDAR frame**: `/lidar_odom` is in `camera_init` frame. Always apply
  `T_{odom←camera_init}` before comparing with `/ekf` or VICON.
- **Walking phase only**: restrict quantitative metrics to `t < t_walk_end`.
  Post-lift data (robot in air) should be visually separated but excluded from RMSE.
- **VICON G marker NaN gaps**: foot markers can briefly disappear mid-flight.
  Interpolate short gaps (< 10 frames) or mark as "unknown" for contact logic.
- **GMO contact topic rate**: `/gmo/contact_state` may run at leg-odometry rate
  (~400 Hz). Downsample to VICON 500 Hz grid for fair comparison.
- **Bias sign**: `/ekf/ba` and `/ekf/bw` are subtracted from raw measurements.
  Plot raw values to assess convergence direction.
- **Covariance**: if `/ekf` covariance diagonals are available, plot 3σ bands on
  position and velocity traces.

---

## Lessons Learned (from 20260528 batch)

### 座標軸反向（RMSE 異常大時）

若某試驗的 RMSE 異常大（例如 > 100 cm），**第一步先懷疑座標系反向**：

1. 繪製估測器 X/Y 與 VICON X/Y 的時序圖：若兩者鏡像對稱（一正一負），即為反向。
2. 對估測器信號取負（flip）：`px, py, vx, vy, roll, pitch` 全部乘以 `-1`。
3. NEW（ESEKF）bags：設 `flip_new = {'px','py','vx','vy','roll','pitch'}`。
4. OLD（Legacy）bags：設 `flip_old = {'px','py','vx','vy'}`（姿態通常不受影響）。
5. 修正後 RMSE 應恢復正常（< 30 cm 量級）。

```python
# 在 analyze.py manifest 中標記需要 flip 的試驗
('FLAT_WLW_NEW_REAL_2', 'NEW_WLW', ..., {'px','py','vx','vy','roll','pitch'}, set()),
('FLAT_WLW_OLD_REAL_3', 'OLD_WLW', ..., set(), {'px','py','vx','vy'}),
```

### 軌跡圖只畫 XZ（不畫 XY）

- 平地實驗的水平平面（XY）軌跡資訊量有限（走直線），**改用 XZ 軌跡圖**（X 為前進方向，Z 為垂直）可看出：
  - 機器人步態起伏（Z 震盪）
  - LiDAR 累積垂直漂移
  - 估測器高度追蹤能力
- 儲存為 `fig_ekf_xz.png`（ESEKF）和 `fig_legacy_xz.png`（Legacy）。

### Z 軸初始對齊

VICON 的 Z 原點與估測器的 Z 原點通常不同（高度參考不同）。繪製 XZ 圖時，**必須將 VICON Z 平移至與估測器起始點對齊**，否則兩條曲線在垂直方向會有固定偏移，視覺上無法比較：

```python
# analyze_new 中
z_offset = epz[0] - vi_pz_e[0]          # 估測器起始 Z − VICON 起始 Z
err_pz   = epz - vi_pz_e - z_offset     # metrics 用（正確）
vi_z_plot = pos_vicon[vi_valid, 2] + z_offset  # 圖上 VICON Z（對齊後）
ax.plot(pos_vicon[vi_valid, 0], vi_z_plot, ...)  # ← 必須加 z_offset
ax.plot(epx, epz, ...)                           # 估測器不動
```

> **RMSE metrics 和圖上的 z_offset 是同一個值，務必兩者一致。**

### MPC 分析重點：最終停止位置

MPC 實驗為「定點移動（point-to-point）」任務，分析重點 **不是軌跡 RMSE，而是最終停止位置誤差**：

- 控制器以估測器回報的位置作為停止依據。若估測器有累積誤差，機器人會在未到目標前提前停止。
- 比較方式：`final_estimator_x` vs `final_VICON_x` vs `target_x`（e.g. 3.0 m）。
- 在 `metrics.json` 中儲存 `final_EKF_x`（NEW）/ `final_leg_x`（OLD）和 `final_VICON_x`。
- 報告中加入「終點 X 位置分析」表格，含每筆試驗的估測誤差與 VICON 停止誤差。

**20260528 MPC 結果（目標 3.0 m）：**
| 系統 | 估測器 final X | VICON 實際 final X | 停止誤差 |
|------|--------------|-------------------|---------|
| ESEKF | 3.004 ± 0.011 m | 2.988 ± 0.006 m | **1.2 cm** |
| Legacy | 3.005 ± 0.004 m | 2.758 ± 0.022 m | **24.2 cm** |

Legacy 里程計高估行進距離（腿式積分累積），導致提前停止，誤差 20× 大於 ESEKF。

### 試驗排除統計

若某試驗有已知異常（感測器故障、VICON 資料遺失、姿態偏移過大），應：
1. 仍完整分析並保留 `results/<exp_id>/` 資料夾與 `metrics.json`。
2. 在 manifest 中設 `exclude_stats=True`，讓 `group_stats()` 自動排除該筆。
3. 在報告的每次試驗表格中以上標 ¹ 標記，並附腳注說明原因。
