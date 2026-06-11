# Body Kinematics Reference

## Prerequisites

- `traj_df`, `get_marker_col`, `to_world` are set up (see CSV Parsing and World Frame refs).
- `scipy.spatial.transform.Rotation` and `scipy.signal.savgol_filter` are imported.

---

## 1. Hip Points in World Frame

```python
O1 = to_world(traj_df[get_marker_col("O1")].values)   # (N, 3)
O2 = to_world(traj_df[get_marker_col("O2")].values)
O3 = to_world(traj_df[get_marker_col("O3")].values)
O4 = to_world(traj_df[get_marker_col("O4")].values)
```

---

## 2. Body Centroid

```python
import numpy as np

valid = ~(np.isnan(O1).any(axis=1) | np.isnan(O2).any(axis=1) |
          np.isnan(O3).any(axis=1) | np.isnan(O4).any(axis=1))

centroid_body = np.full((len(traj_df), 3), np.nan)
centroid_body[valid] = (O1[valid] + O2[valid] + O3[valid] + O4[valid]) / 4.0

# Zero-reference XY at the first valid frame; Z is already ground-relative
first_valid = np.where(valid)[0][0]
origin_xy = centroid_body[first_valid, :2].copy()
pos = centroid_body.copy()
pos[:, :2] -= origin_xy
```

---

## 3. Body Frame Rotation (per frame)

### Axis convention (right-hand, Z-up)

| Axis | Definition |
|------|-----------|
| $Z_b$ | Normal to O1–O4 plane (pointing upward) |
| $X_b$ | O4(LH) → O1(LF), projected onto body plane (forward) |
| $Y_b$ | $Z_b \times X_b$ (leftward) |

```python
from scipy.spatial.transform import Rotation

def body_rotation(o1, o2, o3, o4):
    """
    Compute per-frame body rotation matrix (3×3) given hip positions.
    All inputs shape (N, 3). Returns R_body: (N, 3, 3), columns = [Xb, Yb, Zb].
    """
    N = o1.shape[0]
    R = np.full((N, 3, 3), np.nan)

    pts = np.stack([o1, o2, o3, o4], axis=1)       # (N, 4, 3)
    centroid = pts.mean(axis=1, keepdims=True)       # (N, 1, 3)
    centered = pts - centroid                        # (N, 4, 3)

    for i in range(N):
        if np.isnan(centered[i]).any():
            continue
        _, _, Vt = np.linalg.svd(centered[i])
        Zb = Vt[-1]
        if Zb[2] < 0:
            Zb = -Zb

        x_raw = o1[i] - o4[i]                       # LH → LF
        x_raw -= np.dot(x_raw, Zb) * Zb             # project onto body plane
        Xb = x_raw / np.linalg.norm(x_raw)

        Yb = np.cross(Zb, Xb)
        Yb /= np.linalg.norm(Yb)

        R[i] = np.column_stack([Xb, Yb, Zb])
    return R

R_body = body_rotation(O1, O2, O3, O4)   # (N, 3, 3)
```

---

## 4. Roll Pitch Yaw

ZYX Euler convention (extrinsic x-y-z):

```python
rpy = np.full((len(traj_df), 3), np.nan)
for i in range(len(traj_df)):
    if not np.isnan(R_body[i]).any():
        rpy[i] = Rotation.from_matrix(R_body[i]).as_euler("ZYX", degrees=False)[::-1]
        # rpy[i] = [roll, pitch, yaw]
```

---

## 5. Velocity — Savitzky-Golay Derivative

**Recommended parameters at 500 Hz:**
- `window_length = 11` (22 ms window), `polyorder = 3`
- For faster dynamics (running): `window_length = 7`

```python
from scipy.signal import savgol_filter

def sg_velocity(pos: np.ndarray, fs: float = 500.0,
                window: int = 11, poly: int = 3) -> np.ndarray:
    """
    Compute velocity from position (N, 3) using Savitzky-Golay derivative.
    Handles NaN by operating on contiguous valid segments.
    Returns velocity in same unit / second.
    """
    vel = np.full_like(pos, np.nan)
    valid = ~np.isnan(pos).any(axis=1)
    changes = np.diff(valid.astype(int), prepend=0, append=0)
    starts = np.where(changes == 1)[0]
    ends = np.where(changes == -1)[0]
    for s, e in zip(starts, ends):
        seg = pos[s:e]
        if len(seg) >= window:
            for ax in range(3):
                vel[s:e, ax] = savgol_filter(
                    seg[:, ax], window_length=window, polyorder=poly,
                    deriv=1, delta=1.0 / fs
                )
    return vel

# World-frame velocity (mm/s)
v_world = sg_velocity(pos, fs=500.0)

# Body-frame velocity: v_body = R_body^T @ v_world
v_body = np.full_like(v_world, np.nan)
valid = ~np.isnan(R_body).any(axis=(1, 2)) & ~np.isnan(v_world).any(axis=1)
v_body[valid] = np.einsum("nij,nj->ni", R_body[valid].transpose(0, 2, 1), v_world[valid])
```
