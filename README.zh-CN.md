<div align="center">

[English](./README.md) · **简体中文**

# RoboMaster EP 六物品分拣

RoboMaster EP 用 YOLO 识别网球和水瓶，靠红外触发抓取，并按类别分区放置。

[结果](#结果) · [演示](#演示) · [工作流程](#工作流程) · [快速开始](#快速开始) · [环境](#环境)

[![Ubuntu 20.04][ubuntu-shield]][ubuntu-link]
[![ROS 2 Foxy][ros-shield]][ros-link]
[![Python 3.8][python-shield]][python-link]
[![Jetson Orin NX][jetson-shield]][jetson-link]

<img src="assets/hero-sorting.gif" width="480" alt="RoboMaster EP 逐个抓取网球和水瓶，并把每类放到各自的区域">

<sub>真机整轮运行：3 个网球、3 个水瓶分到两个区域（20 倍速）</sub>

</div>

## 结果

| 部分 | 内容 | 关键结果 | 说明 / 证据 |
| :-- | :-- | :-- | :-- |
| ✅&nbsp;六物品分拣 | 真机完整任务 | [6/6 放置（3 球、3 瓶），用时 4 分 10 秒](records/object_sorting/20260919/object-sort-19700101T011245258885Z/run.json)¹ | [流程与角度](docs/object_sorting.md) · [视频](photos/result.mp4) |
| ✅&nbsp;单物品&nbsp;demo | 真机，单个网球或水瓶 | [网球](records/yolo_alignment/20260918/yolo-align-19700101T035651387074Z/run.json)和[水瓶](records/yolo_alignment/20260918/yolo-align-19700101T041506006690Z/run.json)均完成整轮；回位误差 6.33 mm / 5.66 mm | [demo 说明](docs/yolo_align_pick_place_demo.md) · [测试记录](docs/yolo_alignment_hardware_tests.md) |
| 🚧&nbsp;ROS 2 封装 | 节点、Action、状态机、Launch | 尚无存档的真机整轮运行 | [ROS 2 说明](ros2_ws/README.md) |
| 🚧&nbsp;仿真 | Gazebo 中使用同一检测节点 | [放置 1/6，未完成整轮](records/simulation/gazebo-sort-19700101T052904Z/summary.json) | [仿真说明](ros2_ws/src/ep_sorting_sim/README.md) |

<sub>✅ 已完成 · 🚧 进行中</sub>

<sub>¹ “放置”按 `sorting_item_placed` 事件计数，表示机械臂完成了放置动作；日志不记录物品最终落位。Jetson 时钟未同步，运行目录名中的 `1970` 不是实际运行日期。</sub>

## 演示

<table>
  <tr><th>测试布局</th><th>夹爪上的红外传感器</th></tr>
  <tr>
    <td><img src="presentation/assets/sorting-layout.png" width="300" alt="俯视图：机器人位于起始位置，周围有三个网球、三个水瓶和两个放置区"></td>
    <td><img src="presentation/assets/ep-infrared.jpg" width="300" alt="夹爪正面，上方是红外测距传感器，前方有一个网球"></td>
  </tr>
  <tr>
    <td align="center"><sub>六个物品围绕起始位置，每类一个放置区</sub></td>
    <td align="center"><sub>红外原始读数连续三次 ≤ 24 mm 时夹紧</sub></td>
  </tr>
  <tr><th>车载识别</th><th>Gazebo 仿真</th></tr>
  <tr>
    <td><img src="records/object_sorting/20260919/object-sort-19700101T011245258885Z/latest_detections.jpg" width="400" alt="夹爪相机画面：水瓶以 0.95 置信度被识别，位于抓取轴上"></td>
    <td><img src="assets/sim-gazebo.png" width="400" alt="Gazebo 场景：EP 模型、网球和水瓶"></td>
  </tr>
  <tr>
    <td align="center"><sub>6/6 运行中的 <code>yolo26m.pt</code>：水瓶 0.95，位于抓取轴上</sub></td>
    <td align="center"><sub>Gazebo Classic 11 场景；存档运行放置了 6 件中的 1 件</sub></td>
  </tr>
</table>

真机整轮运行（加速）：

https://github.com/user-attachments/assets/731c6d1a-fc6a-4c20-8b2b-e82314770e40

完整原片（3:25）：[photos/result.mp4](photos/result.mp4) · 仿真录屏：[simulation-final.mp4](records/simulation/simulation-final.mp4)

## 工作流程

- **初始姿态**：机械臂外伸、夹爪松开，夹爪相机正视前方。起始位置和朝向是整轮的基准。
- **搜索**：六个方位（0°、±36°、±144°、180°）。每次放置后转向转角最小的未搜索方位，转向时机械臂内收。
- **对准与接近**：底盘每步转 5°，直到 YOLO 检测框位于抓取轴上，再每次前进约 1 cm。相机没有深度信息，由红外判断物品是否够得着。
- **搬运与放置**：内收，沿记录的路径倒回起点，再转向对应类别的区域：球放右侧、瓶放左侧，各三个放置位。
- **异常**：停车并将灯光切为红色，不自动复位。

课程任务要求桌面网格；本项目改为围绕机器人的六个搜索方位（[任务原文](docs/task_requirements.md)）。

## 快速开始

无需机器人，运行单元测试（已在 macOS 上用 Python 3.9 验证）：

```bash
python3 -m pip install numpy opencv-python-headless pyyaml
python3 -m unittest discover -s tests
```

在机器人上，命令取自 [docs/object_sorting.md](docs/object_sorting.md)（编写本 README 时未重新运行）：

```bash
bash scripts/run_sorting_ubuntu.sh
bash scripts/run_sorting_ubuntu.sh --observe --observe-frames 3
bash scripts/run_sorting_ubuntu.sh --execute
```

第一条命令只预览配置，不连接机器人。ROS 2 Launch 用法见 [ros2_ws/README.md](ros2_ws/README.md)。

## 环境

| 设备 | 作用 | 软件 |
| :-- | :-- | :-- |
| RoboMaster EP | 麦轮底盘、双舵机机械臂、夹爪、相机、红外传感器 | `robomaster` SDK |
| Jetson Orin NX | 识别与控制 | Ubuntu 20.04、Python 3.8、ROS 2 Foxy、Gazebo Classic 11、Ultralytics |
| 任意电脑 | 单元测试 | Python 3、numpy、OpenCV、PyYAML |

<sub>Jetson 上的版本取自仓库内的说明文档。</sub>

> [!NOTE]
> 不要在同一个 shell 中同时 source ROS 1 Noetic 和 ROS 2 Foxy。详见 [ros2_ws/README.md](ros2_ws/README.md)。

## 目录结构

```text
.
├── scripts/       机器人程序：红外 demo、YOLO 对准 demo、六物品分拣
├── config/        舵机、红外、方位和放置参数；状态机 YAML
├── ros2_ws/       ROS 2 包：ep_sorting、接口、Gazebo 仿真、EP 模型
├── models/        yolo26m.pt（Ultralytics 官方权重）
├── records/       真机与仿真运行日志
├── docs/          使用说明和实机测试记录
├── tests/         单元测试
├── photos/        真机视频和布置照片
├── presentation/  演示文稿
└── assets/        README 媒体
```

## 致谢

- EP 模型网格来自 [jeguzzi/robomaster_ros](https://github.com/jeguzzi/robomaster_ros)（MIT，许可证保留在 `ros2_ws/src/robomaster_description/`）。
- 检测权重：[Ultralytics `yolo26m.pt` v8.4.0](https://github.com/ultralytics/assets/releases/tag/v8.4.0)，使用 COCO 类别 32（sports ball）和 39（bottle）。

<sub>以 [MIT 许可证](LICENSE)发布。</sub>

<!-- badge definitions: copy this block verbatim into every translation -->
[ubuntu-shield]: https://img.shields.io/badge/Ubuntu-20.04-E95420?logo=ubuntu&logoColor=white
[ubuntu-link]: https://releases.ubuntu.com/focal/
[ros-shield]: https://img.shields.io/badge/ROS%202-Foxy-22314E?logo=ros&logoColor=white
[ros-link]: https://docs.ros.org/en/foxy/
[python-shield]: https://img.shields.io/badge/Python-3.8-3776AB?logo=python&logoColor=white
[python-link]: https://www.python.org/downloads/release/python-380/
[jetson-shield]: https://img.shields.io/badge/Jetson-Orin%20NX-76B900?logo=nvidia&logoColor=white
[jetson-link]: https://developer.nvidia.com/embedded/jetson-orin
