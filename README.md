# RoboMaster EP Six-Object Sorting

English | [简体中文](README.zh-CN.md)

## Demo

Real-robot run: sorting 3 tennis balls + 3 water bottles (sped up).

https://github.com/user-attachments/assets/731c6d1a-fc6a-4c20-8b2b-e82314770e40

Full-length original (3:25): [photos/result.mp4](photos/result.mp4)

## Original Task Requirements

### 2. Task Simplification and System Interfaces

Set up 4–6 fixed pick-up grid cells on the table. The vision system determines whether each cell contains an object and its class; the arm calls a fixed-point grasp routine by cell number. Grasping at arbitrary positions is not required.

* Image interface: `sensor_msgs/Image`.
* Detection interface: `vision_msgs/Detection2DArray`, containing at least class, bounding box and confidence.
* Grasp interface: ROS 2 Action or an equivalent long-running task interface with feedback.
* Task control: BehaviorTree or state machine.

### 3. Simulation Tasks

1. Build the arm, gripper, top-down camera, table grid and at least 2 object classes.
2. The simulated camera publishes images; the detection node publishes object class, bounding box and confidence.
3. Determine the object's grid cell from the bounding-box center, and choose the bin by class.
4. Call the grasp action to pick, place by class and return.
5. Loop over the objects on the table until all are handled or no executable target remains.
6. Handle exceptions such as empty cells, unrecognized objects, unreachable targets and grasp failures.
7. Start the camera, detection, arm and task-control nodes from a single launch file.

### 4. Simulation Acceptance Criteria

* Automatically detect and sort at least 2 object classes.
* Test at least 6 objects and correctly sort at least 5.
* No manual specification of object class, grasp cell or placement area after the task starts.
* No obvious collisions or joint-limit violations during the whole run.
* Correctly handle at least 2 kinds of exceptions, and save detection, grasp and task-state logs.
* One launch file starts the complete simulation system.

### 5. Real-Robot Validation Tasks

1. Fix a top-down RGB camera; mark pick-up cells and at least 2 sorting areas on a real table.
2. Prepare 6 target objects across at least 2 classes.
3. Use the detection model trained in earlier experiments to recognize object class and grid position.
4. Use the real-robot grasp routine to put each class into its bin.
5. Skip empty cells and unrecognized objects; on grasp failure or an unreachable target, return to a safe pose or stop.
6. Save the detected class, grasp result, placement result and exception info for every object.

### 6. Real-Robot Acceptance Criteria

* Automatically sort at least 2 object classes.
* Correctly sort at least 5 of 6 objects.
* No manual class judgment or action triggering during a normal run.
* Correctly handle at least two cases: empty cells and unrecognized objects.
* No collisions, limit violations or dangerous motions; stop safely on exceptions.
* One launch file starts the image, detection, arm and task-control nodes.

### 7. Deliverables

* Complete ROS 2 workspace, launch files and dependency notes.
* Object detection model, class description and table grid configuration.
* BehaviorTree XML or state-machine configuration.
* Simulation and real-robot demo videos.
* Sorting results, exception test records, task logs and a short experiment report.

## Personal Notes

* Core hardware: RoboMaster EP + Jetson Orin NX
* Two object classes: tennis balls + standard 550 ml water bottles

Implementation notes (work in progress):

* 1. The arm's initial pose serves the camera: it keeps the view, detection rate and detection range comfortable.
* 2. Most of the time the arm stays "retracted" with both joints pulled in (including the zero state, and chassis rotation/movement after grabbing an object). It extends only when picking.
* 3. The chassis starts relatively far from the objects so the camera has a good viewpoint.
* Thanks to the RoboMaster EP arm's simple dual-parallel-joint structure, a necessary condition for "extending" to grasp is that the object is centered in the camera image. The open question is how to tell whether the object is within reach without depth information from the camera.
* Tennis balls and bottles differ in height, but two fixed gripper heights (one per class) are enough. Since vision already identifies the class, choosing the height is not a problem.

