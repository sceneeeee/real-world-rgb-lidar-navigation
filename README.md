# Real-World RGB-LiDAR Navigation

[English](README.md) | [简体中文](README.zh-CN.md)

A real-world ROS 2 mobile-robot project that combines **RGB semantics**, **Livox MID360 geometry**, optional **RGB-D depth as an auxiliary signal**, **persistent spatial memory**, and eventually **language-guided navigation / VLN**.

> **Project status:** initialization and platform onboarding.  
> The roadmap below defines the intended system. Features are only considered implemented after they are integrated and validated on the real robot.

## Why This Project

This project extends my previous work in Vision-and-Language Navigation (VLN), Habitat simulation, and LiDAR-aware navigation toward a real robotic system.

The goal is **not** to replace the existing mobile-robot autonomy stack or to reimplement SLAM and Nav2. Instead, this repository adds the missing semantic and language layer on top of a working real-robot platform:

```text
see objects
    ↓
locate them in 3D
    ↓
navigate to semantic targets
    ↓
remember the environment
    ↓
follow longer language instructions
```

VLN is therefore **not abandoned**. It is moved to a higher-level module that operates on top of reliable real-world perception, localization, and navigation.

## Platform Boundary

The target robot platform is maintained separately in:

**[sceneeeee/XJTLU-autonomous-vehicle-rtk](https://github.com/sceneeeee/XJTLU-autonomous-vehicle-rtk)**

That repository already provides the low-level autonomy infrastructure used by this project:

- NVIDIA Jetson Orin NX with Ubuntu 22.04 and ROS 2 Humble
- Livox MID360 LiDAR and IMU support
- FAST-LIO2 local odometry
- PGO-based global correction
- `map -> odom -> base_link` TF chain
- point-cloud conversion for navigation costmaps
- Nav2 navigation with MPPI control
- serial bridge to the STM32 lower controller
- real-robot indoor navigation and obstacle-avoidance workflows
- runtime logging and rosbag-based debugging

This repository does **not** duplicate those components.

## System Architecture

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

The intended software boundary is simple:

- **Existing platform:** make the robot localize, plan, avoid obstacles, and physically reach a metric goal.
- **This repository:** decide *what* the robot should navigate to from RGB, LiDAR geometry, optional camera depth, spatial memory, and language.

## Roadmap

| Milestone | Engineering goal | Demonstrable result |
|---|---|---|
| **M0 — Platform Onboarding** | Run and understand the existing real-robot stack end to end | Click a goal in RViz and have the robot navigate to it autonomously |
| **M1 — RGB / RGB-D Camera Integration** | Integrate the available RGB-D camera: RGB stream, optional depth stream, intrinsics, timestamps, TF, and camera-LiDAR extrinsics | Live RGB aligned with MID360 geometry, with aligned camera depth available for debugging / baselines |
| **M2 — RGB-LiDAR 3D Grounding** | Use RGB semantics + MID360 geometry as the primary grounding path; compare against RGB-D and optional RGB-D+LiDAR variants | See a chair and estimate its 3D / map-frame position, with sensor-source ablations available |
| **M3 — Semantic Navigation** | Convert a semantic target into a safe Nav2 approach pose | `go to the chair` causes the real robot to navigate to the chair |
| **M4 — Persistent Spatial Memory** | Fuse repeated observations into a persistent object-level map | The robot can navigate back to an object that is no longer visible |
| **M5 — Language-Guided Navigation / VLN** | Use language reasoning plus memory to select sequential semantic waypoints | Follow instructions such as “exit the room, pass the sofa, turn left, and stop near the printer” |
| **M6 — Deployment & Simulation Extension** | Optimize selected AI modules for Jetson and add simulation where useful | TensorRT / profiling results and, if justified, an Isaac Sim counterpart |

### Current execution rule

The project is intentionally staged:

**M0 → M1 → M2 → M3 must work on the real robot before M4/M5 becomes the main development focus.**

This prevents the project from turning into another simulation-only VLN stack before the real perception and navigation interfaces are reliable.

## Planned Technical Direction

### M1 — RGB / RGB-D camera integration

The available lab RGB-D camera is preferred over buying or adding a separate RGB-only camera. However, **camera depth is not a hard dependency of the project**.

Sensor roles are intentionally separated:

- **RGB:** primary semantic / visual input.
- **Livox MID360:** primary geometry source for grounding, mapping, and navigation.
- **RGB-D depth:** auxiliary signal for debugging, fast baselines, cross-checking, and later ablation / fusion experiments.

Expected interfaces:

```text
RGB-D camera
   ├─> RGB image + camera_info ──> required semantic input
   └─> aligned depth image ──────> optional auxiliary input
                    │
                    ↓
          camera intrinsics
                    ↓
       camera ↔ LiDAR extrinsic calibration
                    ↓
       tf2 alignment with base_link / map
```

M1 is complete when the RGB stream is temporally and spatially usable together with MID360. If the camera provides depth, that depth stream should also be exposed and sanity-checked, but semantic navigation must not depend on it.

### M2 — RGB-LiDAR 3D grounding

The **primary research / system path** remains RGB + MID360:

```text
RGB image ──> object / open-vocabulary detector ──> 2D box or mask
                                                       │
MID360 point cloud ──> projection into camera ─────────┘
                                                       ↓
                                            associated 3D points
                                                       ↓
                                           filtered object position
                                                       ↓
                                                map-frame object
```

The RGB-D camera also enables two useful comparison paths:

```text
A. RGB-D baseline
   RGB detection + aligned camera depth
        └─> object 3D position

B. RGB + MID360                 [primary]
   RGB detection + LiDAR projection
        └─> object 3D position

C. RGB-D + MID360               [optional]
   camera depth + LiDAR geometry
        └─> cross-check / fusion
```

This makes it possible to compare localization error, robustness, range, latency, and navigation success without changing the core project goal. A useful later research question is whether a dedicated depth stream remains necessary when the mobile robot already carries a 3D LiDAR.

The detector is intentionally **not fixed yet**. Candidates can be compared later based on real-time performance, open-vocabulary capability, and Jetson deployment cost.

### M3 — semantic navigation

```text
semantic target
      ↓
3D object position
      ↓
safe approach-pose generation
      ↓
Nav2 NavigateToPose
      ↓
existing MPPI / localization / chassis stack
```

The first target is deliberately simple: reliable object-goal navigation before introducing long-horizon language reasoning.

### M4 — persistent spatial memory

The robot should accumulate object observations over time instead of relying only on the current camera frame.

A semantic-memory entry may contain:

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

This stage connects the current embedding / semantic work with real-world robot perception.

### M5 — VLN

The planned VLN architecture is **hierarchical**, not direct low-level motor control:

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

The exact VLM or VLN algorithm is **not locked at project initialization**.

Current architectural references include hierarchical memory/reasoning approaches such as HiCo-Nav, while ETPNav / 3DFF-style systems remain relevant research baselines and NaVIDA can serve as a direct-action comparison. These are references, not claims that their code is already integrated here.

## Design Principles

1. **Real robot first.** Simulation is used when it reduces risk or improves reproducibility, not as a substitute for hardware validation.
2. **Do not rebuild the platform.** FAST-LIO2, PGO, Nav2, MPPI, chassis control, and other stable platform functions remain external dependencies.
3. **Stable interfaces over tightly coupled algorithms.** Perception, grounding, memory, language reasoning, and navigation should be replaceable modules.
4. **Measure before optimizing.** Each milestone should have repeatable real-world tests, logs, and failure cases.
5. **Do not lock models prematurely.** The project goal stays fixed even if the detector, VLM, memory representation, or deployment backend changes.
6. **Keep research and engineering connected.** New VLN or spatial-intelligence methods should enter the system only when they solve a demonstrated failure or enable a measurable capability.
7. **Use depth as an option, not a crutch.** RGB-D depth may accelerate baselines and debugging, but the main semantic grounding path must remain valid with RGB + MID360 alone.

## Initial Non-Goals

The following are intentionally out of scope for the early milestones:

- reimplementing FAST-LIO2 or developing a new SLAM algorithm
- rebuilding Nav2 or the existing robot chassis stack
- training a large VLM from scratch
- making 3D Gaussian Splatting a requirement before a concrete need is demonstrated
- making RGB-D camera depth mandatory for semantic navigation
- starting full VLN research before M3 semantic navigation is stable
- claiming end-to-end autonomy before real-robot validation exists

## Planned Repository Structure

The repository will grow with the milestones rather than creating empty packages in advance.

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

Only packages that are actually being implemented should be added.

## Success Criteria

The project is considered successful when it progresses from metric navigation to semantic and language-guided navigation on the real robot:

```text
RViz goal
   ↓
visible object goal
   ↓
remembered object goal
   ↓
multi-step language instruction
```

The final value of the project is not a single model. It is a reproducible robotics software stack that connects **real sensors → semantic perception → spatial memory → language reasoning → safe robot navigation**.
