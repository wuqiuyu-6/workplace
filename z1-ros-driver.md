---
name: z1-ros-driver
description: z1_ros 的包构成、官方不兼容性、pinocchio v2.7.0 依赖与启动方式
metadata: 
  node_type: memory
  type: project
  originSessionId: ca3d4080-7c55-4834-90a7-faf122652d74
  modified: 2026-09-22T06:30:55.708Z
---

`/home/wuqiuyu/桌面/robot_arm/z1_ros` — ROS noetic 驱动（catkin 工作区，建议放 `~/z1_ws/src`）。属 [[z1-arm-project-overview]]。

包构成（README.md 有图）：
- **z1_bringup** — 启动：`real_arm.launch` / `sim_arm.launch`（MoveIt + 仿真/真机）、`real_ctrl.launch` / `sim_ctrl.launch`（UDP 直控）。
- **z1_controller** — **分支版控制器，与官方 z1_controller/z1_sdk 不兼容**。用 pinocchio 解析 `config/z1.urdf`，ROS 参数（`udp/mcu_ip`、`port_to_mcu=8881`、`port_to_sdk=8871`、`collision/*`、`cmdCache`），状态为 Init/Passive/JointCtrl/Teach/TeachRepeat/Calibration/LowCmd/ClearError。链接 `lib/libUnitreeArm_{arch}.so`。
- **z1_hw** — ros_control 硬件接口，桥接 MoveIt→机械臂，提供 `joint_trajectory_controller` + 夹爪 action server。
- **z1_moveit_config** — MoveIt 示例配置。
- **z1_rviz** — RViz 启动/配置（view_model / view_robot）。
- **z1_examples** — `move_group_interface.cpp`、`ros_gripper_ctrl.cpp`（夹爪角度，0=闭合，负=张开）。
- **z1_sdk** — UDP 直控示例（`example_joint_ctrl`、`example_lowcmd`，含 C++ 与 py）。
- **unitree_ros** — git submodule（本机未 checkout，目录为空）。

关键约束/坑：
- 依赖 **pinocchio 必须 ≤ v2.7.0**（`git checkout v2.7.0`），v3.0.0 起不兼容；需装 pybind11（Python 接口）、liburdfdom-dev；ROS 端还需 moveit、joint-trajectory-controller、trac-ik。
- 端口：到 MCU `8881`、到 SDK `8871`（与官方 8071/8072 不同）。
- 夹爪语义：goal topic 的角度 **0=闭合，负值=张开**。
- 启动真机：`roslaunch z1_bringup real_arm.launch rviz:=true`；仿真：`sim_arm.launch UnitreeGripperYN:=true rviz:=true`。
