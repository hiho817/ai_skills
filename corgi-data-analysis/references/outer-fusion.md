# Outer Fusion Node Analysis — Reference

## Topics

| Topic | Type | Rate | Content |
|-------|------|------|---------|
| `/odom_mapping` | `nav_msgs/msg/Odometry` | ~30–50 Hz (LiDAR rate) | fused pose + velocity in `odom` frame |
| `/fusion/bv` | `geometry_msgs/msg/Vector3Stamped` | ~30–50 Hz | body-frame velocity estimate |

`/odom_mapping` is the outer EKF output that fuses `/ekf` (inner leg-odom EKF)
with `/lidar_odom` (FAST-LIO2). It runs at the LiDAR update rate.

**Important**: `/odom_mapping` is published in the `odom` frame (same as `/ekf`)
after the coordinate-frame fix. Verify with `msg.header.frame_id == "odom"`.

---

## Loading Helpers

```python
from nav_msgs.msg import Odometry
from geometry_msgs.msg import Vector3Stamped

def load_odom_mapping(db_path, t0_ns, t_walk_end=None):
    conn = sqlite3.connect(db_path)
    cur  = conn.cursor()
    cur.execute("SELECT id FROM topics WHERE name='/odom_mapping'")
    tid  = cur.fetchone()[0]
    cur.execute(f"SELECT timestamp, data FROM messages WHERE topic_id={tid} ORDER BY timestamp")
    rows = cur.fetchall(); conn.close()

    out = {k: [] for k in ['t','px','py','pz','vx','vy','vz','qw','qx','qy','qz']}
    for ts, data in rows:
        msg = deserialize_message(data, Odometry)
        t   = (ts - t0_ns) / 1e9
        out['t'].append(t)
        out['px'].append(msg.pose.pose.position.x)
        out['py'].append(msg.pose.pose.position.y)
        out['pz'].append(msg.pose.pose.position.z)
        out['vx'].append(msg.twist.twist.linear.x)
        out['vy'].append(msg.twist.twist.linear.y)
        out['vz'].append(msg.twist.twist.linear.z)
        out['qw'].append(msg.pose.pose.orientation.w)
        out['qx'].append(msg.pose.pose.orientation.x)
        out['qy'].append(msg.pose.pose.orientation.y)
        out['qz'].append(msg.pose.pose.orientation.z)

    for k in out: out[k] = np.array(out[k])
    if t_walk_end: mask = out['t'] < t_walk_end; [out.__setitem__(k, out[k][mask]) for k in out]
    return out


def load_fusion_bv(db_path, t0_ns, t_walk_end=None):
    conn = sqlite3.connect(db_path)
    cur  = conn.cursor()
    cur.execute("SELECT id FROM topics WHERE name='/fusion/bv'")
    tid  = cur.fetchone()[0]
    cur.execute(f"SELECT timestamp, data FROM messages WHERE topic_id={tid} ORDER BY timestamp")
    rows = cur.fetchall(); conn.close()

    out = {'t':[], 'x':[], 'y':[], 'z':[]}
    for ts, data in rows:
        msg = deserialize_message(data, Vector3Stamped)
        out['t'].append((ts - t0_ns) / 1e9)
        out['x'].append(msg.vector.x)
        out['y'].append(msg.vector.y)
        out['z'].append(msg.vector.z)

    for k in out: out[k] = np.array(out[k])
    if t_walk_end: mask = out['t'] < t_walk_end; [out.__setitem__(k, out[k][mask]) for k in out]
    return out
```

---

## Sub-analysis 3.1 — odom_mapping Position

Compare against VICON centroid and inner EKF.

```python
# Three-way comparison: VICON, /ekf, /odom_mapping
# Interpolate VICON to odom_mapping timestamps (lower rate than /ekf)
vicon_px_on_map = interp_to(t_traj, pos_vicon[:,0], odom['t'])
vicon_py_on_map = interp_to(t_traj, pos_vicon[:,1], odom['t'])

ekf_px_on_map = interp_to(ekf['t'], ekf['px'], odom['t'])
ekf_py_on_map = interp_to(ekf['t'], ekf['py'], odom['t'])

err_map_vs_vicon_3d = np.sqrt(
    (odom['px'] - vicon_px_on_map)**2 +
    (odom['py'] - vicon_py_on_map)**2
)
err_map_vs_ekf_3d = np.sqrt(
    (odom['px'] - ekf_px_on_map)**2 +
    (odom['py'] - ekf_py_on_map)**2
)
valid = ~np.isnan(err_map_vs_vicon_3d)

metrics_map_pos = {
    'RMSE_2D_vs_VICON': np.sqrt(np.mean(err_map_vs_vicon_3d[valid]**2)),
    'MAX_2D_vs_VICON':  np.max(err_map_vs_vicon_3d[valid]),
    'RMSE_2D_vs_EKF':   np.sqrt(np.mean(err_map_vs_ekf_3d[valid]**2)),
    'final_odom': (odom['px'][-1], odom['py'][-1]),
}
```

