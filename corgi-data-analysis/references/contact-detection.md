# Contact Detection — Reference

## Data Sources

| Source | Signal | Frame |
|--------|--------|-------|
| VICON CSV | G1 (LF), G2 (RF), G3 (RH), G4 (LH) foot marker Z | world (mm) |
| ROS2 bag | `/gmo/contact_state` (`corgi_msgs/msg/GMOContactStateStamped`) | — |

Foot marker order matches hip marker order:
- G1 = LF (Left Front)
- G2 = RF (Right Front)
- G3 = RH (Right Hind)
- G4 = LH (Left Hind)

---

## Algorithm: VICON Ground-Truth Contact

### 1. Get World-Frame Foot Heights

After building `R_ground` from Ground1–Ground4 (see vicon-robot-analysis skill):

```python
def get_foot_height_m(marker_name, R_ground, centroid_ground):
    """Return Z-height in world frame (metres) for each frame."""
    xyz = get_xyz(marker_name)          # (N,3) mm, VICON frame
    xyz_world = to_world(xyz)           # R_ground applied, centroid subtracted
    return xyz_world[:, 2] / 1000.0    # mm → m
```

### 2. Threshold-Based Contact

```python
CONTACT_THRESHOLD_M = 0.025   # 25 mm — tune per experiment surface

for leg, marker in [('LF','G1'),('RF','G2'),('RH','G3'),('LH','G4')]:
    height = get_foot_height_m(marker, R_ground, centroid_ground)
    contact_vicon[leg] = height < CONTACT_THRESHOLD_M   # True = in contact
    # NaN frames → unknown (np.nan; mask separately)
```

> **Threshold tuning**: plot the raw height time-series first. The foot Z during
> stance is typically 0–10 mm; swing peaks at 30–60 mm. Choose threshold in the
> trough between the two modes.

### 3. Contact Events

```python
def contact_events(contact_bool, t_axis):
    """Return list of (t_start, t_end) for each stance phase."""
    edges = np.diff(contact_bool.astype(int), prepend=0, append=0)
    starts = t_axis[np.where(edges == 1)[0]]
    ends   = t_axis[np.where(edges == -1)[0]]
    return list(zip(starts, ends))
```

### 4. Fill Short Gaps (Optional)

```python
def fill_gaps(contact_bool, max_gap_frames=5):
    """Bridge swing phases shorter than max_gap_frames (VICON NaN artifacts)."""
    filled = contact_bool.copy()
    in_gap = False; gap_start = 0
    for i, c in enumerate(contact_bool):
        if np.isnan(c): continue
        if not c and not in_gap:
            in_gap = True; gap_start = i
        elif c and in_gap:
            if i - gap_start <= max_gap_frames:
                filled[gap_start:i] = True
            in_gap = False
    return filled
```

---

## Algorithm: GMO Contact from Bag

```python
from corgi_msgs.msg import GMOContactStateStamped

def load_gmo_contact(db_path, t0_ns):
    conn = sqlite3.connect(db_path)
    cur  = conn.cursor()
    cur.execute("SELECT id FROM topics WHERE name='/gmo/contact_state'")
    tid  = cur.fetchone()[0]
    cur.execute(f"SELECT timestamp, data FROM messages WHERE topic_id={tid} ORDER BY timestamp")
    rows = conn.fetchall(); conn.close()

    t, lf, rf, rh, lh = [], [], [], [], []
    for ts_ns, data in rows:
        msg = deserialize_message(data, GMOContactStateStamped)
        t.append((ts_ns - t0_ns) / 1e9)
        lf.append(msg.lf_contact)
        rf.append(msg.rf_contact)
        rh.append(msg.rh_contact)
        lh.append(msg.lh_contact)
    return {'t': np.array(t), 'LF': np.array(lf), 'RF': np.array(rf),
            'RH': np.array(rh), 'LH': np.array(lh)}
```

> Verify field names in `GMOContactStateStamped` with `ros2 interface show corgi_msgs/msg/GMOContactStateStamped`.

---

## Plots

### Plot A — Contact Timeline (4-leg raster)

```python
fig, axes = plt.subplots(4, 1, figsize=(14, 6), sharex=True)
legs = ['LF', 'RF', 'RH', 'LH']
colors = {'VICON': '#2196F3', 'GMO': '#FF5722'}

for ax, leg in zip(axes, legs):
    # VICON ground truth — filled bar
    for t_s, t_e in contact_events(contact_vicon[leg], t_traj):
        ax.axvspan(t_s, t_e, color=colors['VICON'], alpha=0.4, label='VICON' if t_s==contact_events(contact_vicon[leg], t_traj)[0][0] else '')
    # GMO estimated — hatched bar
    for t_s, t_e in contact_events(gmo[leg].astype(bool), gmo['t']):
        ax.axvspan(t_s, t_e, color=colors['GMO'], alpha=0.3, hatch='//', label='GMO' if t_s==contact_events(gmo[leg].astype(bool), gmo['t'])[0][0] else '')
    ax.set_ylabel(leg, rotation=0, labelpad=20)
    ax.set_ylim(-0.1, 1.1); ax.set_yticks([])
    ax.axvline(t_walk_end, color='red', ls='--', lw=1)

axes[-1].set_xlabel('Time [s]')
axes[0].set_title('Contact Timeline: VICON (blue) vs GMO estimate (orange)')
```

### Plot B — Foot Height with Contact Threshold

```python
fig, axes = plt.subplots(4, 1, figsize=(14, 8), sharex=True)
for ax, leg, marker in zip(axes, ['LF','RF','RH','LH'], ['G1','G2','G3','G4']):
    h = get_foot_height_m(marker, R_ground, centroid_ground)
    ax.plot(t_traj, h * 1000, lw=0.8, label=f'{leg} height (mm)')
    ax.axhline(CONTACT_THRESHOLD_M * 1000, color='red', ls='--', lw=1, label='threshold')
    ax.set_ylabel(f'{leg} Z [mm]')
    ax.legend(loc='upper right', fontsize=8)
axes[-1].set_xlabel('Time [s]')
fig.suptitle('VICON Foot Height vs Contact Threshold')
```

---

## Metrics

```python
def contact_metrics(vicon_bool, gmo_bool, t_common):
    """Resample both to common time grid, compute binary classification stats."""
    from scipy.interpolate import interp1d
    v = np.interp(t_common, t_traj, vicon_bool.astype(float)) > 0.5
    g = np.interp(t_common, gmo['t'], gmo_bool.astype(float)) > 0.5
    tp = np.sum(v & g);   tn = np.sum(~v & ~g)
    fp = np.sum(~v & g);  fn = np.sum(v & ~g)
    precision = tp / (tp + fp + 1e-9)
    recall    = tp / (tp + fn + 1e-9)
    # Latency: mean time shift of rising edges
    v_edges = np.diff(v.astype(int), prepend=0); g_edges = np.diff(g.astype(int), prepend=0)
    v_on = t_common[v_edges == 1]; g_on = t_common[g_edges == 1]
    latencies = []
    for vt in v_on:
        diffs = g_on - vt
        near = diffs[np.abs(diffs) < 0.1]
        if len(near): latencies.append(near[np.argmin(np.abs(near))])
    return {'precision': precision, 'recall': recall,
            'mean_latency_ms': np.mean(latencies)*1000 if latencies else np.nan}
```

### Report table

| Leg | Precision | Recall | Mean Latency (ms) |
|-----|-----------|--------|-------------------|
| LF  | …         | …      | …                 |
| RF  | …         | …      | …                 |
| RH  | …         | …      | …                 |
| LH  | …         | …      | …                 |
