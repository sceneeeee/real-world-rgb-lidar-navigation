# Real-World RGB-D + LiDAR Object Navigation

[English](README.md) | [简体中文](README.zh-CN.md)

**A hardware-adapted reproduction of modular Object Goal Navigation, based on Gervet et al., _Navigating to Objects in the Real World_ (Science Robotics, 2023), and its SemExp algorithm lineage.**

> **Status - 2026-10-06:** proposal and platform onboarding. This repository currently documents the plan; camera integration, model/checkpoint execution, and real-robot navigation have not been validated by this project. The repository URL remains `real-world-rgb-lidar-navigation`; the title describes the updated sensor roles.

## 1. Fixed scope: one implementation

**D435 RGB-D detects objects and estimates where they are. MID360 supplies the existing robot localization and navigation geometry; FAST-LIO2/PGO and Nav2/MPPI turn that geometry into localization, planning, and motion.**

| Component | Selected responsibility |
|---|---|
| RealSense D435 RGB | Object instance segmentation with Mask R-CNN |
| RealSense D435 depth | Lift image observations into 3D and update the semantic task map |
| MID360 + IMU, FAST-LIO2 + PGO | Estimate the robot pose and maintain the platform's geometric reference |
| Existing LiDAR-derived costmaps + Nav2/MPPI | Check reachable motion goals, plan paths, and execute motion |
| SemExp learned policy | Select where to search when the requested category has not been found |

Depth is a required perception input, **not an auxiliary comparison branch**. There is no RGB-only grounding implementation, no LiDAR-to-image object-point association, and no depth-LiDAR object-localization fusion. A failed depth estimate must not silently switch to another geometry source.

The initial task is **category-level ObjectNav**: search for an instance of a requested category, then stop near it. It is not yet unrestricted language understanding, instance re-identification, or long-instruction VLN.

## 2. Primary paper and why it was selected