**Plots needed:**
- XY trajectory: VICON + `/ekf` + `/odom_mapping` in one plot
- X(t), Y(t) time series with shaded walking region

---

## Sub-analysis 3.2 — odom_mapping Yaw

```python
rpy_odom = quat_to_rpy(odom['qw'], odom['qx'], odom['qy'], odom['qz'])

vicon_yaw_on_map = interp_to(t_traj, np.degrees(rpy_vicon[:,2]), odom['t'])
ekf_yaw_on_map   = interp_to(ekf['t'], rpy_ekf[:,2], odom['t'])

valid = ~np.isnan(vicon_yaw_on_map)
metrics_map_yaw = {
    'RMSE_yaw_vs_VICON_deg': np.sqrt(np.mean((rpy_odom[valid,2] - vicon_yaw_on_map[valid])**2)),
    'RMSE_yaw_vs_EKF_deg':   np.sqrt(np.mean((rpy_odom[valid,2] - ekf_yaw_on_map[valid])**2)),
    'final_yaw_odom':  rpy_odom[-1, 2],
    'final_yaw_vicon': vicon_yaw_on_map[valid][-1],
}
```

---

## Sub-analysis 3.3 — Body Velocity (bv)

`/fusion/bv` is the body-frame velocity estimate from the outer fusion node.
Compare with:
1. VICON body-frame velocity (`v_body_vicon` from vicon-robot-analysis Step 5)
2. Inner EKF body-frame velocity (`/ekf` twist.linear)

```python
# Interpolate VICON body velocity to bv timestamps
vicon_vx_on_bv = interp_to(t_traj, v_body_vicon[:,0], bv['t'])
vicon_vy_on_bv = interp_to(t_traj, v_body_vicon[:,1], bv['t'])
vicon_vz_on_bv = interp_to(t_traj, v_body_vicon[:,2], bv['t'])

ekf_vx_on_bv = interp_to(ekf['t'], ekf['vx'], bv['t'])
ekf_vy_on_bv = interp_to(ekf['t'], ekf['vy'], bv['t'])

valid = ~np.isnan(vicon_vx_on_bv)
metrics_bv = {
    'RMSE_vx_vs_VICON': np.sqrt(np.nanmean((bv['x'] - vicon_vx_on_bv)**2)),
    'RMSE_vy_vs_VICON': np.sqrt(np.nanmean((bv['y'] - vicon_vy_on_bv)**2)),
    'RMSE_vx_vs_EKF':   np.sqrt(np.nanmean((bv['x'] - ekf_vx_on_bv)**2)),
    'RMSE_vy_vs_EKF':   np.sqrt(np.nanmean((bv['y'] - ekf_vy_on_bv)**2)),
}
```

**Plots needed:**
- vx(t), vy(t): fusion/bv vs VICON vs inner EKF (3 curves)
- Speed magnitude `|v_xy|` vs time — useful for verifying consistent forward velocity

---

## Fusion Quality Indicators

Beyond the metrics above, also check:

| Indicator | How to check |
|-----------|-------------|
| LiDAR update gaps | Plot `/odom_mapping` message intervals; gaps > 0.2 s suggest LiDAR dropout |
| Fusion weight shift | If `/ekf` and `/odom_mapping` diverge during walking, LiDAR weight may have dropped |
| Z-axis plausibility | `/odom_mapping` Z should stay near 0 during walking; jumps indicate LiDAR drift |
| Covariance growth | Track `odom.pose.covariance[0]` and `[7]` for XY uncertainty growth between LiDAR updates |

```python
# Message interval check
dt_map = np.diff(odom['t'])
print(f"odom_mapping: mean dt={dt_map.mean()*1000:.1f}ms, max dt={dt_map.max()*1000:.1f}ms, "
      f"gaps>200ms: {np.sum(dt_map > 0.2)}")
```
