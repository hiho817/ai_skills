---
name: vicon-robot-analysis
description: >
  Use when analyzing VICON motion capture CSV data from quadruped robot experiments.
  Triggers on: VICON CSV, force plate, trajectory markers, O1 O2 O3 O4, ground plane,
  trigger sync, ROS2 bag, Savitzky-Golay velocity, quadruped robot, corgi, 四足機器人,
  ground1 ground2 ground3 ground4, leg odometry, world frame transformation,
  walk experiment, hip markers, foot markers, RPY attitude, body velocity.
argument-hint: "CSV path or step (csv-parse | ground-plane | time-sync | kinematics | force-plates)"
---

# VICON Quadruped Robot Data Analysis

## Overview

This skill guides end-to-end analysis of **VICON motion capture data** (`*.csv`) paired
with **ROS2 bag files** from quadruped robot (corgi) walking experiments.

Pipeline stages:

| # | Stage | Reference |
|---|-------|-----------|
| 1 | Parse VICON CSV (Devices + Trajectories sections) | [CSV Parsing](./references/csv-parsing.md) |
| 2 | Identify markers (body, feet, ground) | [Markers & World Frame](./references/world-frame.md) |
| 3 | Fit ground plane → Z-up world frame | [Markers & World Frame](./references/world-frame.md) |
| 4 | **Robot-centric alignment** — 以機器人初始位置為原點、初始 heading 為 +X | [Markers & World Frame](./references/world-frame.md) |
| 5 | Synchronise VICON time to ROS2 via Tigger marker | [Time Sync](./references/time-sync.md) |
| 6 | Compute body centroid, attitude (RPY), velocity | [Kinematics](./references/kinematics.md) |
| 7 | Access force plate columns | [Force Plates](./references/force-plates.md) |

---

## Session-Specific Constants

| Parameter | Value |
|-----------|-------|
| Trajectory sample rate | **500 Hz** |
| Force plate sample rate | **1000 Hz** |
| Trigger marker name | **`Tigger`** (known typo — NOT "Trigger") |
| Hip marker order | O1 = LF, O2 = RF, O3 = RH, O4 = LH |
| Foot marker order | G1 = LF, G2 = RF, G3 = RH, G4 = LH |
| Body X axis | LH(O4) → LF(O1) = forward |
| Body Y axis | $Z_b \times X_b$ = leftward |
| Body Z axis | plane normal = upward |
| Position units | mm (raw VICON output) |
| Minimum ground points | 3 of Ground1–Ground4 |
| ROS2 trigger topic | `/trigger` (1 msg, `corgi_msgs/msg/TriggerStamped`) |

---

## Step-by-Step Procedure

### Step 1 — Parse the CSV

Load both sections with `find_section_row()`. See **[CSV Parsing](./references/csv-parsing.md)**
for the exact `pd.read_csv` recipe and the `build_marker_col_map` helper.

### Step 2 & 3 — Ground Plane & Z-up World Frame

Read the four static `Ground1–Ground4` markers (first valid frames) and fit the plane via SVD.
Build `R_ground` to transform all marker positions to a Z-up world frame.
See **[Markers & World Frame](./references/world-frame.md)** Steps 1–3.

### Step 4 — Robot-Centric Alignment（機器人初始位置為原點）

> ⚠️ **必做**：VICON lab 的 X/Y 軸方向與機器人初始朝向不一定一致。
> 機器人走的是自身的 +X 方向，必須將世界座標系對齊機器人初始 heading。

1. 取 t=0 時的 O1（LF）與 O4（LH）位置，計算 heading 向量（投影到 XY 平面）
2. 計算偏轉角，建立繞 Z 軸的旋轉矩陣 `R_heading`
3. 以機器人初始質心（O1–O4 平均）為平移原點
4. 使用 `to_robot_world()` 取代所有 `to_world()` 呼叫

完整 code 見 **[Markers & World Frame](./references/world-frame.md)** Steps 4–7。

### Step 5 — Time Synchronisation

Detect the first frame where `Tigger` becomes visible (VICON side) and read the single
`/trigger` ROS2 message timestamp (bag side). The offset `dt = t_ros - t_vicon` applies to
both the 500 Hz trajectory axis and the 1000 Hz force plate axis.
See **[Time Sync](./references/time-sync.md)**.

### Step 6 — Body Kinematics

1. Transform O1–O4 to **robot-centric world frame** (use `to_robot_world()`).
2. Compute per-frame centroid; origin is already zeroed from Step 4.
3. Fit per-frame body rotation (SVD on hip plane) → extract RPY via ZYX Euler.
4. Differentiate position with `savgol_filter(deriv=1)` (window=11, poly=3 at 500 Hz).
5. Project world-frame velocity into body frame with $v_b = R_b^T v_w$.

See **[Kinematics](./references/kinematics.md)** for full code.

### Step 7 — Force Plates

Plate column order in the CSV: **FP4, FP2, FP1, FP3** (export artifact).
Each plate has 9 columns: Fx Fy Fz Mx My Mz Cx Cy Cz.
See **[Force Plates](./references/force-plates.md)**.
