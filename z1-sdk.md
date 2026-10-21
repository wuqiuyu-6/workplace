---
name: z1-sdk
description: 官方 z1_sdk 的 API 面、高层/低层控制与 Python 绑定
metadata: 
  node_type: memory
  type: project
  originSessionId: ca3d4080-7c55-4834-90a7-faf122652d74
  modified: 2026-09-30T06:42:31.361Z
---

`/home/wuqiuyu/robot_arm/z1_sdk` — 官方用户侧 SDK，头文件 `include/unitree_arm_sdk/` + 预编译 `lib/libZ1_SDK_{aarch64|x86_64}.so`。属 [[z1-arm-project-overview]]，与 [[z1-controller]] 配套。

核心 API（`control/unitreeArm.h` 的 `unitreeArm` 类，构造 `unitreeArm(hasGripper)`）：
- **高层 FSM**：`backToStart()`、`MoveJ/MoveL/MoveC(位姿, 夹爪, 速度)`、`setFsm(ArmFSMState)`、`labelRun("forward"/"backward")`（相对/绝对运动方向）。
- **持续控制**：`startTrack(ArmFSMState::JOINTCTRL/CARTESIAN)` 后循环调 `jointCtrlCmd(directions, speed)` / `cartesianCtrlCmd(directions, angular_vel, linear_vel)`，按 `_ctrlComp->dt` sleep。
- **低层直控**：`setFsm(LOWCMD)` 后 `setArmCmd(q, qd, tau)` + `setGripperCmd(...)` + `sendRecv()`，tau 用 `armModel->inverseDynamics(...)` 前馈。可 `setControlGain(KP, KW)`。
- **状态读取**：`lowstate->getQ()/getQd()/getTau()`、`lowstate->endPosture`（roll pitch yaw x y z）。
- 发送接收线程 `arm.sendRecvThread->start()/shutdown()`。

示例：`examples/highcmd_basic.cpp`（三种模式）、`highcmd_development.cpp`、`lowcmd_development.cpp`（扭矩直控）、`lowcmd_multirobots.cpp`（多机）。

Python：`examples_py/`，pybind11 绑定为 `lib/unitree_arm_interface`（`ArmInterface`），`example_highcmd.py` / `example_lowcmd.py` / `example_model.py` / `example_http_*.py`（HTTP 服务/客户端）。注意 arm 变量用 `arm._ctrlComp.dt` 取周期。

## 代码架构逻辑（2026-09-30 精读）

五层结构（自下而上）：`math/`（Eigen 别名+位姿↔齐次矩阵）→ `message/`（报文）+ `utilities/loop.h`（Timer 周期器/LoopFunc 后台线程）→ `model/ArmModel.h`（正逆解/雅可比/逆动力学/QP）→ `control/`（核心）→ `examples/`。

- **核心设计 = 容器 + 门面**：`CtrlComponents` 是依赖注入容器（聚合 `lowcmd/lowstate/udp/armModel/dt/sendCmd/recvState`）；`unitreeArm` 是用户唯一门面（持有 `_ctrlComp`，命令参数暂存区 `q/qd/tau/twist/gripperQ...`）。
- **数据流**：用户填 `unitreeArm` 命令参数 → 后台 `sendRecvThread`(500Hz，dt=0.002，控制器实际 250Hz 超采样) 序列化进 `SendCmd` → `udp->send()` → 收 `RecvState` 解包进 `lowstate`。
- **报文**（`message/arm_common.h`）：`#pragma pack(1)` 让结构体内存布局==线上字节，直接 cast 免序列化；`SendCmd.valueUnion` 用 union 复用空间（`name[10]`/`jointCmd[7]`/`trajCmd`）；`RecvState` 含 `jointState[7]` + `cartesianState`。
- **两种模式**：高层 FSM（`MoveJ/MoveL/MoveC`，轨迹由 z1_controller 规划，默认阻塞，可 `setWait(false)`）vs 低层持续（`startTrack(JOINTCTRL/CARTESIAN)` + `jointCtrlCmd/cartesianCtrlCmd`，自己周期发）。
- **代码风格**：头文件即文档（注释带单位/范围/注意）；Eigen 固定尺寸 `Vec6`(6 关节)+`std::vector<double>`(含夹爪 7 维)；`UNITREE_ARM` 命名空间、成员下划线前缀 `_xxx`。
- **学习路径**：`highcmd_basic.cpp` → `unitreeArm.h` 全注释 → `sendRecv` 数据流 → `arm_common.h` 报文 → `arm_python_interface.cpp`(pybind 桥，`ArmInterface` 继承 `unitreeArm` 加 loopOn/loopOff/getCurrentState) → `ArmModel.h`。
- **坑**：`labelRun("forward")` 定方向基准；`hasGripper` 定 6/7 维；`MoveJ` 默认阻塞(GUI 主线程会卡，需 `setWait(false)` 或子线程)；pybind 里 `getQTau` 是拼写错误(正确 `getTau`，两个都导出了)。

`.so` 预编译、实现黑盒，只能读头文件注释 + 示例学。
