# Real-World RGB-D + LiDAR Object Navigation

[English](README.md) | [简体中文](README.zh-CN.md)

**以 Gervet 等人的《Navigating to Objects in the Real World》（Science Robotics，2023）及其 SemExp 算法谱系为依据，开展模块化物体目标导航的硬件适配复现。**

> **状态 — 2026-10-06：** 方案与平台接入阶段。本仓库目前记录实施计划；本项目尚未完成相机集成、模型及权重运行、实机导航验收。仓库地址保留为 `real-world-rgb-lidar-navigation`，标题反映更新后的传感器分工。

## 1. 固定范围：只实现一条路线

**D435 的 RGB-D 负责看见物体并估计物体位置；MID360 为现有机器人提供定位与导航几何信息，FAST-LIO2/PGO 和 Nav2/MPPI 负责利用这些信息完成定位、规划和运动。**

| 组件 | 选定职责 |
|---|---|
| RealSense D435 RGB | 通过 Mask R-CNN 完成物体实例分割 |
| RealSense D435 depth | 将图像观测提升到三维，并更新语义任务地图 |
| MID360 + IMU、FAST-LIO2 + PGO | 估计机器人位姿，维护平台的几何参考 |
| 现有 LiDAR 代价地图 + Nav2/MPPI | 检查运动目标可达性、规划路径并执行运动 |
| SemExp 学习式策略 | 尚未找到目标类别时，选择下一步搜索位置 |

深度是感知链路的必需输入，**不再是辅助对照分支**。不实现 RGB-only 目标定位、不通过 LiDAR 投影关联图像中的物体点，也不开发 depth–LiDAR 物体定位融合。深度估计失败时，不得悄悄切换到其他几何来源。

初期任务是**类别级 ObjectNav**：寻找指定类别的一个物体实例，并停在其附近。它不等同于任意自然语言理解、特定实例重识别或长指令 VLN。

## 2. 主论文与选择理由

