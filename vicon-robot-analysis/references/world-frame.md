# Markers & World Frame Reference

## Marker Naming Convention

Namespace format: `session_label:MarkerName`

Examples: `corgi_0507:O1`, `20260507_ground:Ground1`

### Robot Body Markers (`corgi_MMDD` namespace)

| Marker | Role | Leg |
|--------|------|-----|
| `O1` | Hip point | Left Front (LF) |
| `O2` | Hip point | Right Front (RF) |
| `O3` | Hip point | Right Hind (RH) |
| `O4` | Hip point | Left Hind (LH) |
| `Tigger` | **Trigger / sync point** — see Time Sync | — |

> **⚠️ Known typo**: The trigger marker is named **`Tigger`** (not "Trigger") in VICON.
> This is an established naming convention. Always search for `"Tigger"` in code.

The four hip markers **O1–O4 form a rigid body plane** on the robot chassis.

### Foot Markers (`corgi_MMDD` namespace)

| Marker | Leg |
|--------|-----|
| `G1` | Left Front (LF) |
| `G2` | Right Front (RF) |
| `G3` | Right Hind (RH) |
| `G4` | Left Hind (LH) |

### Ground Reference Markers (`YYYYMMDD_ground` namespace)

| Marker | Notes |
|--------|-------|
| `Ground1` | Static ground plane points |
| `Ground2` | |
| `Ground3` | |
| `Ground4` | Optional; at least **3 points required** |

Ground markers are **static** — use their position from the **first valid (non-NaN) frame**.

---

## Ground Plane → World Frame

### Step 1 — Fit Ground Plane via SVD

```python
import numpy as np

def fit_plane_svd(points: np.ndarray) -> tuple[np.ndarray, np.ndarray]:
    """
    Fit a plane to N×3 points using SVD.
    Returns (centroid, normal) where normal points upward (+Z preference).
    """
    centroid = points.mean(axis=0)
    _, _, Vt = np.linalg.svd(points - centroid)
    normal = Vt[-1]  # smallest singular value → plane normal
    if normal[2] < 0:
        normal = -normal  # ensure normal points upward
    return centroid, normal

# Collect first valid frame positions for Ground markers
ground_pts = []
for marker in ["Ground1", "Ground2", "Ground3", "Ground4"]:
    col = get_marker_col(marker)
    valid = traj_df[col].dropna().iloc[0].values  # first non-NaN row
    ground_pts.append(valid)
ground_pts = np.array(ground_pts)   # shape (3 or 4, 3)

centroid_ground, normal_ground = fit_plane_svd(ground_pts)
```

### Step 2 — Build World Frame Rotation Matrix

```python
def rotation_to_align_z(n: np.ndarray) -> np.ndarray:
    """Return 3×3 rotation matrix R such that R @ n ≈ [0, 0, 1]."""
    n = n / np.linalg.norm(n)
    z = np.array([0.0, 0.0, 1.0])
    v = np.cross(n, z)
    s = np.linalg.norm(v)
    c = np.dot(n, z)
    if s < 1e-10:
        return np.eye(3)
    Vx = np.array([[0, -v[2], v[1]], [v[2], 0, -v[0]], [-v[1], v[0], 0]])
    R = np.eye(3) + Vx + Vx @ Vx * ((1 - c) / s**2)
    return R

R_ground = rotation_to_align_z(normal_ground)
```

### Step 3 — Transform All Markers to World Frame

```python
def to_world(p_vicon: np.ndarray) -> np.ndarray:
    """p_vicon: (N, 3) array in VICON frame → (N, 3) in world frame."""
    return (R_ground @ (p_vicon - centroid_ground).T).T
```

After transformation, the ground plane lies at **Z = 0**.  
However, the horizontal X/Y axes are still in VICON lab orientation — not aligned with the robot.  
**Proceed to Step 4 to define the robot-centric world frame.**

---

## Robot-Centric World Frame（以機器人初始狀態為原點）

> **原則**：機器人初始位置 = 世界原點 (0,0,0)，機器人初始前進方向 = 世界 +X 軸。  
> 機器人雖然大致往正 X 走，但 VICON lab 座標不一定與機器人初始 heading 平行，  
> 必須額外做一次水平旋轉。

### Step 4 — 求機器人初始朝向

機器人 body X 軸定義為 **LH(O4) → LF(O1)** 方向，投影到 XY 平面（地面平行面）：