* Grasp trigger: raw infrared reading ≤ 20 mm, three times in a row. Measured close-range readings do not equal ruler distance.
* The servo near the gripper never moves. The chassis-side servo extends forward to its farthest position by default. It retracts fully only after grabbing a target, stays retracted until the chassis turns to the placement angle, then extends to place.
* The gripper closes only while grasping; it stays open otherwise.
* Combining the above gives the initial state: extended, gripper open. The program returns to it at start and at the end. (If it gets stuck on an error mid-run it does not need to return, but it switches the body LEDs to red.)

## Infrared Pick-and-Place Test Demo

On Ubuntu/Jetson, run `./scripts/run_ir_demo_ubuntu.sh` to preview the configuration; by default it does not connect to the robot.
A single run grasps when the raw infrared reading is ≤ 20 mm, retracts to carry, turns right 90° to place, then returns to the extended, gripper-open initial state.
The stock official `models/yolo26m.pt` is included in the repo. YOLO only writes its own log and does not take part in action decisions.
Before running on the real robot, confirm the servo mapping and extend/retract angles in the config; see the
[demo guide](docs/ir_pick_place_demo.md).

Measure infrared distance with `bash scripts/run_ir_distance_ubuntu.sh --seconds 8 --label tennis-30mm --reference-mm 30`.
This only records distance feedback and sends no motion commands; place the target at the ruler-measured distance first, then run it. See the
[infrared measurement guide](docs/ir_distance_measurement.md).

## YOLO Turn-and-Approach Pick-and-Place Demo (single tennis ball or bottle)

When the object is off to the left or right, the chassis turns at most 5° at a time based on the YOLO box center, then re-detects and corrects.
After three consecutive centered frames, if the infrared reading is above the threshold it approaches at 4 cm/s in ~1 cm steps, stopping to re-detect each time; by default it advances at most 20 cm.
It grasps after three consecutive infrared readings meet the raw 20 mm threshold (this is not an actual 2 cm). On invalid feedback or a lost target it stops and lights red.
Gripper-side servo ID2 is fixed at 1073; chassis-side ID1 moves between 600 and 1190. After grasping and retracting, it backs up to the starting center using odometry,
then places the ball 90° to the right of the start heading; `--target bottle` tracks only bottles and places them 90° to the left.
Actual yaw feedback compensates the alignment turn. Uses the stock `yolo26m.pt`, continuous background frame capture and 540p video streaming.
Both the tennis ball and the bottle have completed alignment, approach, grasp, reversing back, class-direction placement and initial-state recovery on the real robot, confirmed on site.

```bash
bash scripts/run_yolo_align_demo_ubuntu.sh --observe
bash scripts/run_yolo_align_demo_ubuntu.sh --execute
bash scripts/run_yolo_align_demo_ubuntu.sh --target bottle --observe
bash scripts/run_yolo_align_demo_ubuntu.sh --target bottle --execute
```

Run the script without arguments for preview only. See the [new demo guide](docs/yolo_align_pick_place_demo.md).

## Six-Object Sorting

A ROS 2 workspace is in `ros2_ws/`: image topic, YOLO detection topic, sorting Action, task client, launch file and a YAML state machine that actually drives the flow transitions. See the [ROS 2 workspace guide](ros2_ws/README.md) for building and running. The original script entry points are kept for comparison with the verified real-robot motions.

The program tracks six directions as pending, picked or not found, and each time picks the pending direction with the smallest turn from the current actual heading.
It reuses demo 2 to search for balls and bottles. After grasping it backs up to the starting center; balls go to the right area and bottles to the left, each area with three fixed placement angles.
After placing, it opens the gripper and retracts, turns straight to the next search angle, then extends. Left/right scanning likewise retracts before turning and extends afterward. At the end of a full round it retracts, re-centers, then restores the initial state.
Grasp and the six placement angles are configured in `config/object_sorting.json`. The 2026-09-19 full-round run log records that all 3 balls and 3 bottles completed placement and the program ended normally; the actual placement positions still need on-site checking.

```bash
bash scripts/run_sorting_ubuntu.sh
bash scripts/run_sorting_ubuntu.sh --observe --observe-frames 3
bash scripts/run_sorting_ubuntu.sh --execute
```

See [sorting flow and angle configuration](docs/object_sorting.md).

Current user-designated stable tag: `stable-2026-09-18`. For parameters and verification scope see the [stable baseline](docs/stable_version.md).

> The linked docs under `docs/` and `ros2_ws/` are currently in Chinese.
