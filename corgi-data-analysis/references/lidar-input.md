# LiDAR Input Quality — Reference

## Topic

| Topic | Type | Rate | Frame |
|-------|------|------|-------|
| `/lidar_odom` | `nav_msgs/msg/Odometry` | ~10 Hz (scan rate) | `camera_init → base_link` |

**Critical**: `/lidar_odom` is in the `camera_init` frame, **not** `odom`.
Always apply `T_{odom←camera_init}` before any comparison.

---

## Getting the Transform `T_{odom←camera_init}`

The transform is logged by the fusion node at startup. Example log line:

```
[corgi_fusion_node] T_{odom←camera_init} set: t=[0.092 -0.021 0.178] q=[0.6988 0.1261 0.1264 0.6927]
```

Parse it manually or record it during the experiment. Store as:

```python
T_CO_t = np.array([tx, ty, tz])            # translation (metres)
T_CO_q = np.array([qw, qx, qy, qz])        # rotation quaternion
```

---

## Loading and Transforming

```python
def load_lidar_odom(db_path, t0_ns, T_CO_t, T_CO_q, t_walk_end=None):
    conn = sqlite3.connect(db_path)
    cur  = conn.cursor()
    cur.execute("SELECT id FROM topics WHERE name='/lidar_odom'")
    tid  = cur.fetchone()[0]
    cur.execute(f"SELECT timestamp, data FROM messages WHERE topic_id={tid} ORDER BY timestamp")
    rows = cur.fetchall(); conn.close()

    out = {k: [] for k in ['t','px','py','pz','qw','qx','qy','qz']}
    for ts, data in rows:
        msg = deserialize_message(data, Odometry)
        out['t'].append((ts - t0_ns) / 1e9)
        out['px'].append(msg.pose.pose.position.x)
        out['py'].append(msg.pose.pose.position.y)
        out['pz'].append(msg.pose.pose.position.z)
        out['qw'].append(msg.pose.pose.orientation.w)
        out['qx'].append(msg.pose.pose.orientation.x)
        out['qy'].append(msg.pose.pose.orientation.y)
        out['qz'].append(msg.pose.pose.orientation.z)

    for k in out: out[k] = np.array(out[k])

    # Apply T_{odom←camera_init}
    R_co = quat_to_rotmat(*T_CO_q)   # see inner-ekf.md for helper
    p    = np.column_stack([out['px'], out['py'], out['pz']])
    p_odom = (R_co @ p.T).T + T_CO_t
    out['px_odom'] = p_odom[:, 0]
    out['py_odom'] = p_odom[:, 1]
    out['pz_odom'] = p_odom[:, 2]

    if t_walk_end: mask = out['t'] < t_walk_end; [out.__setitem__(k, out[k][mask]) for k in out]
    return out
```

---

## Quality Checks

### Message Rate

```python
dt_lidar = np.diff(lidar['t'])
print(f"LiDAR odom: count={len(lidar['t'])}, "
      f"mean dt={dt_lidar.mean()*1000:.1f} ms, "
      f"expected ~100 ms (10 Hz), "
      f"gaps >500 ms: {np.sum(dt_lidar > 0.5)}")
```

Gaps > 500 ms indicate scan matching failure or LiDAR dropout.

### Trajectory Jumps

```python
# Detect position jumps (inter-frame displacement > threshold)
dp = np.sqrt(np.diff(lidar['px_odom'])**2 + np.diff(lidar['py_odom'])**2)
jumps = np.where(dp > 0.05)[0]   # 5 cm threshold
print(f"Position jumps > 5 cm: {len(jumps)}")
if len(jumps): print(f"  at t = {lidar['t'][jumps]}")
```

### Phase Validity

- **Valid phase**: `t_walk_start ≤ t ≤ t_walk_end` (robot on ground, clear LiDAR FOV)
- **Invalid phase**: `t > t_walk_end` (robot lifted, LiDAR occluded → large drift expected)

```python
walk_mask = lidar['t'] < t_walk_end
# Only use walk_mask data for quantitative metrics
```

### Degeneracy Score (if available)

If the fusion node logs a degeneracy score (e.g., from FAST-LIO2 `eigenvalue_ratio`),
plot it over time. Values near 0 indicate degenerate environments (flat corridors, etc.).

---

## Plots

### Plot A — LiDAR XY Trajectory in odom Frame

```python
fig, ax = plt.subplots(figsize=(8, 6))
ax.plot(lidar['px_odom'][walk_mask], lidar['py_odom'][walk_mask],
        'g-.', lw=1.2, label='/lidar_odom (odom frame, walking)')
ax.plot(ekf['px'], ekf['py'], 'b-', lw=1.0, label='/ekf')
ax.plot(pos_vicon[valid_hip, 0], pos_vicon[valid_hip, 1], 'k--', lw=1.5, label='VICON')
ax.set_xlabel('X [m]'); ax.set_ylabel('Y [m]')
ax.set_title('LiDAR Odom vs EKF vs VICON — odom frame')
ax.legend(); ax.axis('equal'); ax.grid(True, alpha=0.4)
```

### Plot B — LiDAR Odom Z (altitude consistency)

```python
fig, ax = plt.subplots(figsize=(10, 3))
ax.plot(lidar['t'], lidar['pz_odom'], label='/lidar_odom Z (odom)')
ax.plot(ekf['t'], ekf['pz'], label='/ekf Z')
ax.axvline(t_walk_end, color='red', ls='--', lw=1, label='robot lifted')
ax.set_ylabel('Z [m]'); ax.set_xlabel('Time [s]')
ax.set_title('Altitude — LiDAR vs EKF')
ax.legend(); ax.grid(True, alpha=0.4)
```

### Plot C — Message Interval

```python
fig, ax = plt.subplots(figsize=(10, 3))
ax.plot(lidar['t'][1:], dt_lidar * 1000, lw=0.8)
ax.axhline(100, color='orange', ls='--', lw=1, label='100 ms (10 Hz)')
ax.axhline(500, color='red',    ls='--', lw=1, label='500 ms dropout')
ax.set_ylabel('Interval [ms]'); ax.set_xlabel('Time [s]')
ax.set_title('/lidar_odom Message Interval')
ax.legend(); ax.grid(True, alpha=0.4)
```

---

## Summary Metrics

| Metric | Typical good value |
|--------|--------------------|
| Mean message interval | 90–110 ms |
| Gaps > 500 ms | 0 |
| Position jumps > 5 cm | 0 |
| XY RMSE vs VICON (walk phase) | < 3 cm |
| Z drift (walk phase) | < 2 cm |
