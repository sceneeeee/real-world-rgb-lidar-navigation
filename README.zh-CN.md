# Real-World RGB-LiDAR Navigation

[English](README.md) | [简体中文](README.zh-CN.md)

一个面向真实移动机器人的 ROS 2 项目：在现有 **MID360 + Nav2** 平台之上，逐步加入 **开放词汇视觉感知、RGB-LiDAR 3D Grounding、语义导航、持续物体记忆**，最终扩展到 **VLN / 语言引导导航**。

> **当前状态：proposal / platform onboarding。**  
> 只有在真实机器人上完成集成和验证后，某项能力才算真正实现。

## 项目目标

现有机器人已经能够定位、规划、避障并到达一个 metric pose。本项目增加的是它目前缺少的语义层：

~~~text
metric goal
    ↓
visible object goal
    ↓
remembered object goal
    ↓
language-guided navigation
~~~

整个系统采用分层结构。VLN 不负责替代 SLAM、Nav2 或底层控制，而是在真实感知和语义导航稳定之后，作为高层决策模块加入。

## 已冻结的硬件与传感器方案

初期硬件路线固定为：

~~~text
Jetson Orin NX
├── Intel RealSense D435
│   ├── RGB    → 主要语义 / 视觉输入
│   └── Depth  → baseline、验证、消融、可选融合
│
└── Livox MID360
    └── 3D geometry → 3D grounding 与 navigation 的主要几何来源
~~~

除非后续实测证明存在明确硬件问题，否则不再额外挂 RGB-only Camera；**D435 的 RGB 就作为项目的 RGB Camera。**

### 主线和 Depth baseline 的区分

项目的**主 semantic-grounding 路线**固定为：

~~~text
D435 RGB + MID360
~~~

D435 depth 会保持可用并记录，但主 semantic-navigation pipeline 必须能够在 camera depth 关闭时正常工作。

计划比较三种 grounding 方案：

| Variant | Semantic 3D grounding 使用的几何 | 角色 |
|---|---|---|
| **A — RGB-D** | D435 aligned depth | baseline |
| **B — RGB + MID360** | 投影到 RGB 图像中的 MID360 点云 | **主方案** |
| **C — RGB-D + MID360** | camera depth + LiDAR geometry | 可选融合 / upper-bound comparison |

注意：三组实验中 MID360 仍然参与现有 FAST-LIO2 / Nav2 平台。这里比较的是**把语义观测提升到 3D 时使用哪一种几何来源**，不是把机器人底层定位切换成 RGB-D。

因此项目形成一个明确问题：

> **当移动机器人已经搭载 3D LiDAR 时，额外的 RGB-D depth 对真实 semantic grounding 和 semantic navigation 到底能带来多少收益？**

## 与现有机器人仓库的边界

机器人底层平台由以下仓库维护：

