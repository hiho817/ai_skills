# Time Synchronisation Reference

## Concept

A physical trigger fires at the same instant in both VICON and ROS2.

- **VICON side**: The `Tigger` marker first becomes visible when the trigger fires.
- **ROS2 side**: A single `/trigger` message is recorded in the bag.

The offset `dt = t_ros_trigger − t_vicon_trigger` shifts the VICON time axis to absolute ROS time.

---

## VICON Side — Tigger Marker Detection

```python
# Get Tigger marker columns from traj_df
tigger_col = get_marker_col("Tigger")   # returns [col_X, col_Y, col_Z]

# First frame where Tigger is visible (all three XYZ are not NaN)
first_valid_mask = traj_df[tigger_col].notna().all(axis=1)
frame_trigger = traj_df.index[first_valid_mask][0]   # 0-based index

traj_fs = 500.0   # Hz
t_vicon_trigger = frame_trigger / traj_fs   # seconds from VICON recording start
```

> **Reminder**: The marker is named `"Tigger"` (known typo, not "Trigger").

---

## ROS2 Side — `/trigger` Topic

```python
import rosbag2_py
from rclpy.serialization import deserialize_message
from rosidl_runtime_py.utilities import get_message

def read_trigger_stamp(bag_path: str) -> float:
    """Return ROS time (seconds) of the single /trigger message."""
    reader = rosbag2_py.SequentialReader()
    storage_opts = rosbag2_py.StorageOptions(uri=bag_path, storage_id="sqlite3")
    conv_opts = rosbag2_py.ConverterOptions("", "")
    reader.open(storage_opts, conv_opts)

    msg_type = get_message("corgi_msgs/msg/TriggerStamped")
    while reader.has_next():
        topic, data, _ = reader.read_next()
        if topic == "/trigger":
            msg = deserialize_message(data, msg_type)
            stamp = msg.header.stamp
            return stamp.sec + stamp.nanosec * 1e-9
    raise RuntimeError("/trigger message not found in bag")

t_ros_trigger = read_trigger_stamp("leg_odom20260507_161231/")
```

---

## Compute Offset and Aligned Time Axes

```python
dt = t_ros_trigger - t_vicon_trigger   # seconds

# Trajectory time axis (500 Hz)
n_traj = len(traj_df)
t_traj = np.arange(n_traj) / traj_fs + dt   # absolute ROS time

# Force plate time axis (1000 Hz) — same dt, different frequency
fp_fs = 1000.0
n_fp = len(fp_df)
t_fp = np.arange(n_fp) / fp_fs + dt         # absolute ROS time
```

> The same `dt` applies to **both** the 500 Hz Trajectory and 1000 Hz Force Plate data.
> To align them to a common time axis use `np.interp` or `scipy.interpolate.interp1d`.
