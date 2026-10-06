# Real-World RGB-LiDAR Navigation

[English](README.md) | [简体中文](README.zh-CN.md)

A real-world ROS 2 mobile-robot project that adds **open-vocabulary visual perception, RGB-LiDAR 3D grounding, semantic navigation, persistent object memory, and eventually VLN** on top of an existing MID360 + Nav2 robot platform.

> **Project status:** proposal / platform onboarding.  
> A component is considered implemented only after it has been integrated and validated on the real robot.

## Project Goal

The existing robot can already localize, plan, avoid obstacles, and navigate to a metric pose. This project adds the missing semantic layer:

~~~text
metric goal
    ↓
visible object goal
    ↓
remembered object goal
    ↓
language-guided navigation
~~~

The project is intentionally hierarchical. VLN does not replace SLAM, Nav2, or low-level control. It is introduced later as a high-level decision layer after real-world perception and semantic navigation are stable.

## Frozen Hardware and Sensor Decision

The initial sensor configuration is fixed:

~~~text
Jetson Orin NX
├── Intel RealSense D435
│   ├── RGB    → primary semantic / visual input
│   └── Depth  → baseline, validation, ablation, optional fusion
│
└── Livox MID360
    └── 3D geometry → primary geometry source for grounding and navigation
~~~

The D435 replaces the need for a separate RGB-only camera unless a later hardware limitation is demonstrated.

### Main path versus depth baseline

The **main semantic-grounding path** is:

~~~text
D435 RGB + MID360
~~~

The D435 depth stream remains enabled and recordable, but the main semantic-navigation pipeline must still work when camera depth is disabled.

Three grounding variants will be evaluated:

| Variant | Geometry used for semantic 3D grounding | Role |
|---|---|---|
| **A — RGB-D** | D435 aligned depth | baseline |
| **B — RGB + MID360** | MID360 projected into the RGB image | **main method** |
| **C — RGB-D + MID360** | camera depth + LiDAR geometry | optional fusion / upper-bound comparison |

MID360 remains active in the existing FAST-LIO2 / Nav2 platform for all variants. The comparison above concerns the **geometry source used to lift semantic observations into 3D**, not the underlying robot localization stack.

This creates a concrete project question:

> **When a mobile robot already carries a 3D LiDAR, how much additional value does a dedicated RGB-D depth stream provide for real-world semantic grounding and navigation?**

## Platform Boundary

The robot platform is maintained separately in:

