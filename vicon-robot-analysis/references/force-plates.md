# Force Plates Reference

## Column Layout in CSV

The Devices section contains **four force plates**. Export column order:

> **ForcePlate4 → ForcePlate2 → ForcePlate1 → ForcePlate3**  
> (this is an artifact of the VICON export — not sequential)

### Columns per Plate (9 columns each)

| # | Label | Unit |
|---|-------|------|
| 0 | Fx | N |
| 1 | Fy | N |
| 2 | Fz | N |
| 3 | Mx | N·mm |
| 4 | My | N·mm |
| 5 | Mz | N·mm |
| 6 | Cx | mm |
| 7 | Cy | mm |
| 8 | Cz | mm |

Columns 0 and 1 of `fp_df` are `Frame` and `Sub Frame`. Plate data starts at column 2.

---

## Accessing a Specific Plate

```python
# Plate index in export order: 0=FP4, 1=FP2, 2=FP1, 3=FP3
PLATE_EXPORT_ORDER = ["ForcePlate4", "ForcePlate2", "ForcePlate1", "ForcePlate3"]

def get_plate_cols(plate_name: str) -> list[int]:
    """Return column indices [Fx..Cz] for the named force plate in fp_df."""
    idx = PLATE_EXPORT_ORDER.index(plate_name)
    start = 2 + idx * 9
    return list(range(start, start + 9))

# Example: get ForcePlate1 data
fp1_cols = get_plate_cols("ForcePlate1")
fp1_df = fp_df[fp1_cols].copy()
fp1_df.columns = ["Fx", "Fy", "Fz", "Mx", "My", "Mz", "Cx", "Cy", "Cz"]
```

---

## Time Alignment

Force plate data runs at **1000 Hz**. After computing `dt` from the trigger sync:

```python
fp_fs = 1000.0
t_fp = np.arange(len(fp_df)) / fp_fs + dt   # absolute ROS time

# Interpolate to trajectory time axis (500 Hz) if needed
import numpy as np
fp1_fz = fp_df[get_plate_cols("ForcePlate1")[2]].values   # Fz column
fp1_fz_interp = np.interp(t_traj, t_fp, fp1_fz)
```
