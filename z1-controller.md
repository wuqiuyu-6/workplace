---
name: z1-controller
description: 官方 z1_controller 实时控制器的 FSM 状态、实时性与通信参数
metadata: 
  node_type: memory
  type: project
  originSessionId: ca3d4080-7c55-4834-90a7-faf122652d74
  modified: 2026-09-22T06:30:46.306Z
---

`/home/wuqiuyu/桌面/robot_arm/z1_controller` — 官方实时控制器，编译产物 `z1_ctrl`，链接预编译的 `lib/libZ1_{aarch64|x86_64}.so`（按 `CMAKE_HOST_SYSTEM_PROCESSOR` 选）。依赖 Boost + Eigen3。属 [[z1-arm-project-overview]]。

要点：
- **实时**：`main.cpp` 里 `setProcessScheduler()` 设 SCHED_FIFO 最高优先级；控制周期 `dt = 1/250`（250Hz）。入口 [main.cpp](z1_controller/main.cpp)。
- **架构**：`CtrlComponents`（控制上下文）→ `FiniteStateMachine`（FSM）→ 各 `State_*`；命令源 `cmdPanel` 二选一（`Control::SDK` 或 `Control::KEYBOARD`）。
- **FSM 状态**（`include/common/enumClass.h` 的 `ArmFSMStateName`）：INVALID/PASSIVE/JOINTCTRL/CARTESIAN/MOVEJ/MOVEL/MOVEC/TRAJECTORY/TOSTATE/SAVESTATE/TEACH/TEACHREPEAT/CALIBRATION/SETTRAJ(弃用)/BACKTOSTART/NEXT/LOWCMD。每个状态在 `include/FSM/State_*.h`。
- **通信**：`IOUDP` → 下位机 MCU，IP `192.168.123.110`、端口 `8881`（`config/config.xml`）。SDK 模式监听 `127.0.0.1`，收 8072、发 8071。报文结构见 `include/message/`（7 电机：发送 238 字节 / 接收 546 字节）。
- **其它**：轨迹生成在 `include/trajectory/`（SCurve、EndLineTraj、EndCircleTraj、TrajectoryManager 等）；运动学/动力学在 `include/model/ArmModel.h`；QP 求逆解用 `thirdparty/quadProgpp`。键盘键位映射见 main.cpp 的 `StateAction`（`键对应状态、q/w/e/r/t/y 控制关节、上下键控制夹爪`）。`config/savedArmStates.csv` 存示教保存的位姿。