**主要实机研究参考：** Theophile Gervet、Soumith Chintala、Dhruv Batra、Jitendra Malik、Devendra Singh Chaplot，*Navigating to Objects in the Real World*，**Science Robotics 8(79)，eadf6991，2023**。[正式发表](https://doi.org/10.1126/scirobotics.adf6991) | [论文手稿](https://arxiv.org/abs/2212.00922) | [作者项目主页](https://theophilegervet.github.io/projects/real-world-object-navigation/)

**算法来源：** Chaplot 等人，*Object Goal Navigation using Goal-Oriented Semantic Exploration*（**SemExp**，NeurIPS 2020）。[会议论文集](https://proceedings.neurips.cc/paper/2020/hash/2c75cf2681788adaca63aa95ae028b22-Abstract.html) | [官方代码与权重说明](https://github.com/devendrachaplot/Object-Goal-Navigation)

两篇论文属于**同一算法谱系，不是两条竞争实现**。我们选择 2023 年研究中的学习式模块化方法，不复现其比较研究中的全部方法。

选择依据是任务和硬件匹配：该研究使用 RGB-D 观测及 LiDAR 位姿信息开展真实环境实验，具有明确的语义地图接口，并能追溯到学习式探索代码。论文作者单位包括 CMU、Meta AI Research、Georgia Tech 和 UC Berkeley。发表渠道与团队经历支持其可信度，但不意味着该方法是最新或在所有条件下最优。

选型时也检查了 [HOV-SG（RSS 2024）](https://www.roboticsproceedings.org/rss20/p077.html) 和 [VLFM（ICRA 2024）](https://doi.org/10.1109/ICRA57147.2024.10610712)。前者在真实机器人上结合 RGB-D 与 3D LiDAR，实现分层语言目标导航；后者利用视觉语言模型为探索前沿赋值，进行零样本搜索。它们仅作为选型背景，不新增实施分支：HOV-SG 侧重分层场景图，VLFM 则改变了本项目选定的探索算法。初期继续沿用 Gervet/SemExp。

## 3. 复现边界：必须说明的差异

**本项目不是硬件完全相同的重复实验，也不是对原论文全部对比实验的复现。**

| 方面 | 原研究 | 本项目的单一实现 |
|---|---|---|
| 机器人与相机 | Stretch + D435i | 现有移动机器人 + D435 |
| LiDAR 用途 | 位姿估计与碰撞检测；第 4.1 节明确不用于建图 | MID360 位姿估计，**同时提供导航代价地图** |
| 局部运动执行 | FMM 规划器与离散动作 | 现有 Nav2/MPPI，加目标适配器 |
| 学习组件 | 面向目标的语义探索 | 保留 SemExp 策略及其输入和类别约定 |
| 记忆范围 | 单次任务内的语义地图 | 单次任务内的语义地图，不做跨天数据库 |
| 评估 | 原住宅、任务与动作预算 | 在现有机器人上另行记录测试协议 |

LiDAR 代价地图和 Nav2 改变了运动执行可用的信息、机器人运动以及失败模式。因此，它们是**需要显式记录的重要适配**，不是可以忽略的实现细节。原论文成功率不是本项目的承诺，也不能与不同平台和协议下的结果直接比较。

一套系统中保留两种不同职责的地图：RGB-D 任务地图保留 SemExp 的语义、障碍、已探索区域和机器人位置表示；LiDAR 导航代价地图供 Nav2 使用。**不能直接用 LiDAR 地图通道替换预训练策略的输入，却仍宣称算法未改变。**

## 4. 目标系统架构

```text
D435 RGB -> Mask R-CNN -> masks -----+
D435 aligned depth -----------------+-> RGB-D semantic task map
Timestamped robot pose -------------+              |
                                                  v
                                  Requested category already mapped?
                                      | yes                 | no
                                      v                     v
                               Target region        Learned SemExp goal
                                      +----------+----------+
                                                 v
                                    Coordinate / goal adapter
                                                 |
MID360 + IMU -> FAST-LIO2 + PGO -> pose            |
MID360 -> existing navigation costmaps ----------+-> Nav2 + MPPI
                                                        |
                                                     Robot
                                                        |
                                 new observation -> map update
```

坐标与目标适配器、ROS 接口是本仓库的工程工作。它们不以新的探索规划器或语言模型替代论文的学习式搜索策略。

## 5. M0–M4：单一路线、明确参考与验收

| 阶段 | 实现与来源 | 为什么选择 | 必需证据 |
|---|---|---|---|
| **M0 — 平台验收** | [现有机器人栈](https://github.com/sceneeeee/XJTLU-autonomous-vehicle-rtk)，以及其 Livox 驱动、FAST-LIO2、PGO、Nav2 | 复用维护中的平台，不重做定位和控制 | 实际部署版本与环境清单、TF/action 检查、有人监督的导航及停止测试、日志 |
| **M1 — RGB-D 接入** | [RealSense 官方 ROS wrapper](https://github.com/realsenseai/realsense-ros)、相机安装变换、[ROS 2 tf2](https://docs.ros.org/en/humble/Tutorials/Intermediate/Tf2/Introduction-To-Tf2.html) | 使用经过标定的 RGB-D 观测，无须运行时相机–LiDAR 点关联 | 有效 RGB/depth/CameraInfo、深度单位、配准图像、带时间戳的相机位姿及安装几何检查 |
| **M2 — 语义建图** | SemExp 的 Mask R-CNN 前端与 RGB-D 语义建图模块 | 保留论文表示及预训练模型接口 | 掩码和深度产生的目标观测在地图中位置一致，无效观测被拒绝 |
| **M3 — 导航到已发现物体** | 目标类别地图区域 → 自研目标适配器 → [Nav2](https://docs.nav2.org/commander_api/index.html) | 将语义决策与现有运动执行分离 | 有效且可达的接近位姿、正确目标确认、最终停止位置测量 |
| **M4 — 利用语义记忆搜索** | SemExp 学习式目标导向策略、多次观测、单次任务地图 | 复现寻找初始不可见目标，而不止演示可见目标接近 | 学习策略输出日志、未知目标搜索、发现、接近、停止和任务重置 |

在协调平台权限期间，可以进行 M1 准备和离线代码检查。实物集成需先获得许可；M3/M4 必须建立在 M0 验收及可靠 RGB-D 输入之上。

### M0：平台文档报告不等于本项目已经验收

平台 README 报告了室内 FAST-LIO2 + PGO + Nav2/MPPI 栈和 `indoor-nav` 启动入口。构建、升级或切换分支之前，先核实机器人实际安装情况。GitHub 默认分支不能证明实车部署版本。

未经平台负责人同意、没有现场操作人员或尚未验证停止与接管措施时，不发送电机命令，不开展自主行驶测试。共享导航参数与固件不属于本仓库的修改范围。

### M1：使用 RGB-D 仍然需要标定与同步检查

使用 RGB、与所选彩色图像几何配准的深度，以及对应内参。核实深度单位、图像缩放或裁剪、坐标约定和传感器时钟。RealSense 配准不等于零时间误差，也不会消除遮挡和无效深度。

所需外参是经过测量验证的相机相对真实底盘坐标系的安装位姿，其中包含相机光学坐标系变换。在观测时间 `t`：

```text
T_map_camera(t) = T_map_base(t) * T_base_camera
p_map = T_map_camera(t) * p_camera
```

这里的 `camera` 指深度投影使用的光学坐标系。位姿查询采用图像采集时间，而不是模型推理完成时间。虽然不再需要在线 LiDAR→RGB 投影，但外参验证仍然必需。TF 缺失或深度不可靠时，不生成有效语义导航目标。

### M2：保留原前端及模型输入约定

使用 **COCO 预训练的 Mask R-CNN R50-FPN**，对应官方 [`agents/utils/semantic_prediction.py`](https://github.com/devendrachaplot/Object-Goal-Navigation/blob/5d76902fe9be821926a1de32557ca9a8dc21d0f5/agents/utils/semantic_prediction.py)。选定配置为 `mask_rcnn_R_50_FPN_3x.yaml`，上游权重引用为 `model_final_f10217.pkl`，实现使用 [Detectron2](https://github.com/facebookresearch/detectron2)。

将 [`Semantic_Mapping`](https://github.com/devendrachaplot/Object-Goal-Navigation/blob/5d76902fe9be821926a1de32557ca9a8dc21d0f5/model.py) 和官方预处理适配为不依赖 ROS 的核心模块，再增加薄 ROS 2 接口。保留类别顺序、地图分辨率、坐标约定、局部/全局地图构造以及权重张量维度。六个任务目标类别不等于全部语义通道。

初期目标类别沿用论文与代码：**chair、couch、potted plant、bed、toilet、tv**，即椅子、沙发、盆栽、床、马桶和电视。只在允许的测试区域选择实际可用类别，并报告测试子集。`door` 和 `printer` 不属于初期学习策略目标；增加它们不是简单地改一句文本提示。

不能将无效深度当成自由空间，也不能将投影得到的表面采样点称为精确物体中心。分割与深度不一致时应记录失败，而不是随意生成坐标。

### M3：把语义区域接到安全运动目标

通过显式的任务坐标系→ROS map 变换转换策略栅格坐标。在目标占据区域之外选择接近位姿，检查机器人完整轮廓、路径、目标时效性与当前定位。物体中心默认不是有效停车位。

采用有界运动段、持续更新的观测及明确的重规划规则。学习策略指向未观测区域时，可能需要朝该区域选取可达中间子目标；必须记录这项适配行为，而不是悄悄换成另一个探索策略。`NavigateToPose` 成功不等于 ObjectNav 成功：还需要确认请求类别和停止条件。

### M4：学习式探索，不用前沿策略替代

使用官方 [`Goal_Oriented_Semantic_Policy` / `RL_Policy`](https://github.com/devendrachaplot/Object-Goal-Navigation/blob/5d76902fe9be821926a1de32557ca9a8dc21d0f5/model.py) 以及公开的 `sem_exp.pth` 权重获取路径。初期不重新训练策略，也不并行开发其他探索算法。

一次任务中保留语义地图，独立测试之间重置。不预加载目标坐标。搜索测试开始时不加载已有语义/探索地图；任何已有定位或几何地图信息都必须披露。地图或位姿发生不连续变化时，当前目标失效，应停止并重置受影响任务，直至实现可处理修正的地图重放机制。

必须完成的闭环是：**观测 → 更新语义地图 → 选择学习式搜索目标 → 运动 → 再观测 → 发现并接近目标**。只会到达已经可见的物体，代表完成 M3，不代表完成 M4。

## 6. 代码来源与第一项实施检查

**已检查的 SemExp 源码版本：** `5d76902fe9be821926a1de32557ca9a8dc21d0f5`。这是参考源码的标识，不是部署已通过测试的证明。

2023 年论文作者主页链接到 [HomeRobot](https://github.com/facebookresearch/home-robot)。但其当前 [`ObjectNavAgentModule`](https://github.com/facebookresearch/home-robot/blob/main/src/home_robot/home_robot/agent/objectnav_agent/objectnav_agent_module.py) 创建的是 `ObjectNavFrontierExplorationPolicy`。运行该默认配置，**不能证明复现了本项目选定的学习式 SemExp 策略**。HomeRobot 仅作为接口组织参考，不是另一条实施路线，也不替代目标策略的权重来源。

在开发完整 ROS 链路之前：

1. 获取官方策略与分割权重，记录来源与哈希。**本次文档修订没有验证权重能否实际下载或成功加载。**
2. 在隔离开发环境中加载并完成推理/预处理冒烟测试；核对完整类别及通道映射，区分仿真真值输入与真实传感器输入。
3. 确定一套兼容的部署环境，测量内存、观测到目标的延迟，以及与导航栈共同运行的资源竞争。SemExp 仓库记录的是 Habitat 0.1.5 / PyTorch 1.6 / CUDA 10.2 等旧依赖，并非现成的 ROS 2 Humble 或 Jetson 软件包。不得把这套旧环境直接覆盖安装到共享机器人上。

权重缺失或运行环境不兼容属于需要解决的复现阻塞，不是静默换策略的理由。工作站上的推理检查属于同一套系统的准备，不是第二条研究路线。车载实时性能仍是待验证目标。

## 7. 只验证一套算法，不并行比较多方案

用**固定的一套实现**执行预先定义的室内许可任务。记录起点、目标类别、初始是否可见、地图初始化、标定版本、权重哈希、软件版本、执行预算以及全部失败尝试。

| 证据 | 测量内容 |
|---|---|
| RGB-D/TF 有效性 | 时间戳一致性、有效深度、投影/坐标正确性、拒绝事件 |
| 语义建图 | 目标地图位置及重复观测一致性 |
| 导航 | 正确类别到达、最终距离/接近区域有效性、耗时与路径长度 |
| 搜索 | 初始不可见目标的成功率、策略输出、目标拒绝及重规划记录 |
| 运行与安全 | 端到端延迟、资源占用、过期数据、人工干预、停止事件和失败原因 |

测试前确定保守的实体停止与净空规则，不把原论文的碰撞次数预算照搬为实机操作规则。安全状态不确定时停止测试。

Nav2 目标调用次数不等于原论文的离散动作步数。应明确本项目的时间/距离预算。只有能够独立、合理地确定通往有效目标区域的最短可行路径时才报告 SPL；否则报告实际路径长度与耗时，不将其误称为 SPL。不同协议下不能声称优于原论文，也不能宣称复现了原论文的成功率数字。

## 8. 延后扩展与开发规则

**M5 不属于初期复现：** 后续可以基于已验证的 ObjectNav，增加 [VLMaps 启发的语言→导航接口](https://github.com/vlmaps/vlmaps)。它不是当前依赖，也不代表已经复现 VLMaps、R2R 或 RxR 的结果。

M0–M4 不包含：任意开放词汇搜索、跨天物体数据库、多楼层导航、操作抓取、端到端电机策略、传感器来源消融、多套检测前端或 Isaac Sim 迁移。

在工作站开发，与平台维护者协商后再部署到共享 Jetson。现有机器人底层栈与本仓库保持分离。只在真正实现时创建软件包；本项目目前没有可直接运行的启动命令。

Codex 每次会话应读取并更新本地 `temp_report.md`，记录进度、测试、阻塞和下一步。该文件必须排除在 Git 之外，不得提交。凭据、模型权重、rosbag、非公开图像和运行日志也不得提交。适配上游代码时保留许可证与署名。

**初期完成标准：** 一条有文档和实测记录的 RGB-D 语义建图与学习式搜索链路，能够寻找初始不可见的受支持类别目标，并使用现有 LiDAR/Nav2 平台到达经独立验证的停止区域。
