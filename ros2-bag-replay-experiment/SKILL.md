---
name: ros2-bag-replay-experiment
description: 'ROS2 bag 重跑實驗流程：replay 舊 bag 驗證新節點程式碼，再錄新 bag 並分析。Use when: ros2 bag replay, ros2 bag record, corgi_leg_odom, corgi_odometry, corgi_fusion_node, odom_mapping, re-run experiment, 重播 bag, 重錄 bag, replay and record, EKF analysis, VICON comparison, 驗證修改效果, 座標系修正, coordinate frame fix, camera_init, odom frame。包含常見陷阱：殘留節點、/ekf 污染、metadata.yaml 缺失、兩個節點同時 publish、bash source 失敗。'
argument-hint: '說明要測試什麼修改，例如：velocity_r/l 除以 4、fusion node 座標系修正'
---

# ROS2 Bag Replay 實驗流程

## 目的

修改 node 程式碼後，透過 replay 舊 bag 驗證新行為，並錄製新 bag 供後續分析比較。

---

## 關鍵陷阱（必讀）

### ⚠️ 陷阱 1：殘留舊節點（最常發生）

**症狀**：分析圖出現帶狀震盪，position 範圍異常大（>0.5m），但 velocity RMSE 卻很小。

**原因**：上次實驗的節點沒有被確實 kill，背景還在 publish `/ekf`。新舊兩個節點同時發布，bag 錄到兩條軌跡交替出現（可能 >10000 次位置重置）。

**檢測**：
```bash
ps aux | grep corgi_leg_odom | grep -v grep
ros2 node list | grep leg
```

**修復**：
```bash
kill -9 <PID>
# 確認
ps aux | grep corgi_leg_odom | grep -v grep || echo "all killed"
```

---

### ⚠️ 陷阱 2：replay 時把舊 /ekf 一起播出

**症狀**：新錄的 bag 訊息數量是預期的兩倍（例如 15000 而非 7700）。

**原因**：原始 bag 本身含有 `/ekf` topic（舊的計算結果）。`ros2 bag play` 預設播所有 topic，導致舊 `/ekf` 也被播出並錄進新 bag。

**正確做法**：replay 時必須用 `--topics` 只播輸入資料：
```bash
ros2 bag play <原始bag路徑> --topics /motor/state /imu_raw /trigger
```

**確認方法**：新 bag 的 `/ekf` 訊息數應接近原始 bag 的 `/motor/state` 訊息數的 1/3 左右（受限於 EKF 發布頻率）。

---

### ⚠️ 陷阱 3：metadata.yaml 缺失

**症狀**：`ros2 bag info` 或 rosbag2_py 開啟 bag 時報錯 `disk I/O error` 或 `No storage could be initialized`。

**原因**：recorder 被強制 kill 時來不及寫入 `metadata.yaml`，但 `.db3` 檔案本身是完整的。

**修復**：用 python sqlite3 直接查詢 db3，再手動產生 metadata.yaml：
```bash
python3 - <<'EOF'
import sqlite3, yaml
db = '<bag_dir>/<bag_name>_0.db3'
conn = sqlite3.connect(db)
cur = conn.cursor()
cur.execute("SELECT id, name, type FROM topics")
topics = cur.fetchall()
topic_counts = {}
for t in topics:
    cur.execute(f"SELECT COUNT(*), MIN(timestamp), MAX(timestamp) FROM messages WHERE topic_id={t[0]}")
    row = cur.fetchone()
    topic_counts[t[1]] = {'type': t[2], 'count': row[0], 'tmin': row[1], 'tmax': row[2]}
conn.close()
tmin_all = min(v['tmin'] for v in topic_counts.values())
tmax_all = max(v['tmax'] for v in topic_counts.values())
total = sum(v['count'] for v in topic_counts.values())
metadata = {'rosbag2_bagfile_information': {
    'version': 6, 'storage_identifier': 'sqlite3',
    'duration': {'nanoseconds': int(tmax_all - tmin_all)},
    'starting_time': {'nanoseconds_since_epoch': int(tmin_all)},
    'message_count': total,
    'topics_with_message_count': [
        {'topic_metadata': {'name': n, 'type': i['type'],
         'serialization_format': 'cdr', 'offered_qos_profiles': ''},
         'message_count': i['count']}
        for n, i in topic_counts.items()],
    'compression_format': '', 'compression_mode': '',
    'relative_file_paths': ['<bag_name>_0.db3'],
    'files': [{'path': '<bag_name>_0.db3',
               'starting_time': {'nanoseconds_since_epoch': int(tmin_all)},
               'duration': {'nanoseconds': int(tmax_all - tmin_all)},
               'message_count': total}]}}
with open('<bag_dir>/metadata.yaml', 'w') as f:
    yaml.dump(metadata, f, default_flow_style=False)
print("metadata.yaml written")
for n, i in topic_counts.items():
    print(f"  {n}: count={i['count']}")
EOF
```