**Main experimental reference:** Theophile Gervet, Soumith Chintala, Dhruv Batra, Jitendra Malik, and Devendra Singh Chaplot. _Navigating to Objects in the Real World_. **Science Robotics 8(79), eadf6991, 2023.** [Publisher](https://doi.org/10.1126/scirobotics.adf6991) | [Manuscript](https://arxiv.org/abs/2212.00922) | [Author project page](https://theophilegervet.github.io/projects/real-world-object-navigation/)

**Algorithm source:** Chaplot et al. _Object Goal Navigation using Goal-Oriented Semantic Exploration_ (**SemExp**, NeurIPS 2020). [Proceedings](https://proceedings.neurips.cc/paper/2020/hash/2c75cf2681788adaca63aa95ae028b22-Abstract.html) | [Official code and checkpoint instructions](https://github.com/devendrachaplot/Object-Goal-Navigation)

These are **one algorithm lineage, not two competing implementations**. We select the learned modular approach studied in the 2023 paper, not all methods in its comparative study.

The selection is based on task and hardware fit: a real-world study with RGB-D observations and LiDAR-based pose sensing, an explicit semantic-map interface, and a traceable learned exploration implementation. The paper's author affiliations include CMU, Meta AI Research, Georgia Tech, and UC Berkeley. Publication and team provenance support its credibility; they do not establish that the method is the newest or universally best option.

Related work reviewed during selection includes [HOV-SG, RSS 2024](https://www.roboticsproceedings.org/rss20/p077.html), whose real robot combines RGB-D and 3D LiDAR for hierarchical language-grounded navigation, and [VLFM, ICRA 2024](https://doi.org/10.1109/ICRA57147.2024.10610712), which uses vision-language frontier values for zero-shot search. Neither is an additional implementation track: the first emphasizes hierarchical scene graphs, while the second changes the selected exploration algorithm. We retain Gervet/SemExp for the initial reproduction.

## 3. Reproduction boundary: important differences

**This is not a hardware-identical replication or a reproduction of the entire paper's comparative study.**

| Aspect | Original study | This project's single implementation |
|---|---|---|
| Robot and camera | Stretch with D435i | Existing mobile robot with D435 |
| LiDAR use | Pose estimation and collision detection; explicitly not mapping in Sec. 4.1 | MID360-based pose **and navigation costmaps** |
| Local execution | FMM planner and discrete actions | Existing Nav2/MPPI with a goal adapter |
| Learned component | Goal-oriented semantic exploration | Preserve the SemExp policy and its input/category contract |
| Memory scope | Episode-local semantic map | Episode-local semantic map; no cross-day database |
| Evaluation | Original homes, episodes, and action budgets | Separately documented tests on the available robot |

LiDAR costmaps and Nav2 change the information available to execution, robot motion, and failure modes. They are **substantive, documented adaptations**, not an invisible implementation detail. The original paper's success rate is not a promised result or a directly comparable score for this robot.

Two map products serve different roles within this one system: the RGB-D task map preserves SemExp's semantic, obstacle, explored-area, and agent-location representation; the LiDAR navigation costmaps serve Nav2. **Do not replace the pretrained policy's map channels with LiDAR channels without explicitly treating that as an algorithm change.**

## 4. Target architecture

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

The adapter and ROS integration are this repository's engineering work. They do not replace the paper's learned search policy with a new planner or a language model.

## 5. M0-M4: one route, explicit references and deliverables

| Stage | Implementation and source | Why this choice | Required evidence |
|---|---|---|---|
| **M0 - Platform validation** | [Existing robot stack](https://github.com/sceneeeee/XJTLU-autonomous-vehicle-rtk), its Livox driver, FAST-LIO2, PGO, and Nav2 | Reuse the maintained robot platform rather than rebuild localization/control | Deployed revision and environment inventory, TF/action checks, supervised navigation and stop tests, saved logs |
| **M1 - RGB-D integration** | [Official RealSense ROS wrapper](https://github.com/realsenseai/realsense-ros), camera mounting transform, [ROS 2 tf2](https://docs.ros.org/en/humble/Tutorials/Intermediate/Tf2/Introduction-To-Tf2.html) | Use calibrated RGB-D observations without runtime camera-LiDAR point association | Valid RGB/depth/CameraInfo, depth scale, aligned images, timestamped camera pose, verified mounting geometry |
| **M2 - Semantic mapping** | SemExp's Mask R-CNN frontend and RGB-D semantic mapper | Preserve the paper's representation and pretrained-model interface | Masks and depth-based object observations appear at consistent map locations; invalid observations are rejected |
| **M3 - Navigate to a found object** | Target-category map region -> project goal adapter -> [Nav2](https://docs.nav2.org/commander_api/index.html) | Keep semantic decisions separate from existing motion execution | A valid, reachable approach pose; correct target confirmation and measured final stop |
| **M4 - Search using semantic memory** | SemExp learned goal-oriented policy, repeated observations, episode-local map | Reproduce search for an initially unseen target, not only a visible-object demo | Logged learned-policy goals, unknown-target search, discovery, approach, stop, and episode reset |

M1 preparation and offline code checks may proceed while platform access is arranged. Physical integration requires permission; M3/M4 require a validated M0 and reliable RGB-D inputs.

### M0: platform facts are not yet our results

The platform README reports an indoor FAST-LIO2 + PGO + Nav2/MPPI stack and an `indoor-nav` launch entry. Verify what is actually installed before building, updating, or switching branches. A GitHub default branch is not proof of the robot's deployed revision.

No motor commands or autonomous tests without the platform owner's permission, an on-site operator, and verified stop/override controls. Shared navigation parameters and firmware remain outside this repository's scope.

### M1: RGB-D still needs calibration and synchronization

Use RGB, depth aligned to the chosen color image geometry, and the corresponding intrinsics. Verify depth units, image resizing/cropping, frame conventions, and the sensor clock. RealSense alignment does not guarantee zero timing error or remove occlusions and invalid depth.

The required transform is the measured camera mounting pose relative to the actual robot base, including the camera's optical-frame transform. At observation time `t`:

```text
T_map_camera(t) = T_map_base(t) * T_base_camera
p_map = T_map_camera(t) * p_camera
```

Here `camera` denotes the optical frame used by the depth projection. Lookup the pose at the image timestamp, not at the end of model inference. No online LiDAR-to-RGB projection is required, but extrinsic verification is still mandatory. Missing TF or unreliable depth means no valid semantic navigation goal.

### M2: preserve the original frontend and model contract

Use **Mask R-CNN R50-FPN, COCO pretrained**, following the official [`agents/utils/semantic_prediction.py`](https://github.com/devendrachaplot/Object-Goal-Navigation/blob/5d76902fe9be821926a1de32557ca9a8dc21d0f5/agents/utils/semantic_prediction.py). The selected config is `mask_rcnn_R_50_FPN_3x.yaml`; the upstream weight reference is `model_final_f10217.pkl`. The implementation uses [Detectron2](https://github.com/facebookresearch/detectron2).

Adapt [`Semantic_Mapping`](https://github.com/devendrachaplot/Object-Goal-Navigation/blob/5d76902fe9be821926a1de32557ca9a8dc21d0f5/model.py) and the official preprocessing into a ROS-independent core with a thin ROS 2 adapter. Preserve category ordering, map resolution, coordinate conventions, local/global-map construction, and checkpoint tensor dimensions. The six task categories are not the entire semantic-channel set.

Supported initial goal categories follow the paper/code: **chair, couch, potted plant, bed, toilet, tv**. Use available categories in permitted test areas and report the subset. `door` and `printer` are not initial learned-policy goals; adding them is not merely a text-prompt change.

Do not treat invalid depth as free space, or a projected surface sample as an exact object center. Segmentation/depth inconsistency must be recorded rather than converted into an arbitrary goal.

### M3: bridge a semantic region to a safe motion goal

Convert policy-grid coordinates through an explicit episode-frame-to-ROS-map transform. Select an approach pose outside the target's occupied region; validate the full robot footprint, path, goal freshness, and current localization. An object centroid is not a valid parking location by default.

Use bounded motion segments with fresh observations and documented replanning rules. A policy goal in an unobserved area may need a reachable intermediate subgoal toward that region. Record this adapter behavior; do not quietly substitute a separate exploration policy. `NavigateToPose` success alone does not prove ObjectNav success: verify the requested category and stopping condition.

### M4: learned exploration, not a frontier substitute

Use the official [`Goal_Oriented_Semantic_Policy` / `RL_Policy`](https://github.com/devendrachaplot/Object-Goal-Navigation/blob/5d76902fe9be821926a1de32557ca9a8dc21d0f5/model.py) and the published `sem_exp.pth` checkpoint route. No policy retraining or parallel exploration methods are planned for the initial implementation.

Keep the task map during an episode and reset it between independent tests. Do not preload target coordinates. Start search trials without a preloaded semantic/exploration map; any pre-existing localization or metric-map information must be disclosed. Map/pose discontinuities invalidate ongoing goals: stop and reset the affected episode until correction-aware map replay is implemented.

The required progression is **observe -> update semantic map -> select learned search goal -> move -> observe again -> find and approach target**. A system that only reaches an already visible object has completed M3, not M4.

## 6. Code provenance and the first implementation gate

**Inspected SemExp source revision:** `5d76902fe9be821926a1de32557ca9a8dc21d0f5`. This identifies the reference code, not a successfully tested deployment.

The 2023 author page links to [HomeRobot](https://github.com/facebookresearch/home-robot). However, its current [`ObjectNavAgentModule`](https://github.com/facebookresearch/home-robot/blob/main/src/home_robot/home_robot/agent/objectnav_agent/objectnav_agent_module.py) constructs `ObjectNavFrontierExplorationPolicy`. Running that default is **not** sufficient evidence of reproducing the selected learned SemExp policy. HomeRobot is an integration reference, not a second implementation track or a substitute checkpoint source.

Before developing the full ROS pipeline:

1. Obtain the official policy and segmentation weights; record download origin and hashes. **Weight download/loadability has not been verified in this documentation update.**
2. Load them in an isolated development environment and run inference/preprocessing smoke tests. Check the complete category/channel mapping and distinguish ground-truth simulator inputs from real sensor inputs.
3. Establish one compatible deployment environment and measure memory, observation-to-goal latency, and contention with the navigation stack. The SemExp repository documents legacy Habitat 0.1.5 / PyTorch 1.6 / CUDA 10.2 dependencies; it is not a ready-made ROS 2 Humble or Jetson package. Do not install that legacy stack over the shared robot environment.

A missing checkpoint or incompatible runtime is a reproduction blocker to resolve, not permission to silently run a different policy. Development-machine inference checks are preparation for the same system, not a second research route. Onboard real-time performance is an unverified deployment target.

## 7. Validation without parallel algorithms

Evaluate **one fixed implementation** on a predefined set of permitted indoor tasks. Record start pose, goal category, whether the target was initially visible, map initialization, calibration version, checkpoint hashes, software revisions, execution budget, and all failed attempts.

| Evidence | What is measured |
|---|---|
| RGB-D/TF validity | Timestamp consistency, valid depth, projection/frame correctness, rejection events |
| Semantic mapping | Target-map alignment and repeated-observation consistency |
| Navigation | Correct-category arrival, final distance/approach-region validity, elapsed time and path length |
| Search | Initially unseen-target success; policy goals and rejection/replanning history |
| Runtime and safety | End-to-end latency, resource use, stale data, interventions, stop events and failure causes |

Specify a conservative physical stopping/clearance rule before testing. Do not copy the original study's collision allowance into real-robot operations. Stop tests when safety is uncertain.

Nav2 goal calls are not the original paper's discrete action steps. Use an explicit local time/distance budget. Report SPL only when an independently justified shortest feasible path to the valid goal region is available; otherwise report path length and time without calling them SPL. Do not claim superiority or reproduction of the published percentage from a different task protocol.

## 8. Deferred extension and development rules

**M5 is outside the initial reproduction:** a [VLMaps-inspired language-to-navigation interface](https://github.com/vlmaps/vlmaps) may later build on the validated ObjectNav system. It is not a current dependency or a claim of reproducing VLMaps, R2R, or RxR results.

Not in M0-M4: arbitrary open-vocabulary search, cross-day persistent object databases, multi-floor navigation, manipulation, end-to-end motor policies, sensor-source ablations, alternative detector pipelines, or an Isaac Sim migration.

Develop on the workstation; deploy to the shared Jetson only by agreement with its maintainer. Keep the maintained robot stack separate from this repository. Add packages only when implemented; there is no runnable launch command for this project yet.

For Codex sessions, read and update local `temp_report.md` with progress, tests, blockers, and next actions. Keep it excluded from Git and never commit it. Do not commit credentials, model weights, rosbags, private imagery, or runtime logs. Preserve upstream licenses and attribution when code is adapted.

**Initial completion means:** one documented RGB-D semantic-mapping and learned-search pipeline can find an initially unseen supported object and navigate to an independently verified stopping region using the existing LiDAR/Nav2 platform.
