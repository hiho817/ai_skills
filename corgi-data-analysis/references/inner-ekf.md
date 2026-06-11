# Inner EKF Analysis — Reference

## Topics

| Topic | Type | Rate | Content |
|-------|------|------|---------|
| `/ekf` | `nav_msgs/msg/Odometry` | ~400–900 Hz | position, velocity, covariance |
| `/ekf/orientation` | `geometry_msgs/msg/Quaternion` | ~400–900 Hz | attitude quaternion (same rate as `/ekf`) |
| `/ekf/ba` | `geometry_msgs/msg/Vector3` | ~400–900 Hz | accelerometer bias [m/s²] |
| `/ekf/bw` | `geometry_msgs/msg/Vector3` | ~400–900 Hz | gyroscope bias [rad/s] |

All topics share the same timestamp cadence and can be loaded together.

---

## Loading Helpers

```python
import sqlite3, numpy as np
from rclpy.serialization import deserialize_message
from nav_msgs.msg import Odometry
from geometry_msgs.msg import Quaternion, Vector3

def load_ekf(db_path, t0_ns=None, t_walk_end=None):
    """Load all inner EKF topics from a bag file."""
    conn = sqlite3.connect(db_path)
    cur  = conn.cursor()
    cur.execute("SELECT name, id FROM topics")
    tmap = {r[0]: r[1] for r in cur.fetchall()}

    def fetch(topic):
        cur.execute(f"SELECT timestamp, data FROM messages WHERE topic_id={tmap[topic]} ORDER BY timestamp")
        return cur.fetchall()

    rows_odom = fetch('/ekf')
    rows_ori  = fetch('/ekf/orientation')
    rows_ba   = fetch('/ekf/ba')
    rows_bw   = fetch('/ekf/bw')
    conn.close()

    if t0_ns is None:
        t0_ns = rows_odom[0][0]

    def t(ts): return (ts - t0_ns) / 1e9

    # --- Odometry (position + velocity) ---
    out = {k: [] for k in ['t','px','py','pz','vx','vy','vz',
                            'qw','qx','qy','qz',
                            'cov_px','cov_py','cov_pz',
                            'cov_vx','cov_vy','cov_vz']}
    for ts, data in rows_odom:
        msg = deserialize_message(data, Odometry)
        out['t'].append(t(ts))
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
        c = msg.pose.covariance  # 36-element row-major
        out['cov_px'].append(c[0]); out['cov_py'].append(c[7]); out['cov_pz'].append(c[14])
        c2 = msg.twist.covariance
        out['cov_vx'].append(c2[0]); out['cov_vy'].append(c2[7]); out['cov_vz'].append(c2[14])

    # --- Bias terms ---
    ba = {'t':[], 'x':[], 'y':[], 'z':[]}
    for ts, data in rows_ba:
        msg = deserialize_message(data, Vector3)
        ba['t'].append(t(ts)); ba['x'].append(msg.x); ba['y'].append(msg.y); ba['z'].append(msg.z)

    bw = {'t':[], 'x':[], 'y':[], 'z':[]}
    for ts, data in rows_bw:
        msg = deserialize_message(data, Vector3)
        bw['t'].append(t(ts)); bw['x'].append(msg.x); bw['y'].append(msg.y); bw['z'].append(msg.z)

    for d in [out, ba, bw]:
        for k in d:
            d[k] = np.array(d[k])

    if t_walk_end is not None:
        mask = out['t'] < t_walk_end
        for k in out:
            out[k] = out[k][mask]
        for d in [ba, bw]:
            m = d['t'] < t_walk_end
            for k in d:
                d[k] = d[k][m]

    return out, ba, bw
```

---

## Sub-analysis 2.1 — Position

Compare `/ekf` position with VICON centroid (O1–O4 in world frame).

```python
from scipy.interpolate import interp1d

def interp_to(src_t, src_val, tgt_t):
    f = interp1d(src_t, src_val, bounds_error=False, fill_value=np.nan)
    return f(tgt_t)

# Interpolate VICON centroid (500 Hz) to EKF timestamps
vicon_px_on_ekf = interp_to(t_traj, pos_vicon[:,0], ekf['t'])
vicon_py_on_ekf = interp_to(t_traj, pos_vicon[:,1], ekf['t'])
vicon_pz_on_ekf = interp_to(t_traj, pos_vicon[:,2], ekf['t'])

err_x = ekf['px'] - vicon_px_on_ekf
err_y = ekf['py'] - vicon_py_on_ekf
err_z = ekf['pz'] - vicon_pz_on_ekf
err_3d = np.sqrt(err_x**2 + err_y**2 + err_z**2)
valid  = ~np.isnan(err_3d)

metrics_pos = {
    'RMSE_X': np.sqrt(np.mean(err_x[valid]**2)),
    'RMSE_Y': np.sqrt(np.mean(err_y[valid]**2)),
    'RMSE_Z': np.sqrt(np.mean(err_z[valid]**2)),
    'RMSE_3D': np.sqrt(np.mean(err_3d[valid]**2)),
    'MAX_3D':  np.max(err_3d[valid]),
    'final_EKF':   (ekf['px'][-1], ekf['py'][-1]),
    'final_VICON': (pos_vicon[np.where(valid_hip)[0][-1], 0],
                    pos_vicon[np.where(valid_hip)[0][-1], 1]),
}
```