---

### ⚠️ 陷阱 4：Async terminal 的 recorder 被 suspend（tty 輸出問題）

**症狀**：recorder 顯示 `suspended (tty output)`，bag 目錄沒有被建立。

**原因**：在同一個 shell session 用 `&` 背景執行 `ros2 bag record` 時，若 stdout/stderr 沒有重導向會被 suspend。

**正確做法**：一律將 recorder 輸出重導向：
```bash
ros2 bag record -o <output_path> /ekf /trigger > /tmp/recorder.log 2>&1
```
或用獨立的 async terminal（run_in_terminal mode=async）。

---

### ⚠️ 陷阱 5：在 terminal 中無法直接 `source` ROS setup 檔

**症狀**：執行 `source /opt/ros/humble/setup.bash` 後接著執行 ROS 指令，出現 `No such file or directory: setup.sh` 或 `ros2: command not found`。

**原因**：Copilot terminal 的預設 shell 環境可能不完整，直接 `source` 不會持久化到後續指令。

**正確做法**：所有 ROS 相關指令一律用 `bash -c "source ... && ..."` 包起來：
```bash
# ✅ 正確
bash -c "source /opt/ros/humble/setup.bash && source <ws>/install/setup.bash && ros2 run ..."
bash -c "source /opt/ros/humble/setup.bash && source <ws>/install/setup.bash && python3 analyze.py"

# ❌ 錯誤
source /opt/ros/humble/setup.bash
ros2 run ...   # 上面的 source 不會生效
```

---

### ⚠️ 陷阱 6：/odom_mapping 輸出在 camera_init 框架而非 odom 框架

**症狀**：`/odom_mapping` 起始位置不在 (0,0,0)，或與 `/ekf` 方向完全不同（幾乎垂直）。

**原因**：FAST-LIO 的 `/lidar_odom` 以 `camera_init` 框架發布，初始四元數約 `q≈(0.70, -0.12, -0.12, -0.69)`（≈90° 旋轉）。若融合節點直接用 lidar 的位置反推 map→odom 的偏移，`map` 框架就會繼承 `camera_init` 的朝向。

**修正方法**：在 `FusionNode.cpp` 的 `cb_lidar()` 中，第一次 lidar+EKF 配對時計算一次 `T_{odom←camera_init}`：
```cpp
// 新增成員（FusionNode.hpp）
bool lidar_frame_init_ = false;
Eigen::Vector3f    t_co_;
Eigen::Quaternionf q_co_;

// 第一次匹配（FusionNode.cpp cb_lidar）
q_co_ = (bs.q_odom * q_lidar.inverse()).normalized();
t_co_ = bs.p_odom - q_co_.toRotationMatrix() * p_lidar;
lidar_frame_init_ = true;
// 外層 EKF 以 map=odom 初始化（從原點出發）
p_mo = Eigen::Vector3f::Zero();
q_mo = Eigen::Quaternionf::Identity();

// 後續每筆 lidar 量測先轉換
p_lidar_odom = q_co_.toRotationMatrix() * p_lidar + t_co_;
q_lidar_odom = (q_co_ * q_lidar).normalized();
```

同時將 `publish_odom_mapping()` 的 `frame_id` 改為 `odom_frame_`（而非 `map_frame_`）。

**驗證**：節點啟動後日誌應出現：
```
T_{odom←camera_init} set: t=[...] q=[...]
OuterEKF initialised: map frame aligned with odom frame (p_mo=0, q_mo=I)
```

---

## 標準實驗流程

### 情境 A：只 replay leg_odom（無 LiDAR 融合）

bag 來源 topics：`/imu_raw /motor/state /trigger /gmo/contact_state`

---

### 情境 B：replay leg_odom + corgi_fusion_node（含 LiDAR 融合）

bag 來源 topics：`/imu_raw /motor/state /trigger /gmo/contact_state /lidar_odom`

#### B.1 使用 Python 腳本統一管理（推薦）

用 Python 腳本同時管理所有 process，用 `os.killpg` 確保乾淨結束：

