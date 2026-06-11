# Markdown Report Template

Fill this template after completing all four analysis pillars.
Save as `experiments/<date>/<trial>/results/analysis_report.md`.

---

```markdown
# CORGI Experiment Analysis Report

**Date:** YYYY-MM-DD  
**Experiment:** `<trial_name>` (e.g., `walk_2m_01`)  
**Bag (replay):** `<bag_name>`  
**VICON CSV:** `<csv_filename>`  
**Walking phase:** t = 0 – XX s  
**Analysis script:** `<script_name>.py`

---

## System Architecture

```
/imu_raw ─┐
/motor/state ──┤──► corgi_leg_odom ──► /ekf ─────────────────► (odom→base_link)
/trigger ──────┘                              │
/gmo/contact_state                            ▼
                          /lidar_odom ──► corgi_fusion_node ──► /odom_mapping
                         (FAST-LIO2,                          /fusion/bv
                          camera_init frame)
```

---

## 1. Contact Detection

### 1.1 VICON Ground-Truth Contact (G1–G4)

Contact threshold: **XX mm** above ground plane.

![Contact Timeline](fig_contact_timeline.png)

| Leg | Stance ratio (walk phase) | Mean stance duration [ms] |
|-----|--------------------------|--------------------------|
| LF (G1) | X.X% | XXX |
| RF (G2) | X.X% | XXX |
| RH (G3) | X.X% | XXX |
| LH (G4) | X.X% | XXX |

### 1.2 GMO vs VICON Contact

![Foot Height](fig_foot_height.png)

| Leg | Precision | Recall | Mean Latency (ms) |
|-----|-----------|--------|-------------------|
| LF  | X.XX      | X.XX   | XX.X              |
| RF  | X.XX      | X.XX   | XX.X              |
| RH  | X.XX      | X.XX   | XX.X              |
| LH  | X.XX      | X.XX   | XX.X              |

**Observations:**
- [Describe any systematic early/late detection, false positives during swing, etc.]

---

## 2. Inner EKF Analysis

### 2.1 Position

![EKF Position XY](fig_ekf_xy.png)
![EKF Position Time](fig_ekf_pos_time.png)

| Metric | Value |
|--------|-------|
| RMSE X (vs VICON) | X.XXX m |
| RMSE Y (vs VICON) | X.XXX m |
| RMSE Z (vs VICON) | X.XXX m |
| RMSE 3D (vs VICON) | X.XXX m |
| Max 3D error | X.XXX m |
| Final position (EKF) | (X.XX, X.XX) m |
| Final position (VICON) | (X.XX, X.XX) m |

**Observations:**
- [Describe drift direction, Z-axis behavior, phase discontinuities, etc.]

### 2.2 Velocity

![EKF Velocity](fig_ekf_vel.png)

| Metric | Value |
|--------|-------|
| RMSE vx (vs VICON) | X.XXX m/s |
| RMSE vy (vs VICON) | X.XXX m/s |
| RMSE vz (vs VICON) | X.XXX m/s |
| Peak forward speed | X.XX m/s |

**Observations:**
- [Describe velocity tracking quality, oscillations, step/stride patterns, etc.]

### 2.3 Attitude (RPY)

![EKF Attitude](fig_ekf_rpy.png)

| Metric | Value |
|--------|-------|
| RMSE roll (vs VICON) | X.X° |
| RMSE pitch (vs VICON) | X.X° |
| RMSE yaw (vs VICON) | X.X° |
| Final yaw (EKF) | X.X° |
| Final yaw (VICON) | X.X° |

**Observations:**
- [Describe roll/pitch coupling with stride, yaw drift rate, etc.]

### 2.4 Accelerometer Bias (ba)

![Accel Bias](fig_ekf_ba.png)

| Axis | Initial [m/s²] | Steady-state [m/s²] | Std [m/s²] |
|------|---------------|---------------------|------------|
| x    | X.XXXX        | X.XXXX              | X.XXXXX    |
| y    | X.XXXX        | X.XXXX              | X.XXXXX    |
| z    | X.XXXX        | X.XXXX              | X.XXXXX    |

### 2.5 Gyroscope Bias (bw)

![Gyro Bias](fig_ekf_bw.png)

| Axis | Initial [rad/s] | Steady-state [rad/s] | Std [rad/s] |
|------|----------------|----------------------|-------------|
| x    | X.XXXXX        | X.XXXXX              | X.XXXXXX    |
| y    | X.XXXXX        | X.XXXXX              | X.XXXXXX    |
| z    | X.XXXXX        | X.XXXXX              | X.XXXXXX    |

**Observations:**
- [Describe convergence speed, residual oscillation, any axis that did not converge, etc.]

---

## 3. Outer Fusion Node

### 3.1 odom_mapping Position

![Fusion XY](fig_fusion_xy.png)
![Fusion Position Time](fig_fusion_pos_time.png)

| Metric | Value |
|--------|-------|
| RMSE 2D vs VICON | X.XXX m |
| Max 2D error vs VICON | X.XXX m |
| RMSE 2D vs EKF | X.XXX m |
| Final position (odom_mapping) | (X.XX, X.XX) m |

**Observations:**
- [Compare with inner EKF: does LiDAR fusion improve accuracy?]
- [Describe behavior after t_walk_end (robot lifted)]

### 3.2 odom_mapping Yaw

![Fusion Yaw](fig_fusion_yaw.png)

| Metric | Value |
|--------|-------|
| RMSE yaw vs VICON | X.X° |
| RMSE yaw vs EKF | X.X° |
| Final yaw (odom_mapping) | X.X° |

### 3.3 Body Velocity (fusion/bv)

![Fusion BV](fig_fusion_bv.png)

| Metric | Value |
|--------|-------|
| RMSE vx vs VICON | X.XXX m/s |
| RMSE vy vs VICON | X.XXX m/s |
| RMSE vx vs EKF | X.XXX m/s |
| RMSE vy vs EKF | X.XXX m/s |

**Observations:**
- [Does /fusion/bv outperform or match /ekf velocity? Describe smoothing effect.]

---

## 4. LiDAR Input Quality

![LiDAR Trajectory](fig_lidar_xy.png)
![LiDAR Z](fig_lidar_z.png)
![LiDAR Interval](fig_lidar_interval.png)

| Metric | Value |
|--------|-------|
| Total messages | XX |
| Mean message interval | XX.X ms |
| Gaps > 500 ms | X |
| Position jumps > 5 cm | X |
| XY RMSE vs VICON (walk phase) | X.XXX m |
| Z drift (walk phase) | X.XXX m |

**Observations:**
- [Describe coverage, dropout events, match quality, post-lift behavior.]

---

## 5. Summary & Conclusions

### Key Metrics

| Component | RMSE (vs VICON) | Note |
|-----------|----------------|------|
| Inner EKF position | X.XXX m | walk phase only |
| Outer fusion position | X.XXX m | walk phase only |
| Inner EKF velocity | X.XXX m/s | |
| Fusion body velocity | X.XXX m/s | |
| Inner EKF yaw | X.X° | |
| Outer fusion yaw | X.X° | |
| Contact recall (mean) | X.XX | |

### Key Findings

1. **Contact detection**: [One sentence summary]
2. **Inner EKF**: [One sentence summary — position accuracy, bias convergence]
3. **Outer fusion**: [One sentence — does LiDAR improve over inner EKF?]
4. **LiDAR quality**: [One sentence — reliable input or degraded?]

### Recommendations

- [ ] [Action item 1]
- [ ] [Action item 2]
- [ ] [Action item 3]
```