```python
def get_robot_initial_heading(traj_df, t0_frame: int = 0) -> np.ndarray:
    """
    Compute the robot's initial forward direction in the ground-plane world frame.
    Returns a unit vector in the XY plane (Z component is 0).
    t0_frame: the first valid frame index (typically 0 or first non-NaN frame).
    """
    # Get O1 (LF hip) and O4 (LH hip) at the initial frame in world frame
    o1_world = to_world(traj_df[col_O1].iloc[t0_frame].values.reshape(1, 3))[0]
    o4_world = to_world(traj_df[col_O4].iloc[t0_frame].values.reshape(1, 3))[0]

    heading = o1_world - o4_world          # forward direction vector (LH → LF)
    heading[2] = 0.0                        # project to ground plane
    heading = heading / np.linalg.norm(heading)
    return heading                          # shape (3,) unit vector in XY plane
```

### Step 5 — 建立水平旋轉矩陣（VICON frame → robot-centric frame）

```python
def rotation_align_heading(heading: np.ndarray) -> np.ndarray:
    """
    Return a 3×3 rotation matrix R_h that rotates the world frame so that
    `heading` aligns with [1, 0, 0] (world +X = robot forward).
    Only rotates around Z axis (yaw).
    """
    angle = np.arctan2(heading[1], heading[0])  # yaw of robot in current world frame
    c, s = np.cos(-angle), np.sin(-angle)        # rotate by -angle to align to +X
    R_h = np.array([
        [c, -s, 0.0],
        [s,  c, 0.0],
        [0.0, 0.0, 1.0]
    ])
    return R_h

heading_0 = get_robot_initial_heading(traj_df, t0_frame=first_valid_frame)
R_heading  = rotation_align_heading(heading_0)
```

### Step 6 — 求初始位置偏移（以機器人質心為原點）

```python
# Robot centroid at t=0 in world frame (after ground plane only)
o1_o2_o3_o4_world_t0 = np.array([
    to_world(traj_df[col_O1].iloc[first_valid_frame].values.reshape(1, 3))[0],
    to_world(traj_df[col_O2].iloc[first_valid_frame].values.reshape(1, 3))[0],
    to_world(traj_df[col_O3].iloc[first_valid_frame].values.reshape(1, 3))[0],
    to_world(traj_df[col_O4].iloc[first_valid_frame].values.reshape(1, 3))[0],
])
centroid_robot_t0 = o1_o2_o3_o4_world_t0.mean(axis=0)  # shape (3,)
```

### Step 7 — 完整的 VICON → Robot World 轉換

```python
def to_robot_world(p_vicon: np.ndarray) -> np.ndarray:
    """
    Full transform: VICON lab frame → robot-centric world frame.
    1. Ground plane alignment (Z-up)
    2. Subtract robot's initial centroid position
    3. Yaw rotation so robot initial heading = +X

    p_vicon: (N, 3) in VICON mm units
    returns: (N, 3) in mm, robot-centric
    """
    p_ground = to_world(p_vicon)                         # Step 3: Z-up
    p_shifted = p_ground - centroid_robot_t0             # Step 6: set origin
    p_robot  = (R_heading @ p_shifted.T).T               # Step 5: align heading
    return p_robot
```

**以 `to_robot_world()` 取代所有原本的 `to_world()` 呼叫**（Step 5 動力學計算、比較圖、速度計算等）。

### 驗證

```python
# Sanity check: robot centroid at t=0 should be (0, 0, 0)
centroid_t0_check = to_robot_world(
    o1_o2_o3_o4_world_t0_vicon  # 原始 VICON 座標
).mean(axis=0)
assert np.allclose(centroid_t0_check[:2], 0, atol=1e-6), f"Origin not aligned: {centroid_t0_check}"

# Robot heading at t=0 should point in +X direction
o1_rw = to_robot_world(traj_df[col_O1].iloc[first_valid_frame].values.reshape(1, 3))[0]
o4_rw = to_robot_world(traj_df[col_O4].iloc[first_valid_frame].values.reshape(1, 3))[0]
heading_check = o1_rw - o4_rw
yaw_check_deg = np.degrees(np.arctan2(heading_check[1], heading_check[0]))
print(f"Initial heading yaw in robot world frame: {yaw_check_deg:.2f}°")  # should be ~0°
```

### 注意事項

- Z 原點不是地面，而是機器人初始**質心高度**（約為髖關節高度）。若要以地面為 Z=0，需另外減去 `centroid_robot_t0[2]`，但通常分析只看 XY 與 ΔZ，不影響結果。
- 若機器人走路時有偏轉（yaw drift），軌跡仍會偏離 X 軸，這是正常現象（也是評估的對象）。
- EKF / odom_mapping 已以 odom 框架（機器人初始位置為原點）發布，可直接與 `to_robot_world()` 的輸出比較。