```python
# run_replay.py 骨架
import subprocess, os, signal, time

WS = "/home/hiho817/corgi_ws/corgi_ros2_ws"
SRC = f"source /opt/ros/humble/setup.bash && source {WS}/install/setup.bash"
INPUT_BAG = "<原始bag路徑>"
OUTPUT_BAG = "<輸出bag路徑>"

def start(cmd):
    return subprocess.Popen(
        ["bash", "-c", f"{SRC} && {cmd}"],
        preexec_fn=os.setsid  # 建立 process group，方便 kill
    )

# 啟動節點
leg = start("ros2 launch corgi_odometry odom_fusion_replay.launch.py")
time.sleep(3)

# 啟動 recorder（含所有要分析的 topics）
rec = start(f"ros2 bag record -o {OUTPUT_BAG} "
            f"/ekf /ekf/orientation /lidar_odom /odom_mapping /fusion/bv "
            f"> /tmp/recorder.log 2>&1")
time.sleep(2)

# Replay（只播輸入 topics，排除計算結果）
play = subprocess.run(["bash", "-c",
    f"{SRC} && ros2 bag play {INPUT_BAG} --clock --rate 2.0 "
    f"--topics /imu_raw /motor/state /trigger /gmo/contact_state /lidar_odom"])

time.sleep(3)

# 乾淨結束（kill 整個 process group）
for p in [rec, leg]:
    try:
        os.killpg(os.getpgid(p.pid), signal.SIGINT)
    except Exception:
        pass
```

#### B.2 使用 launch file（`odom_fusion_replay.launch.py`）

```python
# src/corgi_odometry/launch/odom_fusion_replay.launch.py
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(package='corgi_odometry', executable='corgi_leg_odom',
             parameters=[{'use_sim_time': True}],
             remappings=[('/imu', '/imu_raw')]),
        Node(package='corgi_odometry', executable='corgi_fusion_node',
             parameters=[{'use_sim_time': True}]),
    ])
```

#### B.3 Replay 指令

```bash
bash -c "source /opt/ros/humble/setup.bash && source <ws>/install/setup.bash && \
  ros2 bag play <原始bag路徑> --clock --rate 2.0 \
  --topics /imu_raw /motor/state /trigger /gmo/contact_state /lidar_odom"
```

**注意**：`--rate 2.0` 加速 replay，實際時間約縮短一半。`--clock` 確保節點使用 sim time。

---

### 前置檢查

```bash
# 1. 確認沒有殘留節點
ps aux | grep corgi_leg_odom | grep -v grep || echo "OK"

# 2. 確認舊 bag 內容（確認哪些 topics 要排除）
ros2 bag info <原始bag路徑>

# 3. 清理上次失敗的輸出 bag（若有）
rm -rf <output_bag_dir>
```

### Step 1：啟動新節點（async terminal）

```bash
source <workspace>/install/setup.bash 2>/dev/null
ros2 run corgi_odometry corgi_leg_odom --ros-args --remap /imu:=/imu_raw > /tmp/leg_odom.log 2>&1
```

等待 3 秒後確認啟動：
```bash
cat /tmp/leg_odom.log | tail -5
# 應看到 "Leg Odometry Node Started"
```

### Step 2：啟動 Recorder（async terminal）

```bash
ros2 bag record -o <output_bag_dir> /ekf /trigger > /tmp/recorder.log 2>&1
```

等待 3 秒後確認：
```bash
cat /tmp/recorder.log
# 應看到 "Subscribed to topic '/ekf'" 和 "Recording..."
```

### Step 3：Replay（排除舊 /ekf）

```bash
ros2 bag play <原始bag路徑> --topics /motor/state /imu_raw /trigger
```

等待 replay 結束（約與原始 bag 時長相同）。

### Step 4：停止（依序）

1. kill recorder terminal（kill_terminal 或 Ctrl+C）
2. kill node terminal（kill_terminal 或 Ctrl+C）

### Step 5：驗證 bag 內容

```bash
python3 -c "
import sqlite3
db = '<output_bag_dir>/<bag_name>_0.db3'
conn = sqlite3.connect(db)
cur = conn.cursor()
cur.execute('SELECT id, name FROM topics')
for t in cur.fetchall():
    cur.execute(f'SELECT COUNT(*) FROM messages WHERE topic_id={t[0]}')
    print(f'{t[1]}: {cur.fetchone()[0]} msgs')
conn.close()
"
```

