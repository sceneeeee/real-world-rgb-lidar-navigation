# Real-World RGB-LiDAR Navigation

[English](README.md) | [简体中文](README.zh-CN.md)

一个面向真实移动机器人的 ROS 2 项目，目标是把 **RGB 语义感知、Livox MID360 主几何信息、可选的 RGB-D 深度辅助信号、持续空间记忆** 和最终的 **语言引导导航 / VLN** 接到同一套真实机器人系统中。

> **当前状态：项目初始化与平台熟悉阶段。**  
> 下方 Roadmap 描述的是目标系统；只有在真实机器人上完成集成和验证后，某项能力才算真正实现。

## 为什么做这个项目

这个项目延续此前在 VLN、Habitat 仿真和 LiDAR-aware navigation 上的工作，但重点开始从“仿真中的导航算法”转向“真实机器人上的完整 Robotics Software”。

目标不是重做 SLAM 或 Nav2，而是在已有真实机器人自主导航平台上增加缺失的语义和语言层：

```text
让机器人看见物体
      ↓
知道物体在真实空间哪里
      ↓
能够导航到语义目标
      ↓
能够记住曾经看过的环境
      ↓
最终执行更长的自然语言导航指令
```

因此，**VLN 并没有被放弃**。它被移动到了更合理的高层：建立在稳定的真实感知、定位和导航之上。

## 与现有机器人仓库的边界

目标机器人平台由以下仓库维护：