**[sceneeeee/XJTLU-autonomous-vehicle-rtk](https://github.com/sceneeeee/XJTLU-autonomous-vehicle-rtk)**

The existing platform already provides:

- NVIDIA Jetson Orin NX, Ubuntu 22.04, ROS 2 Humble
- Livox MID360 + IMU
- FAST-LIO2 local odometry
- PGO global correction
- map → odom → base_link TF chain
- Nav2 + MPPI
- STM32 / chassis interface
- indoor real-robot navigation
- runtime logging and rosbag debugging

This repository does **not** reimplement those components.

## Target System Architecture

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

Later:
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

Each milestone has a concrete engineering goal and an initial paper, repository, or existing-platform reference. A reference indicates what will be reused or adapted; it does **not** mean the upstream method is already integrated.

| Milestone | Goal | Primary references / repositories | Demonstrable result |
|---|---|---|---|
| **M0 — Platform Onboarding** | Validate the existing metric-navigation stack | existing robot repo, [Livox ROS Driver 2](https://github.com/Livox-SDK/livox_ros_driver2), [FAST_LIO](https://github.com/hku-mars/FAST_LIO), [Navigation2](https://github.com/ros-navigation/navigation2) | RViz goal → autonomous real-robot navigation |
| **M1 — D435 + MID360 Integration** | Bring up D435 and calibrate camera ↔ LiDAR | [realsense-ros](https://github.com/realsenseai/realsense-ros), [FAST-Calib](https://github.com/hku-mars/FAST-Calib), ROS 2 tf2 | stable RGB/depth + calibrated MID360 projection into RGB |
| **M2 — RGB-LiDAR 3D Grounding** | Detect an object and estimate its map-frame 3D position | [NanoOWL](https://github.com/NVIDIA-AI-IOT/nanoowl), [NanoSAM](https://github.com/NVIDIA-AI-IOT/nanosam), ConceptFusion-style 2D→3D semantic lifting | chair detected → 3D / map-frame position |
| **M3 — Semantic Navigation** | Convert an object target into a safe Nav2 approach pose | [VLMaps](https://github.com/vlmaps/vlmaps) as semantic-navigation reference, [Navigation2](https://github.com/ros-navigation/navigation2) for execution | go to the chair → robot reaches a safe pose near the chair |
| **M4 — Persistent Spatial Memory** | Maintain object-level memory across observations | [ConceptGraphs](https://github.com/concept-graphs/concept-graphs) as object-memory / scene-graph reference | navigate back to a previously observed object |
| **M5 — VLN** | Add language reasoning over the persistent semantic map | **one concrete VLN paper + repository will be selected and frozen before M5 starts** | execute multi-step language navigation |
| **M6 — Deployment / Simulation Extension** | Optimize selected modules and add simulation only when useful | TensorRT / Jetson profiling; Isaac Sim only if justified | deployment benchmarks and reproducible extension |

### Execution rule

~~~text
M0 → M1 → M2 → M3 → M4 → M5
~~~

M0–M3 must work on the real robot before long-horizon VLN becomes a main development task.

---

## M0 — Validate the Existing Real-Robot Baseline

### Purpose

M0 introduces no new perception algorithm. It establishes that the existing robot platform is a trustworthy dependency.

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

### References

- **Robot source of truth:** [sceneeeee/XJTLU-autonomous-vehicle-rtk](https://github.com/sceneeeee/XJTLU-autonomous-vehicle-rtk)
- **LiDAR ROS interface:** [Livox-SDK/livox_ros_driver2](https://github.com/Livox-SDK/livox_ros_driver2)
- **LiDAR-inertial odometry:** [hku-mars/FAST_LIO](https://github.com/hku-mars/FAST_LIO)
- **Navigation:** [ros-navigation/navigation2](https://github.com/ros-navigation/navigation2)
- **PGO:** reuse the implementation already integrated in the robot repository; do not replace it during M0.

### Acceptance criteria

~~~text
[ ] /livox/lidar is stable
[ ] FAST-LIO2 registered cloud / odometry is available
[ ] PGO / global correction is available
[ ] map → odom → base_link is valid
[ ] NavigateToPose is online
[ ] RViz fixed frame is map
[ ] a short real-world click-to-go test succeeds
[ ] logs are preserved for the run
~~~

---

## M1 — D435 + MID360 Integration and Calibration

### 1. D435 bringup

Use the lab **Intel RealSense D435**.

Initial implementation reference:

- [realsenseai/realsense-ros](https://github.com/realsenseai/realsense-ros)

Required outputs:

~~~text
D435
├── RGB image
├── RGB camera_info
├── depth image
├── aligned depth
└── camera TF frames
~~~

RGB is required by the main pipeline. Depth is enabled and recorded for baseline / validation experiments.

### 2. Camera ↔ LiDAR extrinsic calibration

Initial calibration reference:

- [hku-mars/FAST-Calib](https://github.com/hku-mars/FAST-Calib)

Target transform:

~~~text
T_camera_lidar
~~~

The resulting extrinsics are integrated into ROS 2 TF so that observations can be transformed consistently between:

~~~text
camera frame ↔ MID360 frame ↔ base_link ↔ map
~~~

### 3. M1 validation

~~~text
[ ] RGB stream is stable
[ ] depth stream is stable
[ ] camera intrinsics are valid
[ ] timestamps are usable with MID360
[ ] camera ↔ MID360 extrinsics are available
[ ] MID360 points project correctly into the RGB image
[ ] aligned D435 depth is available for later baselines
~~~

---

## M2 — Open-Vocabulary Perception and RGB-LiDAR 3D Grounding

M2 is split into two explicit problems: **find the semantic target in image space**, then **lift that observation into 3D**.

### M2.1 Open-vocabulary perception

The initial default is frozen as:

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

References:

- **OWL-ViT / TensorRT deployment:** [NVIDIA-AI-IOT/nanoowl](https://github.com/NVIDIA-AI-IOT/nanoowl)
- **Promptable segmentation on Jetson:** [NVIDIA-AI-IOT/nanosam](https://github.com/NVIDIA-AI-IOT/nanosam)

Initial validation vocabulary:

~~~text
chair
door
sofa
printer
table
~~~

The default detector / segmenter should only be replaced if real Jetson benchmarks demonstrate a concrete accuracy, latency, or deployment failure.

### M2.2 Main path — RGB + MID360

~~~text
object mask
     │
MID360 point cloud
     ↓
transform with calibrated extrinsics
     ↓
project LiDAR points into RGB
     ↓
retain points inside object mask
     ↓
range / outlier filtering
     ↓
3D clustering
     ↓
robust object centroid / extent
     ↓
transform into map frame
~~~

The overall semantic-lifting idea is related to open-set 2D→3D mapping systems such as **ConceptFusion**, while this project uses MID360 geometry rather than depending on camera depth for the primary path.

The projection / filtering module is project-specific engineering built from calibrated camera projection, ROS TF, and standard point-cloud filtering / clustering operations. It is not presented as a new standalone perception model.

### M2.3 Grounding baselines

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

#### B — RGB + MID360 — main method

~~~text
RGB mask
   +
MID360 projection
   ↓
associated LiDAR points
   ↓
3D object position
~~~

#### C — RGB-D + MID360 — optional fusion

~~~text
D435 depth
   +
MID360 geometry
   ↓
cross-check / joint estimate
   ↓
3D object position
~~~

### M2 evaluation

- 3D object-position error
- grounding success rate
- effective range / coverage
- latency
- Jetson CPU / GPU load
- failure cases under occlusion, reflective surfaces, sparse LiDAR support, and poor depth

---

## M3 — Semantic Object Navigation

M3 converts a semantic target into an executable metric goal.

References:

- [VLMaps — Visual Language Maps for Robot Navigation](https://github.com/vlmaps/vlmaps): architectural reference for spatially grounding semantic / language concepts into navigable locations.
- [Navigation2](https://github.com/ros-navigation/navigation2): actual execution framework on the robot.

The project does **not** replace the existing Nav2 stack with VLMaps.

~~~text
semantic target
      ↓
M2 map-frame object position
      ↓
candidate approach poses
      ↓
costmap / collision / reachability checks
      ↓
select safe pose facing target
      ↓
Nav2 NavigateToPose
      ↓
real robot
~~~

The robot navigates to a **safe approach pose**, not directly to the object centroid.

### M3 acceptance target

~~~text
"go to the chair"
        ↓
chair detected / grounded
        ↓
safe approach pose generated
        ↓
NavigateToPose
        ↓
robot reaches the chair area
~~~

Primary metrics:

- semantic-goal success rate
- navigation success rate
- final distance to target
- approach-pose validity
- end-to-end latency
- failure category

---

## M4 — Persistent Object-Level Spatial Memory

M4 removes the requirement that the target must remain visible.

Primary reference:

- [ConceptGraphs](https://github.com/concept-graphs/concept-graphs)

The first implementation borrows the **object-centric persistent memory / scene representation** idea without requiring a full reproduction of the complete ConceptGraphs pipeline.

A memory node may contain:

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

Repeated observations are associated using:

~~~text
spatial consistency
+
semantic similarity
+
visual similarity
~~~

and then fused into a persistent object record.

Example:

~~~text
chair_01    → (2.1, 4.5)
chair_02    → (5.2, 1.8)
printer_01  → (7.4, 3.1)
door_01     → (...)
~~~

This enables:

~~~text
target not currently visible
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

M5 is intentionally downstream of M4.

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

Unlike earlier exploratory plans, M5 will **not** mix several unrelated VLN methods. Before M5 starts, one concrete paper + repository will be selected as the primary reference and the implementation scope will be frozen around that method.

## Experimental Plan

The semantic-grounding comparison is:

~~~text
A: D435 RGB + D435 depth
B: D435 RGB + MID360          [main]
C: D435 RGB + D435 depth + MID360
~~~

| Metric | A: RGB-D | B: RGB+MID360 | C: RGB-D+MID360 |
|---|---:|---:|---:|
| 3D localization error | measure | measure | measure |
| grounding success rate | measure | measure | measure |
| effective range | measure | measure | measure |
| latency | measure | measure | measure |
| Jetson resource usage | measure | measure | measure |

Semantic-navigation metrics:

| Metric | Definition |
|---|---|
| success rate | robot reaches a valid approach region around the requested object |
| final target distance | distance from robot / approach pose to semantic target |
| execution time | command → arrival |
| failure type | perception / grounding / memory / planning / control |

## Design Principles

1. **Real robot first.** Simulation is secondary to real-robot validation.
2. **Do not rebuild the platform.** FAST-LIO2, PGO, Nav2, MPPI, and chassis control remain external dependencies.
3. **Reference every major design decision.** Each milestone starts from a concrete paper, repository, or existing platform implementation.
4. **Freeze the first implementation.** Avoid changing models before a measured failure justifies it.
5. **MID360 is the primary geometry source.** D435 depth is valuable, but the main semantic pipeline must not depend on it.
6. **Keep comparisons interpretable.** RGB-D, RGB+LiDAR, and RGB-D+LiDAR remain distinct grounding variants.
7. **Use hierarchical autonomy.** High-level semantic / language reasoning outputs goals; Nav2 remains responsible for safe motion execution.
8. **Log failures, not only successes.** Every milestone should produce repeatable tests and categorized failure cases.

## Initial Non-Goals

- reimplement FAST-LIO2
- replace the existing PGO backend during M0
- rebuild Nav2 / MPPI
- make D435 depth a mandatory dependency of the main semantic pipeline
- train a large VLM from scratch
- start long-horizon VLN before M3 / M4 are stable
- pre-create large amounts of unused ROS code
- claim a paper method is integrated before it has been reproduced and validated

## Planned Repository Structure

Packages are added only when implementation starts.

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

## Reference Summary

| Module | Initial reference |
|---|---|
| MID360 ROS | [Livox ROS Driver 2](https://github.com/Livox-SDK/livox_ros_driver2) |
| LiDAR-inertial odometry | [FAST_LIO](https://github.com/hku-mars/FAST_LIO) |
| metric navigation | [Navigation2](https://github.com/ros-navigation/navigation2) + existing robot platform |
| D435 ROS | [realsense-ros](https://github.com/realsenseai/realsense-ros) |
| camera-LiDAR calibration | [FAST-Calib](https://github.com/hku-mars/FAST-Calib) |
| open-vocabulary detection | [NanoOWL](https://github.com/NVIDIA-AI-IOT/nanoowl) |
| object masks | [NanoSAM](https://github.com/NVIDIA-AI-IOT/nanosam) |
| 2D→3D semantic lifting concept | ConceptFusion |
| semantic navigation | [VLMaps](https://github.com/vlmaps/vlmaps) + Nav2 |
| persistent object memory | [ConceptGraphs](https://github.com/concept-graphs/concept-graphs) |
| VLN | one concrete reference to be frozen before M5 |

## Success Criterion

~~~text
RViz metric goal
      ↓
visible semantic object goal
      ↓
remembered object goal
      ↓
multi-step language instruction
~~~

The intended outcome is not a single model, but a reproducible real-robot software stack connecting:

**real sensors → semantic perception → 3D grounding → spatial memory → language reasoning → safe navigation**