**Plots needed:**
- XY trajectory overlay (EKF vs VICON)
- X(t), Y(t), Z(t) time series
- 3D error vs time with shaded 3σ band from covariance

---

## Sub-analysis 2.2 — Velocity

Compare `/ekf` twist.linear with VICON Savitzky-Golay velocity.

```python
# Velocity from EKF is in base_link frame; VICON v_body is also in body frame
# Use v_body_vicon from vicon-robot-analysis skill Step 5

vicon_vx_on_ekf = interp_to(t_traj, v_body_vicon[:,0], ekf['t'])
vicon_vy_on_ekf = interp_to(t_traj, v_body_vicon[:,1], ekf['t'])
vicon_vz_on_ekf = interp_to(t_traj, v_body_vicon[:,2], ekf['t'])

metrics_vel = {
    'RMSE_vx': np.sqrt(np.nanmean((ekf['vx'] - vicon_vx_on_ekf)**2)),
    'RMSE_vy': np.sqrt(np.nanmean((ekf['vy'] - vicon_vy_on_ekf)**2)),
    'RMSE_vz': np.sqrt(np.nanmean((ekf['vz'] - vicon_vz_on_ekf)**2)),
}
```

**Plots needed:**
- vx(t), vy(t) overlay (EKF vs VICON) — forward and lateral velocity
- Speed magnitude `|v|` vs time

---

## Sub-analysis 2.3 — Attitude (RPY)

```python
from scipy.spatial.transform import Rotation

def quat_to_rpy(qw, qx, qy, qz):
    """ZYX Euler → [roll, pitch, yaw] arrays in degrees."""
    R = Rotation.from_quat(np.column_stack([qx, qy, qz, qw]))  # scipy: [x,y,z,w]
    return R.as_euler('ZYX')[:, ::-1]   # → [roll, pitch, yaw]

rpy_ekf = quat_to_rpy(ekf['qw'], ekf['qx'], ekf['qy'], ekf['qz'])  # shape (N,3)

# VICON RPY already computed in rpy_vicon (vicon-robot-analysis Step 5)
# rpy_vicon[:, 0] = roll, [:, 1] = pitch, [:, 2] = yaw

metrics_att = {}
for i, name in enumerate(['roll','pitch','yaw']):
    vicon_rpy_on_ekf = interp_to(t_traj, np.degrees(rpy_vicon[:,i]), ekf['t'])
    err = rpy_ekf[:,i] - vicon_rpy_on_ekf
    metrics_att[f'RMSE_{name}_deg'] = np.sqrt(np.nanmean(err**2))
```

**Plots needed:**
- Roll(t), Pitch(t), Yaw(t) overlay (EKF vs VICON)
- Focus on yaw drift over the experiment

---

## Sub-analysis 2.4 & 2.5 — Bias Convergence

```python
fig, axes = plt.subplots(2, 3, figsize=(14, 6), sharex=True)
labels = ['x', 'y', 'z']
units  = {'ba': 'm/s²', 'bw': 'rad/s'}

for row, (bias_data, bname, unit) in enumerate([(ba, 'ba', 'm/s²'), (bw, 'bw', 'rad/s')]):
    for col, (ax, lbl) in enumerate(zip(axes[row], labels)):
        ax.plot(bias_data['t'], bias_data[lbl], lw=0.8)
        ax.axhline(0, color='gray', ls='--', lw=0.5)
        ax.set_ylabel(f'/ekf/{bname}.{lbl} [{unit}]')
        ax.set_title(f'{"Accel" if bname=="ba" else "Gyro"} bias {lbl}')
        ax.grid(True, alpha=0.4)
        ax.set_xlabel('Time [s]')

fig.suptitle('Inner EKF Bias Convergence')
fig.tight_layout()
```

**Metrics to report:**
- Initial value (t ≈ 0 s)
- Steady-state value (mean over last 5 s of walking)
- Standard deviation over walking phase (proxy for convergence quality)

```python
def bias_stats(bias_data, key, t_steady_start):
    mask = bias_data['t'] > t_steady_start
    vals = bias_data[key][mask]
    return {'mean': np.mean(vals), 'std': np.std(vals),
            'init': bias_data[key][0]}
```