**[sceneeeee/XJTLU-autonomous-vehicle-rtk](https://github.com/sceneeeee/XJTLU-autonomous-vehicle-rtk)**

该仓库已经提供本项目依赖的底层自主导航能力，包括：

- NVIDIA Jetson Orin NX、Ubuntu 22.04、ROS 2 Humble
- Livox MID360 LiDAR 与 IMU 支持
- FAST-LIO2 局部里程计
- PGO 全局修正
- `map -> odom -> base_link` TF 链
- 点云到导航 costmap 的转换
- Nav2 + MPPI 导航与控制
- STM32 下位机串口控制
- 已有真实机器人室内导航与避障流程
- runtime logging 与 rosbag 调试链

**本仓库不重复实现以上功能。**

## 系统架构

```text
┌─────────────────────────────────────────────────────┐
│ real-world-rgb-lidar-navigation                     │
│                                                     │
│  RGB-D Camera                                       │
│   ├─ RGB ──> Visual / Open-Vocabulary Perception    │
│   └─ Depth ─> Auxiliary baseline / validation       │
│                         │                           │
│  RGB-LiDAR 3D Grounding  ←  MID360 point cloud      │
│                         │                           │
│        (MID360 = primary geometry source)            │
│      ↓                                              │
│  3D Object Localization                             │
│      ↓                                              │
│  Persistent Semantic / Spatial Memory               │
│      ↓                                              │
│  Language Reasoning / VLN                           │
│      ↓                                              │
│  Semantic Goal / Waypoint Generation                │
└──────────────────────────┬──────────────────────────┘
                           │ Nav2 goal / waypoint
                           ▼
┌─────────────────────────────────────────────────────┐
│ XJTLU-autonomous-vehicle-rtk                        │
│                                                     │
│  MID360 + IMU                                       │
│      ↓                                              │
│  FAST-LIO2 + PGO                                    │
│      ↓                                              │
│  Nav2 + MPPI                                        │
│      ↓                                              │
│  STM32 + chassis                                    │
│      ↓                                              │
│  Real Robot                                         │
└─────────────────────────────────────────────────────┘
```

职责边界：

- **原平台：** 负责让机器人定位、规划、避障并真实到达一个 metric goal。
- **本仓库：** 根据 RGB、LiDAR 主几何、可选 camera depth、空间记忆和语言决定“机器人应该去哪里”。

## Roadmap

| Milestone | 工程目标 | 做完以后机器人能做什么 |
|---|---|---|
| **M0 — Platform Onboarding** | 跑通并理解现有实机栈 | RViz 点目标，机器人自主导航过去 |
| **M1 — RGB / RGB-D Camera Integration** | 接入实验室现有 RGB-D Camera：RGB、可选 depth、内参、时间戳、TF 和 Camera-LiDAR 外参 | 获得与 MID360 对齐的实时 RGB，并保留对齐 depth 用于调试 / baseline |
| **M2 — RGB-LiDAR 3D Grounding** | 主线使用 RGB 语义 + MID360 几何，并与 RGB-D、可选 RGB-D+LiDAR 方案对比 | 看到 chair 后得到真实 3D / map 坐标，同时具备传感器消融对比 |
| **M3 — Semantic Navigation** | 把语义目标转换为安全 Nav2 approach pose | 输入 `go to the chair`，机器人真实走到椅子附近 |
| **M4 — Persistent Spatial Memory** | 将多次观测融合为持续的 object-level map | 即使目标已经不在视野里，也能回到之前看到的位置 |
| **M5 — Language-Guided Navigation / VLN** | 用语言推理 + 空间记忆选择连续语义 waypoint | 执行“出门、经过沙发、左转、停在打印机旁”一类指令 |
| **M6 — Deployment & Simulation Extension** | 选择性做 Jetson 优化与必要的仿真 | TensorRT / profiling，以及有明确价值时的 Isaac Sim 对应系统 |

### 当前执行规则

项目固定按照：

**M0 → M1 → M2 → M3**

先在真实机器人上跑通。

在 M3 稳定之前，M4/M5 不作为主要开发重点。

这样可以避免项目再次变成“仿真里有 VLN、真实机器人却没有可靠 perception/navigation interface”的状态。

## 各阶段具体做什么

### M1 — RGB / RGB-D Camera Integration

实验室已有 RGB-D Camera，因此硬件上优先直接使用 RGB-D，而不是额外购买或安装 RGB-only Camera。但要明确：**camera depth 不是本项目的硬依赖。**

传感器职责固定为：

- **RGB：** 主要语义 / 视觉输入。
- **Livox MID360：** 3D grounding、mapping 和 navigation 的主要几何来源。
- **RGB-D Depth：** 调试、快速 baseline、交叉验证以及后续消融 / 融合实验的辅助信号。

```text
RGB-D camera
   ├─> RGB image + camera_info ──> 必需的语义输入
   └─> aligned depth image ──────> 可选辅助输入
                    │
                    ↓
             camera intrinsics
                    ↓
       camera ↔ LiDAR extrinsic calibration
                    ↓
          tf2 对齐到 base_link / map
```

M1 的核心验收仍然是 RGB 与 MID360 在时间和空间上能够共同使用。如果相机提供 depth，则同时发布并完成基本 sanity check，但后续 semantic navigation 不能依赖 camera depth 才能运行。

### M2 — RGB-LiDAR 3D Grounding

**项目主线仍然是 RGB + MID360：**

```text
RGB image ──> object / open-vocabulary detector ──> 2D box or mask
                                                       │
MID360 point cloud ──> 投影到 camera ───────────────────┘
                                                       ↓
                                              关联目标点云
                                                       ↓
                                               滤波 / 聚类
                                                       ↓
                                             map-frame object
```

RGB-D Camera 同时提供两个有价值的对照路径：

```text
A. RGB-D baseline
   RGB detection + aligned camera depth
        └─> object 3D position

B. RGB + MID360                 [主线]
   RGB detection + LiDAR projection
        └─> object 3D position

C. RGB-D + MID360               [可选]
   camera depth + LiDAR geometry
        └─> cross-check / fusion
```

后续可以比较 3D localization error、robustness、effective range、latency 和 semantic navigation success rate，而不改变项目主目标。一个自然的研究问题是：**当移动机器人已经搭载 3D LiDAR 时，额外的 dedicated depth stream 是否仍然必要？**

目标检测模型目前**不锁死**。后续根据真实机器人上的实时性、开放词汇能力以及 Jetson 部署成本比较后再确定。

### M3 — Semantic Navigation

```text
semantic target
      ↓
3D object position
      ↓
safe approach-pose generation
      ↓
Nav2 NavigateToPose
      ↓
原有 MPPI / localization / chassis stack
```

第一阶段先把 ObjectNav / Semantic Navigation 做稳定，再进入真正长程语言导航。

### M4 — Persistent Spatial Memory

机器人需要记住已经离开当前视野的物体。

一个语义记忆节点可以包含：

```text
object id
semantic label
visual embedding
map-frame position
confidence
observation count
last-seen timestamp
representative observation(s)
```

这一阶段会把当前已有的 embedding / semantic work 与真实机器人感知重新连接起来。

### M5 — VLN

计划采用**分层 VLN 架构**，而不是让 VLM 直接高频控制电机：

```text
language instruction
        ↓
VLM / language reasoning
        ↓
persistent semantic-spatial memory
        ↓
high-level semantic waypoint
        ↓
Nav2 execution
```

项目初始化阶段**不锁死具体 VLM / VLN 模型**。

目前 HiCo-Nav 一类 hierarchical memory / reasoning 结构可以作为架构参考；ETPNav / 3DFF 保留为 route-VLN research baseline；NaVIDA 可以作为 direct-action comparison。这里描述的是后续候选方向，不表示它们已经集成到本仓库。

## 设计原则

1. **Real robot first。** 仿真用于降低风险和提高可复现性，但不能替代实机验证。
2. **不重造底层平台。** FAST-LIO2、PGO、Nav2、MPPI 和 chassis control 继续作为外部依赖。
3. **模块接口稳定优先。** detector、fusion、memory、VLM 都应该可以替换。
4. **先测量，再优化。** 每个 milestone 都需要可重复的实机测试、日志和 failure cases。
5. **路线锁死，模型不锁死。** detector、VLM 或 memory representation 可以变化，但项目目标不随论文变化。
6. **Research 服务于工程问题。** 新 VLN / spatial intelligence 方法只有在解决真实 failure 或带来可测能力时才进入主线。
7. **Depth 是选项，不是拐杖。** RGB-D depth 可以加速 baseline 和调试，但 RGB + MID360 必须能够独立完成主要 semantic grounding。

## 初期明确不做

- 重写 FAST-LIO2 或研究新 SLAM 算法
- 重建 Nav2 或底盘控制栈
- 从头训练大型 VLM
- 在没有明确需求前把 3D Gaussian Splatting 设成项目必选项
- 把 RGB-D camera depth 设成 semantic navigation 的强制依赖
- 在 M3 尚未稳定前把完整 VLN research 作为主开发任务
- 在没有实机验证前声称系统已经实现 end-to-end autonomy

## 计划中的仓库结构

仓库只随着实际 milestone 增长，不提前创建大量空 package。

```text
real-world-rgb-lidar-navigation/
├── README.md
├── README.zh-CN.md
├── docs/
│   ├── architecture.md
│   ├── platform_interface.md
│   ├── calibration.md
│   └── benchmark_protocol.md
├── src/
│   ├── camera_bringup/          # M1
│   ├── rgb_lidar_fusion/        # M2
│   ├── object_localization/     # M2
│   ├── semantic_goal_server/    # M3
│   └── semantic_map/            # M4
├── launch/
├── config/
├── tests/
└── benchmarks/
```

## 最终成功标准

整个项目希望沿着以下能力逐步推进：

```text
RViz metric goal
      ↓
visible semantic object goal
      ↓
remembered object goal
      ↓
multi-step language instruction
```

最终成果不是某一个模型，而是一套能够把：

**真实传感器 → 语义感知 → 空间记忆 → 语言推理 → 安全机器人导航**

真正连接起来的可复现 Robotics Software 系统。
