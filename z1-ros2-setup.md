---
name: z1-ros2-setup
description: 本机 Z1 仿真实际用的 ROS2(Humble) 社区版方案，及已踩的坑
metadata: 
  node_type: memory
  type: project
  originSessionId: ca3d4080-7c55-4834-90a7-faf122652d74
  modified: 2026-09-30T06:42:38.772Z
---

本机是 **Ubuntu 22.04 + ROS 2 Humble**（无 ROS 1 Noetic），官方 `z1_ros` 是 Noetic 跑不了。改用社区 ROS2 移植版 **`idra-lab/z1_ros2`**，克隆在 `/home/wuqiuyu/robot_arm/ros2_ws/src/z1_ros2`，用 `colcon build --symlink-install` 编译成功。属 [[z1-arm-project-overview]]，官方链路见 [[z1-ros-driver]]。

子包：`z1_description`(URDF/xacro)、`z1_bringup`、`z1_hardware_interface`(ros2_control)、`z1_moveit`、`z1_examples`。

已踩并解决的坑：
- **FetchContent 下载失败**：`z1_hardware_interface` 的 CMake 用 FetchContent 去 github 拉官方 `z1_controller`/`z1_sdk` 的 master.zip，网络不稳定。解决：编译时传 `-DFETCHCONTENT_SOURCE_DIR_Z1_CONTROLLER=/home/wuqiuyu/robot_arm/z1_controller -DFETCHCONTENT_SOURCE_DIR_Z1_SDK=/home/wuqiuyu/robot_arm/z1_sdk` 指向本地已克隆仓库。
- **issue #8 effort 接口 bug**：已删除 `z1_description/urdf/z1.ros2_control.xacro` 里全部 7 个 `<command_interface name="effort">` 块（Ignition 的 ign/gz_ros2_control 插件不兼容）。

启动仿真（会弹 Ignition Gazebo + RViz 窗口）：
```bash
source /opt/ros/humble/setup.bash && source /home/wuqiuyu/robot_arm/ros2_ws/install/setup.bash
ros2 launch z1_bringup z1.launch.py starting_controller:=joint_trajectory_controller
```
切真机加 `sim_ignition:=false`。发轨迹用话题 `/joint_trajectory_controller/joint_trajectory`，示例 `ros2 run z1_examples waypoint_test.py`。

已装依赖（apt）：`ros-humble-moveit` + `moveit-configs-utils`、`ros2-control`/`ros2-controllers`、`ros-gz-sim`、`ign-ros2-control`、`gazebo-ros2-control`、`xacro`、`rviz2`、`tf2-ros`、`trajectory-msgs`、`control-msgs`、`libeigen3-dev`、`libfmt-dev`（`libboost-all-dev` 系统已装）。

注意：机器是 VirtualBox/VMware 虚拟机，**无 3D 加速**，Gazebo GUI 走软渲染（日志有 `VMware: No 3D enabled` / `libEGL failed to create dri2 screen`），功能正常但画面可能差。

2026-09-29 更新：项目已整体迁移到 `/home/wuqiuyu/robot_arm`（三仓库 + `ros2_ws`）。另发现 `/home/wuqiuyu/robot_arm/z1_fangzhen/` 是另一套 ROS2 仿真工作区（包 z1_description/z1_bringup/z1_controllers/z1_moveit_config，已 build），疑似另一移植版，未验证。

## 架构逻辑（2026-09-30 精读）

`ros2_ws/src/z1_ros2` 与 SDK 直连是**不同架构**：把 z1_sdk 包成 ros2_control 硬件插件，命令走 ROS2 标准接口(topic/action)而非直接调 API。

- **五包职责**：`z1_description`(URDF/xacro+网格+关节限位)、`z1_hardware_interface`(核心，`hardware_interface::SystemInterface` 插件，内部持 `UNITREE_ARM::unitreeArm`)、`z1_bringup`(launch+控制器配置)、`z1_moveit`(MoveIt 规划)、`z1_examples`(waypoint_test.py)。
- **ros2_control 循环**（`z1_hardware_interface.cpp`，controller_manager 1000Hz）：`read()` 调 `_arm->sendRecv()` 收状态拷进 `_arm_state` → 控制器 update 插值 → `write()` 饱和扭矩后 `setArmCmd+sendRecv` 下发。`export_state/command_interfaces()` 把 `_arm_state/_arm_cmd` 内存地址注册成 21 个 handle(7 关节×position/velocity/effort)。
- **真机走低层直控**：`on_configure` 里 `setFsm(PASSIVE)`→读一次状态→`setFsm(LOWCMD)`；轨迹规划交给 ros2_control 控制器，不用 SDK 的 MoveJ。`perform_command_mode_switch` 按接口类型(position/velocity/effort)动态设 kp/kd 增益。
- **四控制器**（`z1_bringup/config/z1_controllers.yaml`）：`joint_state_broadcaster`(只发 /joint_states)、`position_controller`(JointGroupPositionController)、`torque_controller`(JointGroupEffortController)、`joint_trajectory_controller`(JointTrajectoryController，MoveIt/action 用)。
- **sim_ignition 分叉**（`z1.launch.py`）：仿真=true 起 Gazebo+`ign_ros2_control/IgnitionSystem` 插件；=false 起 ros2_control_node + ExecuteProcess 跑 `z1_controller_process.py` 启动独立 `z1_ctrl` 进程。两边共用 robot_state_publisher+spawner；RViz 用 OnProcessExit 延迟到 broadcaster 起来后启动。
- **CMake FetchContent** 拉官方 z1_controller/z1_sdk master.zip 作为依赖（对应上面的坑）。
- **上位机视角**：上位机 = 一个 ROS2 节点，发 `FollowJointTrajectory` action(照抄 waypoint_test.py) 或 `/position_controller/commands` 话题；读 `/joint_states`。仿真/真机只差 `sim_ignition` 一个参数。