**[sceneeeee/XJTLU-autonomous-vehicle-rtk](https://github.com/sceneeeee/XJTLU-autonomous-vehicle-rtk)**

现有平台已经提供：

- NVIDIA Jetson Orin NX、Ubuntu 22.04、ROS 2 Humble
- Livox MID360 + IMU
- FAST-LIO2 局部里程计
- PGO 全局修正
- map → odom → base_link TF
- Nav2 + MPPI
- STM32 / chassis interface
- 室内实机导航
- runtime logging 与 rosbag 调试

**本仓库不重复实现这些底层组件。**

## 目标系统架构

~~~text
                     Intel RealSense D435
                    ┌──────────┴──────────┐
                    │                     │
                   RGB                  Depth
                    │                     │
                    │                 baseline /
                    │                 ablation
                    ↓                     │
           NanoOWL open-vocabulary        │
                detection                 │
                    ↓                     │
               NanoSAM mask               │
                    ↓                     │
              semantic mask               │
                    │                     │
                    ├──────────────┐      │
                    │              │      │
                    │         Livox MID360│
                    │              │      │
                    └──────┬───────┘      │
                           ↓              │
                  RGB-LiDAR 3D grounding ←┘ optional
                           ↓
                  map-frame object pose
                           ↓
                  semantic goal generation
                           ↓
                     Nav2 NavigateToPose
                           ↓
                        real robot

后续：
map-frame observations
        ↓
persistent object memory
        ↓
language / VLN reasoning
        ↓
semantic waypoint(s)
        ↓
Nav2
~~~

## Reference-Driven Roadmap

每个 milestone 都绑定一个明确的工程目标和初始 reference。这里的 reference 表示“从什么方案开始复现 / 借鉴”，**不代表这些代码目前已经集成完成**。

| Milestone | 目标 | 主要 reference / repo | 实机结果 |
|---|---|---|---|
| **M0 — Platform Onboarding** | 验证已有 metric-navigation stack | 现有 robot repo、[Livox ROS Driver 2](https://github.com/Livox-SDK/livox_ros_driver2)、[FAST_LIO](https://github.com/hku-mars/FAST_LIO)、[Navigation2](https://github.com/ros-navigation/navigation2) | RViz 点目标 → 实机自主导航 |
| **M1 — D435 + MID360 Integration** | 接入 D435，并完成 camera ↔ LiDAR 标定 | [realsense-ros](https://github.com/realsenseai/realsense-ros)、[FAST-Calib](https://github.com/hku-mars/FAST-Calib)、ROS 2 tf2 | RGB/depth 稳定输出，MID360 正确投影到 RGB |
| **M2 — RGB-LiDAR 3D Grounding** | 检测物体并得到 map-frame 3D 坐标 | [NanoOWL](https://github.com/NVIDIA-AI-IOT/nanoowl)、[NanoSAM](https://github.com/NVIDIA-AI-IOT/nanosam)、ConceptFusion-style 2D→3D semantic lifting | 看到 chair → 得到真实 3D / map 坐标 |
| **M3 — Semantic Navigation** | 把语义目标转换为安全 Nav2 approach pose | [VLMaps](https://github.com/vlmaps/vlmaps) 作为 semantic-navigation reference、[Navigation2](https://github.com/ros-navigation/navigation2) 执行 | go to the chair → 实机走到 chair 附近 |
| **M4 — Persistent Spatial Memory** | 跨时间维护 object-level memory | [ConceptGraphs](https://github.com/concept-graphs/concept-graphs) 作为 object-memory / scene-graph reference | 目标离开视野后仍能回到其位置 |
| **M5 — VLN** | 在 persistent semantic map 上加入语言推理 | **进入 M5 前选定并冻结一篇具体 VLN paper + repo** | 执行多步语言导航 |
| **M6 — Deployment / Simulation Extension** | Jetson 优化；只有有必要时才增加仿真 | TensorRT / Jetson profiling；必要时 Isaac Sim | deployment benchmark / reproducible extension |

### 执行顺序

~~~text
M0 → M1 → M2 → M3 → M4 → M5
~~~

M0–M3 没有在真实机器人上稳定之前，不把 long-horizon VLN 作为主要开发任务。

---

## M0 — 验证已有实机 Baseline

### 目标

M0 不开发新的 perception algorithm，只确认已有 robot platform 能可靠作为后续依赖。

~~~text
MID360 + IMU
      ↓
FAST-LIO2
      ↓
PGO
      ↓
map → odom → base_link
      ↓
Nav2 + MPPI
      ↓
NavigateToPose
      ↓
real robot
~~~

### Reference

- **Robot source of truth：** [sceneeeee/XJTLU-autonomous-vehicle-rtk](https://github.com/sceneeeee/XJTLU-autonomous-vehicle-rtk)
- **MID360 ROS：** [Livox-SDK/livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2)
- **LiDAR-Inertial Odometry：** [hku-mars/FAST_LIO](https://github.com/hku-mars/FAST_LIO)
- **Navigation：** [ros-navigation/navigation2](https://github.com/ros-navigation/navigation2)
- **PGO：** 直接复用现有 robot repo 已集成的实现；M0 不替换它。

### M0 验收

~~~text
[ ] /livox/lidar 稳定
[ ] FAST-LIO2 registered cloud / odometry 正常
[ ] PGO / global correction 正常
[ ] map → odom → base_link 正常
[ ] NavigateToPose action 在线
[ ] RViz fixed frame = map
[ ] 完成一次短距离实机 click-to-go
[ ] 保存本次运行日志
~~~

---

## M1 — D435 + MID360 接入与标定

### 1. D435 Bringup

实验室相机固定为 **Intel RealSense D435**。

初始 implementation reference：

- [realsenseai/realsense-ros](https://github.com/realsenseai/realsense-ros)

需要获得：

~~~text
D435
├── RGB image
├── RGB camera_info
├── depth image
├── aligned depth
└── camera TF frames
~~~

RGB 是主线必须输入；depth 始终保留用于 baseline / validation。

### 2. Camera ↔ LiDAR 外参

初始 calibration reference：

- [hku-mars/FAST-Calib](https://github.com/hku-mars/FAST-Calib)

目标得到：

~~~text
T_camera_lidar
~~~

并通过 ROS 2 TF 统一：

~~~text
camera frame ↔ MID360 frame ↔ base_link ↔ map
~~~

### 3. M1 验收

~~~text
[ ] RGB stream 稳定
[ ] depth stream 稳定
[ ] camera intrinsics 正确
[ ] timestamps 能与 MID360 配合
[ ] camera ↔ MID360 extrinsics 得到
[ ] MID360 点能正确投影到 RGB
[ ] aligned D435 depth 可用于后续 baseline
~~~

---

## M2 — 开放词汇感知 + RGB-LiDAR 3D Grounding

M2 分成两个明确问题：**先在 image space 找到目标，再把目标提升到 3D。**

### M2.1 Open-Vocabulary Perception

第一版直接冻结为：

~~~text
D435 RGB
   ↓
NanoOWL
   ↓
open-vocabulary bounding box
   ↓
NanoSAM
   ↓
object mask
~~~

References：

- **OWL-ViT / TensorRT deployment：** [NVIDIA-AI-IOT/nanoowl](https://github.com/NVIDIA-AI-IOT/nanoowl)
- **Jetson promptable segmentation：** [NVIDIA-AI-IOT/nanosam](https://github.com/NVIDIA-AI-IOT/nanosam)

第一版 vocabulary 保持简单：

~~~text
chair
door
sofa
printer
table
~~~

第一版实现期间不继续随意换 detector。只有 Jetson 实测证明 accuracy、latency 或 deployment compatibility 不满足需求时才替换。

### M2.2 主路线 — RGB + MID360

~~~text
object mask
     │
MID360 point cloud
     ↓
使用外参变换到 camera
     ↓
投影 LiDAR points → RGB image
     ↓
保留 object mask 内的点
     ↓
range / outlier filtering
     ↓
3D clustering
     ↓
robust object centroid / extent
     ↓
transform → map frame
~~~

语义从 2D lifting 到 3D 的整体思想参考 **ConceptFusion** 一类 open-set 2D→3D mapping 工作；但本项目主线不是依赖 camera depth，而是使用 MID360 作为主要 3D geometry。

LiDAR projection / filtering 本身作为本项目 engineering module：基于已标定的 camera projection、ROS TF，以及标准 point-cloud filtering / clustering，不把它包装成新的独立 perception model。

### M2.3 三组 Grounding Baseline

#### A — RGB-D baseline

~~~text
RGB mask
   +
D435 aligned depth
   ↓
back-project object pixels
   ↓
3D object position
~~~

#### B — RGB + MID360 — 主方案

~~~text
RGB mask
   +
MID360 projection
   ↓
associated LiDAR points
   ↓
3D object position
~~~

#### C — RGB-D + MID360 — 可选融合

~~~text
D435 depth
   +
MID360 geometry
   ↓
cross-check / joint estimate
   ↓
3D object position
~~~

### M2 评估

- 3D object-position error
- grounding success rate
- effective range / coverage
- latency
- Jetson CPU / GPU usage
- occlusion、reflective surface、LiDAR sparse support、depth failure 等 failure cases

---

## M3 — Semantic Object Navigation

M3 把语义目标变成可以执行的 metric goal。

Reference：

- [VLMaps — Visual Language Maps for Robot Navigation](https://github.com/vlmaps/vlmaps)：参考“把 semantic / language concept spatially grounded 到可导航位置”的架构思路。
- [Navigation2](https://github.com/ros-navigation/navigation2)：真实机器人上的实际执行框架。

**不会用 VLMaps 替换现有 Nav2。**

~~~text
semantic target
      ↓
M2 map-frame object position
      ↓
candidate approach poses
      ↓
costmap / collision / reachability checks
      ↓
选择安全、朝向目标的 pose
      ↓
Nav2 NavigateToPose
      ↓
real robot
~~~

机器人不能直接导航到 object centroid，而要导航到物体周围一个安全的 approach pose。

### M3 验收示例

~~~text
"go to the chair"
        ↓
chair detected / grounded
        ↓
safe approach pose
        ↓
NavigateToPose
        ↓
robot reaches chair area
~~~

主要指标：

- semantic-goal success rate
- navigation success rate
- final distance to target
- approach-pose validity
- end-to-end latency
- failure category

---

## M4 — Persistent Object-Level Spatial Memory

M4 的核心是：**目标即使已经离开当前 camera FoV，机器人仍然知道它在哪里。**

主要 reference：

- [ConceptGraphs](https://github.com/concept-graphs/concept-graphs)

第一版只借鉴它的 **object-centric persistent memory / scene representation** 思路，不要求完整复现整套 ConceptGraphs。

一个 memory node 可以包含：

~~~text
object_id
semantic_label
visual_embedding
map-frame position
3D extent
confidence
observation_count
first_seen
last_seen
representative observation(s)
~~~

重复观测通过：

~~~text
spatial consistency
+
semantic similarity
+
visual similarity
~~~

进行 object association，然后更新 persistent record。

示例：

~~~text
chair_01    → (2.1, 4.5)
chair_02    → (5.2, 1.8)
printer_01  → (7.4, 3.1)
door_01     → (...)
~~~

随后：

~~~text
target 当前不可见
        ↓
query persistent memory
        ↓
retrieve stored map-frame object
        ↓
generate safe approach pose
        ↓
Nav2
~~~

---

## M5 — VLN

M5 明确放在 M4 之后：

~~~text
language instruction
        ↓
selected VLN / VLM reasoning method
        ↓
persistent semantic-spatial memory
        ↓
semantic waypoint(s)
        ↓
Nav2 execution
~~~

与早期“同时参考很多 VLN 方法”的做法不同，**进入 M5 前只选定一篇具体 VLN paper + 官方 / 主流 repo 作为主 reference，然后冻结实现范围**，不在开发过程中持续换方向。

## 实验设计

核心 semantic-grounding 对比：

~~~text
A: D435 RGB + D435 depth
B: D435 RGB + MID360          [主方案]
C: D435 RGB + D435 depth + MID360
~~~

| Metric | A: RGB-D | B: RGB+MID360 | C: RGB-D+MID360 |
|---|---:|---:|---:|
| 3D localization error | measure | measure | measure |
| grounding success rate | measure | measure | measure |
| effective range | measure | measure | measure |
| latency | measure | measure | measure |
| Jetson resource usage | measure | measure | measure |

Semantic navigation：

| Metric | 定义 |
|---|---|
| success rate | robot 是否到达目标物体周围有效 approach region |
| final target distance | robot / approach pose 到 semantic target 的距离 |
| execution time | command → arrival |
| failure type | perception / grounding / memory / planning / control |

## 设计原则

1. **Real robot first。** 仿真不能替代真实机器人验证。
2. **不重造底层平台。** FAST-LIO2、PGO、Nav2、MPPI、chassis control 都保持为外部依赖。
3. **每个关键设计都有 reference。** 每个 milestone 都从明确 paper、repo 或现有平台实现出发。
4. **先冻结第一版，再根据数据改。** 没有量化 failure，不随意换模型。
5. **MID360 是主要 geometry source。** D435 depth 很有用，但主 semantic pipeline 不能依赖它。
6. **对比必须可解释。** RGB-D、RGB+LiDAR、RGB-D+LiDAR 是三条明确分开的 grounding variant。
7. **Hierarchical autonomy。** 高层 semantic / language module 负责决定去哪；Nav2 继续负责安全运动执行。
8. **记录 failure，而不只记录 success。** 每个 milestone 都保留可复现实验、日志和 failure category。

## 初期明确不做

- 重写 FAST-LIO2
- M0 阶段替换现有 PGO backend
- 重建 Nav2 / MPPI
- 把 D435 depth 设为主 semantic pipeline 的强制依赖
- 从头训练大型 VLM
- M3 / M4 稳定前进入 long-horizon VLN
- 提前创建大量不会立刻使用的 ROS package
- 在没有复现和实机验证前声称某篇 paper 已经集成

## 计划中的仓库结构

只在对应 milestone 真正开始实现时创建 package。

~~~text
real-world-rgb-lidar-navigation/
├── README.md
├── README.zh-CN.md
├── docs/
│   ├── architecture.md
│   ├── platform_interface.md
│   ├── calibration.md
│   ├── references.md
│   └── benchmark_protocol.md
├── src/
│   ├── camera_bringup/          # M1
│   ├── rgb_lidar_grounding/     # M2
│   ├── object_localization/     # M2
│   ├── semantic_goal_server/    # M3
│   └── semantic_memory/         # M4
├── launch/
├── config/
├── tests/
└── benchmarks/
~~~

## Reference 总表

| Module | 第一版 reference |
|---|---|
| MID360 ROS | [Livox ROS Driver 2](https://github.com/Livox-SDK/livox_ros_driver2) |
| LiDAR-Inertial Odometry | [FAST_LIO](https://github.com/hku-mars/FAST_LIO) |
| metric navigation | [Navigation2](https://github.com/ros-navigation/navigation2) + 现有 robot platform |
| D435 ROS | [realsense-ros](https://github.com/realsenseai/realsense-ros) |
| Camera-LiDAR calibration | [FAST-Calib](https://github.com/hku-mars/FAST-Calib) |
| open-vocabulary detection | [NanoOWL](https://github.com/NVIDIA-AI-IOT/nanoowl) |
| object mask | [NanoSAM](https://github.com/NVIDIA-AI-IOT/nanosam) |
| 2D→3D semantic lifting concept | ConceptFusion |
| semantic navigation | [VLMaps](https://github.com/vlmaps/vlmaps) + Nav2 |
| persistent object memory | [ConceptGraphs](https://github.com/concept-graphs/concept-graphs) |
| VLN | M5 前冻结一个具体 reference |

## 最终成功标准

~~~text
RViz metric goal
      ↓
visible semantic object goal
      ↓
remembered object goal
      ↓
multi-step language instruction
~~~

最终成果不是某一个模型，而是一套能够把：

**真实传感器 → 语义感知 → 3D grounding → 空间记忆 → 语言推理 → 安全机器人导航**

真正连接起来的可复现实机 Robotics Software 系统。