**合格標準**：
- `/ekf` 訊息數量接近原始 bag 的 `/motor/state` 數量的 1/3（EKF 以 ~500 Hz 發布，motor_state 以 ~500 Hz 輸入）
- `/trigger` 訊息數量 = 1
- 不應超過原始 `/ekf` 數量的 110%（若超過代表有殘留節點）

### Step 6：產生 metadata.yaml（若缺失）

見陷阱 3 的修復方法。

### Step 7：執行分析

**分析腳本慣例**：使用 `analysis_ws/experiments/<日期>/<實驗名>/results/analyze.py`。

```bash
cd /home/hiho817/analysis_ws
bash -c "source /opt/ros/humble/setup.bash && \
         source /home/hiho817/corgi_ws/corgi_ros2_ws/install/setup.bash && \
         python3 experiments/<日期>/<實驗名>/results/analyze.py"
```

**分析腳本重要模式**：

```python
# 1. 讀取 bag（用 sqlite3 直接查詢，不依賴 rosbag2_py）
import sqlite3
from rclpy.serialization import deserialize_message
from nav_msgs.msg import Odometry

def load_odom(db_path, topic_name, t_start=None, t_cutoff=None):
    conn = sqlite3.connect(db_path)
    # ...

# 2. 兩階段分析（步行 vs 被抬起）
T_WALK_END = 19.0   # 機器人開始被抬起的時間 [s]
T_CUTOFF   = 26.0   # 資料有效截止時間 [s]
mask_walk = data['t'] < T_WALK_END

# 3. lidar_odom 框架轉換（camera_init → odom）
# T_co 從 fusion node 日誌取得：
# "T_{odom←camera_init} set: t=[...] q=[...]"
R_co = quat_to_rotmat(*T_CO_q)
x_odom = R_co[0,0]*x + R_co[0,1]*y + R_co[0,2]*z + T_CO_t[0]
# ...（同理 y, z）

# 4. EKF vs Mapping 差值只在步行階段計算
from scipy.interpolate import interp1d
interp_x = interp1d(odom['t'][walk_mask], odom['x'][walk_mask])
diff_x = ekf['x'][...] - interp_x(t_common)
rms_x = np.sqrt(np.mean(diff_x**2))
```

**合格標準（含 LiDAR 融合）**：
- `/odom_mapping` 起始位置 ≈ (0, 0, 0)（座標系修正生效）
- 步行階段 EKF vs mapping 3D RMS < 0.05 m
- `frame_id = odom`（不是 `map` 或 `camera_init`）

---

## 結果合格性檢查

| 檢查項目 | 判斷標準 |
|---|---|
| Position 圖是單條線 | ✅ 正常。若為帶狀 → 殘留節點問題 |
| /ekf 訊息數量合理 | ✅ ~7700 msgs / 45s bag。若 >15000 → 兩個節點污染 |
| /odom_mapping 起始位置 | ✅ ≈ (0,0,0)。若偏移 > 0.1 m → 座標系框架問題（陷阱 6） |
| RMSE 輸出完整 | ✅ Position / Attitude / Velocity 都有值 |
| 分析時間窗口 >20s | ✅ `Common window: X – Y s (>20s)`，若 <5s → 時間同步問題 |
| 步行階段 RMS 差距 | ✅ EKF vs mapping 3D RMS < 0.05 m（典型值 ~0.016 m） |

---

## 相關檔案位置

- 原始 bag：`/home/hiho817/corgi_ws/corgi_ros2_ws/bag/<bag_name>/`
- 分析工作區：`/home/hiho817/analysis_ws/experiments/<日期>/<實驗名>/`
- replay 腳本：`/home/hiho817/analysis_ws/tools/run_replay.py`
- 修改的節點：`/home/hiho817/corgi_ws/corgi_ros2_ws/src/corgi_odometry/src/`
- fusion launch：`src/corgi_odometry/launch/odom_fusion_replay.launch.py`

## 0508 實驗參考案例

- **問題**：`/odom_mapping` 在 camera_init 框架（陷阱 6）
- **修正**：重寫 `FusionNode.cpp::cb_lidar()`，計算 `T_{odom←camera_init}`
- **驗證 bag**：`replay_fixed_20260510_034717/`（EKF=28088 msgs，Mapping=1207 msgs）
- **分析結果**：步行 1.88 m，EKF vs mapping 3D RMS = 0.016 m
- **分析腳本**：`analysis_ws/datas/0508/analyze_v2.py`（含兩階段分析與 lidar 框架轉換）
