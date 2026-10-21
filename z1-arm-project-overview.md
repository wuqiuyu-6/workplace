---
name: z1-arm-project-overview
description: 宇树 Unitree Z1 六自由度机械臂项目的三仓库构成与整体控制链路
metadata: 
  node_type: memory
  type: project
  originSessionId: ca3d4080-7c55-4834-90a7-faf122652d74
  modified: 2026-09-29T15:38:01.243Z
---

`/home/wuqiuyu/robot_arm/` 下是 Unitree Z1 六自由度机械臂（6 关节 + 1 夹爪，共 7 电机）的完整代码栈，含三个 git 仓库（项目原在 `/home/wuqiuyu/桌面/robot_arm`，2026-09-29 已迁移至此）：

- **z1_controller** — 运行在机械臂板载电脑（aarch64）上的**实时控制器**（官方仓库），是核心。见 [[z1-controller]]。
- **z1_sdk** — 用户侧**控制库/API**（官方仓库），通过 UDP 向控制器发高层指令。见 [[z1-sdk]]。
- **z1_ros** — ROS(noetic) 驱动，内含一个**不兼容官方的 z1_controller 分支** + MoveIt/ros_control 集成。见 [[z1-ros-driver]]。

控制链路（官方，不用 ROS）：
```
用户代码(z1_sdk 的 unitreeArm) --UDP--> z1_controller(实时FSM) --UDP--> 下位机MCU(192.168.123.110:8881) --> 电机
```

ROS 链路（用 MoveIt）：
```
MoveIt --> z1_hw(ros_control) --> z1_controller(ROS分支) --UDP--> MCU --> 电机
```

关键差异：官方 z1_controller + z1_sdk 是一套；z1_ros 里的 z1_controller/z1_sdk 是**另一套独立实现**，两者 UDP 结构/端口不同、不能混用。
