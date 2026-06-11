# CSV Parsing Reference

## File Layout

A VICON export contains **two consecutive sections** in a single CSV file.

### Devices Section (Force Plates)

```
Row 0:  "Devices"                           ← section header
Row 1:  1000                                ← sampling frequency (Hz)
Row 2:  device names  (ForcePlate1, ...)    ← device labels
Row 3:  column names  Frame  Sub Frame  Fx  Fy  Fz  Mx  My  Mz  Cx  Cy  Cz  ...
Row 4:  units         N  N  N  N.mm  N.mm  N.mm  mm  mm  mm  ...
Row 5+: data rows
```

### Trajectories Section (Marker Positions)

Locate by scanning for the line that contains only `"Trajectories"`.

```
Row N+0: "Trajectories"
Row N+1: 500                                ← sampling frequency (Hz)
Row N+2: marker names  namespace:MarkerName  (one name spans 3 cols X Y Z)
Row N+3: column names  Frame  Sub Frame  X  Y  Z  X  Y  Z  ...
Row N+4: units         mm  mm  mm  ...
Row N+5+: data rows
```

- Each marker occupies **3 columns (X, Y, Z)** in millimetres.
- `Frame` counts at 500 Hz (starts from 1).
- **Missing / occluded markers** → empty cells → `NaN` in pandas.

---

## Parsing Recipe

```python
import pandas as pd

def find_section_row(filepath: str, section_name: str) -> int:
    """Return 0-based row index of the section header line."""
    with open(filepath, "r") as f:
        for i, line in enumerate(f):
            if line.strip().startswith(section_name):
                return i
    raise ValueError(f"Section '{section_name}' not found")

csv_path = "walk_2m_01_real_1.csv"

# --- Force Plates (Devices section) ---
dev_row = find_section_row(csv_path, "Devices")
fp_df = pd.read_csv(csv_path, skiprows=dev_row + 4, header=None)
# col 0 = Frame, col 1 = Sub Frame, then 9 cols per plate × 4 plates

# --- Trajectories section ---
traj_row = find_section_row(csv_path, "Trajectories")
marker_header = pd.read_csv(csv_path, skiprows=traj_row + 1, nrows=1, header=None)
traj_df = pd.read_csv(csv_path, skiprows=traj_row + 4, header=None)
# col 0 = Frame, col 1 = Sub Frame, then X Y Z per marker (3 cols each)
```

---

## Marker Column Lookup Helper

```python
def build_marker_col_map(csv_path: str, traj_section_row: int) -> dict[str, list[int]]:
    """
    Parse the marker name row and return:
      marker_short_name → [col_X, col_Y, col_Z]  (0-based, including Frame/SubFrame cols)
    Marker name row = traj_section_row + 2 (0-based).
    """
    raw = pd.read_csv(csv_path, skiprows=traj_section_row + 2,
                      nrows=1, header=None).iloc[0].tolist()
    marker_map = {}
    col = 2  # skip Frame and SubFrame
    while col < len(raw):
        name = str(raw[col]).strip()
        if pd.notna(raw[col]) and name:
            short = name.split(":")[-1]   # strip namespace prefix
            marker_map[short] = [col, col + 1, col + 2]
        col += 3
    return marker_map

# Usage
traj_row = find_section_row(csv_path, "Trajectories")
marker_col_map = build_marker_col_map(csv_path, traj_row)

def get_marker_col(name: str) -> list[int]:
    """Return [col_X, col_Y, col_Z] for the named marker."""
    return marker_col_map[name]
```
