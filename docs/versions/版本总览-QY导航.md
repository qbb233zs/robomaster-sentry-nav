# QY 导航备份版本变更记录（1.0–3.0）

生成日期：2026-10-19；最后补充相邻版本代码级对照：2026-10-20  
审计对象：HOME 下 5 个导航工程压缩包  
审计方式：读取压缩包内容，忽略 `build/install/log`，逐文件 CRC/SHA-256 比较，并对相邻版本做源码 diff  
约束：本文档只读取备份包和工作区历史资料，没有修改压缩包，也没有修改现有导航工程

---

## 0. 先看结论

1. **5 个包的真实顺序是 `1.0 → 1.1 → 2.0 → 2.1 → 3.0`。**
2. **真正发生导航源码/配置净变化的文件很少，核心只有 7 个左右。**
3. **1.1 是一次“底盘角方向补偿”实验，后来被 2.0 回退；不建议把 1.1 当成独立稳定版使用。**
4. **2.0 主要修速度链的时间戳/旧 local_plan 问题；2.1 主要把 `map→odom` 发布从 ROS 定时器改成独立线程；3.0 主要修决策树、到点判定和目标点。**
5. **3.0 里的 `small_gicp` 最终内容与 2.1 逐字节相同。** 3.0 包内新增的 `.bak-20261019-...` 只是记录了当天“先回退、再恢复”的过程，不代表 3.0 又改了一版 GICP。
6. **推荐以 3.0 作为当前基线。** 2026-10-19 18:36–18:55 的实车日志中，10 个目标全部成功，5 次“巡逻↔回家”往返完成。
7. **这 5 个包不包含 `CCUTVision/rm_serial_driver` 和视觉工程。** 串口、USB、udev、云台角 relay、视觉链路的变更不能从这 5 个导航包中完整审计；本文只对导航包内可见内容下结论。
8. **没有 `1.2` 压缩包。** HOME 中实际只有 `1.0、1.1、2.0、2.1、3.0`。如果你说的是“`1.1` 的下一版”，那就是 `2.0`；本文不会把 `2.0` 改写成 `1.2`，以免以后追溯时对不上真实备份。

---

## 0.1 直接回答：按“上一版 → 下一版”看

这 5 个包是连续的版本链，正确的比较方式是：

```text
1.0 → 1.1 → 2.0 → 2.1 → 3.0
```

| 升级 | 相对上一版实际改了什么 | 对应想解决/已解决的 bug |
|---|---|---|
| 无 → 1.0 | 没有更早的 QY 包，无法做“上一版 diff”；1.0 作为起点基线 | 1.0 里已经包含此前完成的 NaN 防护、翻译官死锁修复、GICP 限幅、云台雷达 TF、反应式决策树等旧修复；这些不是 1.0 这一版新改的 |
| 1.0 → 1.1 | `fake_vel_transform` 新增 `odom→base_footprint` 查询，速度改用底盘角旋转 | 试图修正“云台和底盘同时转时速度方向不一致”；但这是实验分支，2.0 已回退，不能算稳定修复 |
| 1.1 → 2.0 | 回退底盘角逻辑，速度重新按 odom/gimbal 基准角旋转；统一节点时钟；旧 `local_plan` 超过阈值时旁路同步；控制器位姿重新打当前时间戳 | 修旧 `local_plan` 时间戳让速度同步一直等待、速度不下发；减少混合时钟和 future extrapolation 问题 |
| 2.0 → 2.1 | `small_gicp` 的 `map→odom` 从 ROS wall timer 改为独立 20 Hz 线程，并用 mutex 保护结果；另外增加仿真资源 | 修 `map→odom` 发布回调被主执行器饿死导致的 TF 冻结、全局代价地图位姿变旧、点目标后规划器从旧位置规划 |
| 2.1 → 3.0 | `SendGoal` 增加“距离目标 ≤0.20 m 连续 1 秒即成功”；决策树改为 45 秒超时、外层 Timeout、回家也无限重试、高血阈值 580、巡逻点改为 `2.376,0.066` | 修 Nav2 停在目标附近不返回结果；修完全没 feedback 时内部 timeout 不触发；修回家分支一次失败打穿整棵树；修 HP 351 提前切回巡逻；修目标点与当前地图不匹配 |

一句话：

- **1.1 是方向实验，不是成功版本。**
- **2.0 修速度链时间戳。**
- **2.1 修定位 TF 发布。**
- **3.0 修决策树和到点判定。**

本文后面第 4、5 节就是按这个顺序展开的。

### 0.2 先说明版本号：`1.1 → 1.2` 这一对不存在

实际备份链是：

```text
1.0 → 1.1 → 2.0 → 2.1 → 3.0
```

所以按“相邻版本”比较，只能是下面四组：

```text
1.0 → 1.1
1.1 → 2.0   （不是 1.2）
2.0 → 2.1
2.1 → 3.0
```

下面第 4 节按这四组给出**文件、代码前后、对应 bug、测试现象和最终结论**。

---

## 1. 五个压缩包

| 版本 | 实际文件 | 打包时间 | 大小 |
|---|---|---|---:|
| 1.0 | `/home/robot/nav-master-ccut-QY-newcccutNav2-1.0.zip` | 2026-10-17 12:56:09 | 244,989,171 B |
| 1.1 | `/home/robot/nav-master-ccut-QY-newcccutNav2-1.1.zip` | 2026-10-17 14:58:18 | 245,049,157 B |
| 2.0 | `/home/robot/nav-master-ccut-QY-newcccutNav2-2.0.zip` | 2026-10-18 09:22:41 | 241,459,121 B |
| 2.1 | `/home/robot/nav-master-ccut-QY-newcccutNav2-2.1.zip` | 2026-10-19 07:50:34 | 241,571,771 B |
| 3.0 | `/home/robot/nav-master-ccut-QY-newcccutNav2-3.0.zip` | 2026-10-19 19:56:29 | 241,659,812 B |

包内公共根目录：

```text
nav-master-ccut-WCZ-newcccutNav2/
```

注意：压缩包文件名是 `QY`，包内目录仍叫 `WCZ-newcccutNav2`，这是命名不一致，不代表有第六个版本。

---

## 2. 相邻版本净差异总表

这里的“净差异”只统计 `nav-master-ccut/src`、`rm_decision/src`、`docs` 等源码/配置/文档；`build`、`install`、`log`、`.git` 全部排除。`.bak-*` 也不参与净差异统计，但会在后文单独说明。

| 区间 | 新增 | 删除 | 修改 | 真正的功能变化 |
|---|---:|---:|---:|---|
| 1.0 → 1.1 | 4 个备份 | 0 | 2 | `fake_vel_transform` 增加底盘角 TF 查询，速度按底盘角旋转 |
| 1.1 → 2.0 | 0 | 0 | 2 | 回退底盘角实验；增加节点时钟、旧 `local_plan` 旁路；控制器重新打时间戳 |
| 2.0 → 2.1 | 9 个仿真/备份文件 | 0 | 3 | `small_gicp` 改独立 20 Hz 发布线程；增加 RMUL2027 3V3 仿真资源 |
| 2.1 → 3.0 | 12 个备份 | 0 | 2 | 决策树目标/阈值/超时/重试；`SendGoal` 增加“近目标 1 秒即到达” |

重要判读：

- 很多“版本变化”实际上是 `.bak-*` 文案或备份文件，不改变运行行为。
- 1.0→3.0 之间，**决策树直到 3.0 才发生净变化**；2.1 及之前仍是老 XML。
- 1.0→3.0 之间，**`pointcloud_to_laserscan`、`recovier_behavior` 的 NaN 保护、真实地图、`nav2_params.yaml` 的实车参数没有净变化**。

---

## 3. 关键文件指纹

下面用 SHA-256 前 12 位确认“哪一版真的变了”。同一指纹表示内容完全相同。

| 文件 | 1.0 | 1.1 | 2.0 | 2.1 | 3.0 |
|---|---|---|---|---|---|
| `fake_vel_transform.cpp` | `ef602a6afc58` | `35a99ebde5da` | `e9dd11669e53` | `e9dd11669e53` | `e9dd11669e53` |
| `omni_pid_pursuit_controller.cpp` | `eb3c6addc286` | `eb3c6addc286` | `9d05412c97e7` | `9d05412c97e7` | `9d05412c97e7` |
| `small_gicp_relocalization.cpp` | `a2283f9f94a6` | `a2283f9f94a6` | `a2283f9f94a6` | `0a1cc89fb177` | `0a1cc89fb177` |
| `full_decision.xml` | `4807c5ca9642` | `4807c5ca9642` | `4807c5ca9642` | `4807c5ca9642` | `128a9fe4ad2c` |
| `send_goal.cpp` | `1e321ca157a1` | `1e321ca157a1` | `1e321ca157a1` | `1e321ca157a1` | `c98997e88df8` |

补充：

- 3.0 的 `small_gicp_relocalization.cpp` 与 2.1 **逐字节相同**。
- 3.0 的 `small_gicp` 文件时间是 2026-10-19 18:30:02，但内容指纹仍是 2.1 的那一版；这是“恢复同一内容”，不是新逻辑。
- 3.0 包内 `~/.bak-20261019-164200` 对应线程版，`~/.bak-20261019-1830` 对应 timer 版，说明当天 16:42 曾回退到 timer，18:30 又恢复线程版。

---

## 4. 小版本逐项变更

### 4.1 1.0：整理后的基线

打包时间：2026-10-17 12:56。  
关键源码时间：

- `fake_vel_transform.cpp`：2026-10-16 18:25:48
- `full_decision.xml`：2026-10-17 12:37:56
- `small_gicp_relocalization.cpp`：2026-09-22 06:55:24（旧版本）

1.0 本身不是“从零开始”的版本。它已经把此前几天的关键修复带进来了，主要包括：

1. **恢复行为空集 NaN 保护**
   - 文件：`nav-master-ccut/src/rm_navigation/recovier_behavior/src/recovier_behavior.cpp`
   - 内容：`free_points.empty()` 时直接 `stopRobot()` 并返回 `FAILED`，不再执行 `0/0` 求平均。
   - 目的：防止 recovery/backup 行为在找不到空闲点时产生 `vx=nan, vy=nan`，进而把底盘 PID 积分器毒死，出现“轮子全瘫”。

2. **翻译官 tf2 死锁修复**
   - 文件：`nav-master-ccut/src/rm_perception/pointcloud_to_laserscan/src/pointcloud_to_laserscan_node.cpp`
   - 内容：绕开 `MessageFilter::waitForTransform` 的 AB-BA 死锁路径，改为直接订阅并在回调内做 TF 转换；TF 太旧时丢帧。
   - 目的：解决 `obstacle_scan` 偶发 0 Hz、`/map` 冻结、建图中断的问题。
   - 证据：`outputs/deadlock_evidence/建图失败根因-翻译官tf2死锁.md` 和 `翻译官修复-说明与回退-20261013.md`。

3. **small_gicp 的 `map→odom` 时间戳修复和限幅**
   - 文件：`small_gicp_relocalization.cpp`
   - `map→odom` 使用 `this->now()`，不再使用旧点云时间戳打戳。
   - 文件：`nav2_params.yaml`
   - 实车参数已含：
     - `max_correction_translation: 1.5`
     - `max_correction_rotation: 0.4`
     - 翻译官 `min_height: -0.5`、`max_height: 6.0`、`min_intensity: 0.1`
   - 目的：避免旧 TF 时间戳把全局定位钉死；同时限制 GICP 单次乱跳。

4. **雷达挂到云台、不挂底盘**
   - 文件：`pb2025_robot_description/resource/xmacro/pb2025_sentry_robot.sdf.xmacro`
   - 当前：`parent="gimbal_yaw"`，pose 约 `(0.0496, 0.0936, 0.409)`。
   - 目的：物理雷达确实随云台转，URDF/TF 必须一致。

5. **云台角和 joint_state_publisher**
   - `params/robot_description.yaml`：`rate: 200`，`source_list: ["serial/gimbal_joint_state"]`。
   - 目的：让 `gimbal_yaw_joint` 及时进入 `/joint_states`，使 `base_footprint→gimbal_yaw→front_mid360` 的 TF 随云台实时变化。
   - 串口 relay 本体在 `CCUTVision`，不在导航压缩包内。

6. **决策树反应式掉血/回家**
   - 文件：`rm_decision/src/rm_behavior_tree/config/full_decision.xml`
   - 已从旧 `Sequence + IfThenElse` 改成 `ReactiveSequence + ReactiveFallback`。
   - `sub_robot_status.cpp` 已改为“有旧值就保持 SUCCESS”，否则每个 tick 没新消息就会打断整棵树。
   - 虚拟裁判 `robot_status_publisher.cpp` 已支持 mode 4 完整循环：掉血→低血停留→回血→再掉血。

1.0 仍存在的问题：

- 回家分支没有 `RetryUntilSuccessful`。
- 两个目标超时仍是 30 秒。
- 高血阈值仍是 `high_hp=350`，所以 HP 一到 351 就会从“回家”切回“巡逻”。
- `gimbal_yaw_fake` 速度链仍会在旧 `local_plan` 时间戳下走同步分支，存在“控制器活跃但速度没下发”的风险。
- `small_gicp` 的 `map→odom` 仍用 ROS wall timer，执行器繁忙时可能饿死。
- 还没有“近目标 0.20 m 持续 1 秒视为到达”的兜底。

基线方向链：

```text
1.0: yaw_diff = current_robot_base_angle_
```

即只按里程计里的机器人基准角旋转速度，不直接使用 `odom→base_footprint` 的底盘角。

---

### 4.2 1.1：底盘角方向补偿实验

打包时间：2026-10-17 14:58。  
关键源码时间：`fake_vel_transform.cpp` 2026-10-17 13:16:12。

净改动只有两个文件：

1. `nav-master-ccut/src/rm_navigation/fake_vel_transform/include/fake_vel_transform/fake_vel_transform.hpp`
2. `nav-master-ccut/src/rm_navigation/fake_vel_transform/src/fake_vel_transform.cpp`

具体改动：

1. 新增 `tf2_ros::Buffer`、`TransformListener`。
2. 新增参数：
   - `odom_frame`，默认 `odom`
   - `chassis_frame`，默认 `base_footprint`
3. 新增 `updateChassisAngle()`：

```cpp
lookupTransform(odom_frame_, chassis_frame_, tf2::TimePointZero)
current_chassis_angle_ = getYaw(...)
```

4. 速度旋转角从：

```cpp
yaw_diff = current_robot_base_angle_;
```

改为：

```cpp
yaw_diff = has_chassis_angle_ ? current_chassis_angle_ : current_robot_base_angle_;
```

#### 1.0 → 1.1 的关键代码前后

`fake_vel_transform.hpp` 新增：

```diff
+#include "tf2_ros/buffer.h"
+#include "tf2_ros/transform_listener.h"

+void updateChassisAngle();

+std::unique_ptr<tf2_ros::Buffer> tf_buffer_;
+std::unique_ptr<tf2_ros::TransformListener> tf_listener_;
+std::string odom_frame_;
+std::string chassis_frame_;
+double current_chassis_angle_{0.0};
+bool has_chassis_angle_{false};
```

`fake_vel_transform.cpp` 新增 TF 查询并改变速度旋转角：

```diff
+void FakeVelTransform::updateChassisAngle()
+{
+  const auto tf_stamped = tf_buffer_->lookupTransform(
+    odom_frame_, chassis_frame_, tf2::TimePointZero);
+  current_chassis_angle_ = tf2::getYaw(tf_stamped.transform.rotation);
+  has_chassis_angle_ = true;
+}

-auto aft_tf_vel = transformVelocity(msg, current_robot_base_angle_);
+const double yaw_diff =
+  has_chassis_angle_ ? current_chassis_angle_ : current_robot_base_angle_;
+auto aft_tf_vel = transformVelocity(msg, yaw_diff);
```

#### 1.0 → 1.1 的实车测试发现

- 用户实车反馈：发送“向前”速度后，小车仍然明显向右偏，说明这次“直接用 `odom→base_footprint` 底盘角补偿”的方向定义与电控接收速度的坐标系不匹配。
- 2026-10-17 13:16–13:18 的两场日志中，一场出现 13 次 `Collision detected in the trajectory`、1 次 `Failed to make progress`；另一场 2 次目标启动、4 次 `Failed to make progress`，没有成功目标。日志不能把所有失败单独归因于 1.1，但足以证明 1.1 没有通过实车验证。
- 因此 2.0 没有继续沿这个方向迭代，而是把速度旋转角恢复为 `current_robot_base_angle_`。

这段改动想解决的问题：

- 云台和底盘同时旋转时，Nav2 输出速度所在的 `gimbal_yaw_fake` 系、里程计系、真实底盘系之间存在角度差。
- 1.1 试图直接把速度旋转到真实底盘 `base_footprint` 系，减少“目标方向正确但车走偏”的问题。

为什么它不是最终方案：

- 2.0 又把 `yaw_diff` 改回 `current_robot_base_angle_`，并在代码注释里明确写了“指令在 gimbal/lidar 系表达，云台→底盘的变换由控制板处理”。
- 说明 1.1 的“直接用底盘角”与电控实际接收的速度坐标系不匹配，属于中间实验。
- 1.1 没有解决时间戳问题，也没有修决策树。

备份文件说明：

- 1.0→1.1 包内新增了 4 个 `.bak-*` 文件，只是保存了 10-17 13:07 和 13:16 的实验版本，不是运行逻辑的一部分。

结论：**1.1 可以保留作方向调试证据，但不建议单独作为比赛版本。**

---

### 4.3 2.0：回退方向实验，修速度链时间戳/旧计划

打包时间：2026-10-18 09:22。  
关键源码时间：

- `fake_vel_transform.cpp`：2026-10-17 19:35:32
- `omni_pid_pursuit_controller.cpp`：2026-10-18 08:22:44

净改动两个文件：

#### A. `fake_vel_transform.cpp`

1. 新增 `<cmath>`。
2. 控制器超时判断统一使用节点时钟：

```cpp
const double controller_age =
  static_cast<double>(this->now().nanoseconds() -
                      last_controller_activate_time_.nanoseconds()) / 1e9;
```

3. `localPlanCallback` 增加旧时间戳检测：

```cpp
const double plan_offset = static_cast<double>(
  now.nanoseconds() - rclcpp::Time(msg->header.stamp).nanoseconds()) / 1e9;
if (std::abs(plan_offset) > CONTROLLER_TIMEOUT) {
  // 把 last_controller_activate_time_ 推到超时之外，强制走直发路径
  last_controller_activate_time_ =
    now - rclcpp::Duration::from_seconds(CONTROLLER_TIMEOUT + 1.0);
  return;
}
```

4. `syncCallback` 也拒绝旧 `local_plan`：

```cpp
if (std::abs(plan_offset) > CONTROLLER_TIMEOUT) {
  return;
}
```

5. 速度旋转角回到：

```cpp
const double yaw_diff = current_robot_base_angle_;
```

6. 注释明确：

```text
command 在 gimbal/lidar frame；控制板负责 gimbal→chassis
```

#### 1.1 → 2.0 的关键代码前后

旧计划时间戳旁路：

```diff
-if ((rclcpp::Clock().now() - last_controller_activate_time_).seconds() > CONTROLLER_TIMEOUT) {
+const double controller_age =
+  static_cast<double>(this->now().nanoseconds() -
+                      last_controller_activate_time_.nanoseconds()) / 1e9;
+if (controller_age > CONTROLLER_TIMEOUT) {

+const double plan_offset = static_cast<double>(
+  now.nanoseconds() - rclcpp::Time(msg->header.stamp).nanoseconds()) / 1e9;
+if (std::abs(plan_offset) > CONTROLLER_TIMEOUT) {
+  last_controller_activate_time_ =
+    now - rclcpp::Duration::from_seconds(CONTROLLER_TIMEOUT + 1.0);
+  return;
+}
```

控制器位姿重新打时间戳：

```diff
-auto transformed_plan = transformGlobalPlan(pose);
+geometry_msgs::msg::PoseStamped current_pose = pose;
+current_pose.header.stamp = clock_->now();
+auto transformed_plan = transformGlobalPlan(current_pose);

-cmd_vel.header = pose.header;
+cmd_vel.header = current_pose.header;
```

注意：2.0 只是**不再用** 1.1 的 `current_chassis_angle_` 来算速度，1.1 引入的 TF listener、`updateChassisAngle()` 和成员变量仍然保留在 2.0/3.0 中。也就是说，它是“行为回退”，不是把 1.1 的增加代码全部删除。

#### 1.1 → 2.0 的实车测试发现

- 2026-10-18 11:30 的 2.0 实车日志中，巡逻目标先触发 4 次碰撞停车，之后两次 `Failed to make progress`；最后只有回原点目标成功（3 次目标启动、1 次成功）。
- 这说明时间戳修复是针对“速度链/旧计划”的一步，但没有解决全局定位 TF 冻结和决策树问题，不能把 2.0 当成最终稳定版。

这段改动解决的问题：

- `local_plan` 时间戳一旦过旧，旧逻辑会把 `last_controller_activate_time_` 一直当成“控制器刚活跃”，导致同步分支一直等待，速度可能不下发。
- 统一用节点时钟计算年龄，避免 `rclcpp::Clock()` 与节点时钟混用导致不同时间源问题。
- 把速度方向逻辑恢复到 1.0 的 gimbal/odom 基准，撤销 1.1 的底盘角实验。

#### B. `omni_pid_pursuit_controller.cpp`

在 `computeVelocityCommands` 中增加：

```cpp
geometry_msgs::msg::PoseStamped current_pose = pose;
current_pose.header.stamp = clock_->now();
auto transformed_plan = transformGlobalPlan(current_pose);
...
cmd_vel.header = current_pose.header;
```

作用：

- Nav2 传进来的位姿可能携带旧 TF 采样时间。
- 重新用当前控制器时钟打戳，使局部路径、TF 查询、后续速度同步使用同一个当前时间。
- 降低 `Lookup would require extrapolation into the future` 和旧 `local_plan` 带来的链路不一致。

2.0 后仍存在的问题：

- `small_gicp` 的 `map→odom` 仍由 ROS wall timer 发布。
- 决策树仍是 1.0 的 30 秒、`high_hp=350`、回家不重试。
- 没有近目标兜底。

---

### 4.4 2.1：`map→odom` 独立发布线程 + 仿真资源

打包时间：2026-10-19 07:50。  
关键源码时间：`small_gicp_relocalization.cpp` 2026-10-18 12:49:12。

净改动：

#### A. `small_gicp_relocalization.hpp/.cpp`

1. 用独立 `std::thread` 替代 ROS timer 发布 `map→odom`：

```cpp
transform_thread_ = std::thread([this]() {
  rclcpp::WallRate rate(20.0);
  while (rclcpp::ok() && !transform_thread_stop_.load()) {
    publishTransform();
    rate.sleep();
  }
});
```

2. 析构时停止线程并 `join()`。
3. 新增 `result_mutex_`，保护 `result_t_` 的读写：

```cpp
{
  std::lock_guard<std::mutex> lock(result_mutex_);
  result_t_ = initial_map_to_odom_ * gicp_correction_;
}
```

4. `publishTransform()` 拷贝一份结果后再发布，避免 GICP 线程同时改写。

#### 2.0 → 2.1 的关键代码前后

```diff
-transform_timer_ = this->create_wall_timer(
-  std::chrono::milliseconds(50),
-  std::bind(&SmallGicpRelocalizationNode::publishTransform, this));
+transform_thread_ = std::thread([this]() {
+  rclcpp::WallRate rate(20.0);
+  while (rclcpp::ok() && !transform_thread_stop_.load()) {
+    publishTransform();
+    rate.sleep();
+  }
+});

-result_t_ = initial_map_to_odom_ * gicp_correction_;
+{
+  std::lock_guard<std::mutex> lock(result_mutex_);
+  result_t_ = initial_map_to_odom_ * gicp_correction_;
+}
```

这段改动解决的问题：

- ROS wall timer 由执行器调度；当 GICP、TF、Nav2 回调把执行器占满时，`map→odom` 可能长时间不发。
- `map→odom` 一旦冻结，全局代价地图中的机器人位姿就会停在旧时刻，规划器会从旧位置规划，表现为“点目标不走”“路径起点不对”“RViz 里点云/车位置冻结”。
- 独立线程按 20 Hz 发布，不依赖主执行器是否繁忙。

#### B. 仿真资源

2.1 新增了：

- `rm_nav_bringup/map/simulation/rmul2027_3v3.pgm/.yaml`
- `rmu_gazebo_simulator/.../rm_rmul2027/model.config`
- `model.sdf`
- `RMUL2027_3V3_V1.0.0.obj`
- `rmul2027_3v3_world.sdf`
- `gz_world.yaml` 的 `rmul2027_3v3` 配置

这些是 Gazebo 仿真资源，不参与实车导航速度链。不要把 2.1 的所有新增文件都理解成实车修复。

#### 2.0 → 2.1 的实车测试发现

| 测试 | `odom→map` 报错 | 最大陈旧时间 | 目标结果 | 说明 |
|---|---:|---:|---|---|
| 10-18 12:43，timer 旧逻辑 | 42 次 | 49.4 s | 1 次启动，1 次 `Goal failed` | 全局代价地图拿到旧的 `map→odom`，规划失败 |
| 10-18 12:46，timer 旧逻辑 | 42 次 | 42.3 s | 1 次启动，1 次 `Goal failed` | 同一类冻结 |
| 10-18 12:52，线程版 | 0 次 | — | 2 次启动，1 次成功 | 干净启动后，`odom→map` 冻结报错消失 |
| 10-18 13:04，线程版 | 0 次 | — | 4 次启动，4 次成功 | 该场目标全部完成，但仍有 71 条 `map→gimbal_yaw_fake` 的 future 警告和 6 次碰撞停车 |
| 10-18 12:55，线程切换期的一次运行 | 42 次 | 28.6 s | 3 次启动，1 次成功 | 仍看到了旧症状；可能是未干净重启/未重编/环境干扰，单凭日志不能判定。结论是验证必须干净重启，不能只看单场 |

结论：

- 2.1 对“`map→odom` 被 ROS timer 饿死”这个具体问题是有效修复；13:04 的 4/4 成功是最直接的正面证据。
- 它不是“所有 TF 都正常”。`map→gimbal_yaw_fake` 的短时 future 警告和碰撞停车仍然存在，所以后面还要继续做决策树/到点兜底和实车调参。

2.1 后仍存在的问题：

- 决策树还是 1.0 版本：30 秒、`high_hp=350`、回家不重试。
- 还没有 `SendGoal` 的近目标判定。
- 2.1 的 GICP 独立线程是真实改动；3.0 最终内容与它相同。

---

### 4.5 3.0：决策树/到点判定/目标点收尾

打包时间：2026-10-19 19:56。  
关键源码时间：

- `send_goal.cpp`：2026-10-19 15:47:54
- `small_gicp`：2026-10-19 18:30:02（内容与 2.1 相同）
- `full_decision.xml`：2026-10-19 18:31:56

净改动只有两个文件：

1. `rm_decision/src/rm_behavior_tree/src/plugins/action/send_goal.cpp`
2. `rm_decision/src/rm_behavior_tree/config/full_decision.xml`

#### A. `send_goal.cpp`：近目标判定

新增：

```cpp
constexpr double kGoalReachedDistance = 0.20;
constexpr double kGoalReachedHoldSeconds = 1.0;
```

实际判定代码：

```cpp
if (dist <= kGoalReachedDistance) {
  if (!in_small_region_) {
    in_small_region_ = true;
    small_region_start_ = now;
  } else if ((now - small_region_start_).seconds() >= kGoalReachedHoldSeconds) {
    setLastReachedGoal(goal_key_);
    return BT::NodeStatus::SUCCESS;
  }
} else {
  in_small_region_ = false;
}
```

规则：

- `distance_remaining <= 0.20 m`
- 连续保持 1 秒
- 则把目标判定为 `SUCCESS`

解决的问题：

- Nav2 有时会停在离目标几厘米处反复旋转/微调，既不返回成功，也不返回失败。
- 外层决策树等到 45 秒后取消目标，实际上车已经到了。
- 现在把“稳定在近目标区域”作为到达条件，避免无意义超时。

#### B. `full_decision.xml`：最终决策逻辑

最终关键值：

| 项目 | 1.0/2.1 | 3.0 最终 |
|---|---|---|
| 低血阈值 | `low_hp=300` | `low_hp=300` |
| 高血阈值 | `high_hp=350` | `high_hp=580` |
| 巡逻目标 | `-5.00, 3.20` | `2.376, 0.066` |
| 回家目标 | `0.00, 0.00` | `0.00, 0.00` |
| 单目标超时 | 30 s | 45 s |
| 外层 Timeout | 无 | 45,000 ms |
| 回家重试 | 无 | `RetryUntilSuccessful num_attempts=-1` |
| 巡逻重试 | 有 | 有 |

最终结构：

```text
ReactiveSequence
  SubRobotStatus
  ReactiveFallback
    ReactiveSequence(巡逻)
      IsStatusOK(low_hp=300, high_hp=580)
      RetryUntilSuccessful
        Timeout(45000ms)
          SendGoal(PatrolGoal, 2.376, 0.066, timeout=45)
    RetryUntilSuccessful(回家)
      Timeout(45000ms)
          SendGoal(HomeGoal, 0.0, 0.0, timeout=45)
```

#### 2.1 → 3.0 的关键配置前后

```diff
-<IsStatusOK name="CheckHP" low_hp="300" high_hp="350" .../>
+<IsStatusOK name="CheckHP" low_hp="300" high_hp="580" .../>

-<SendGoal name="PatrolGoal" timeout="30"
-          goal_pose="-5.00;3.20;0;0;0;0;1" .../>
+<Timeout msec="45000">
+  <SendGoal name="PatrolGoal" timeout="45"
+            goal_pose="2.376;0.066;0;0;0;0;1" .../>
+</Timeout>

-<SendGoal name="HomeGoal" timeout="30"
-          goal_pose="0.00;0.00;0;0;0;0;1" .../>
+<RetryUntilSuccessful num_attempts="-1">
+  <Timeout msec="45000">
+    <SendGoal name="HomeGoal" timeout="45"
+              goal_pose="0.00;0.00;0;0;0;0;1" .../>
+  </Timeout>
+</RetryUntilSuccessful>
```

每个改动对应的问题：

1. `high_hp=350 → 580`
   - 旧逻辑：血量 300 回家，回到 351 就立刻切回巡逻；虚拟裁判回血过程中会反复“回家/巡逻”切换。
   - 新逻辑：回家后必须恢复到 580 以上才回巡逻，形成大的滞环，避免在 351/350 附近抖动。

2. `PatrolGoal` 从 `-5.00,3.20` 改成 `2.376,0.066`
   - 对齐当前地图/实验室的巡逻点。
   - 日志中显示为四舍五入后的 `(2.38, 0.07)`，这是正常显示精度。

3. 30 s → 45 s
   - 给窄门、绕障、GICP 短暂抖动更多时间，避免还没到就超时重发。

4. 外层 `<Timeout msec="45000">`
   - `SendGoal` 内部的 timeout 主要依赖反馈；如果 Nav2 完全停止反馈，内部 timeout 可能不触发。
   - 外层 Timeout 在 BT tick 层强制取消并交给 `RetryUntilSuccessful`。

5. 回家分支加 `RetryUntilSuccessful`
   - 旧逻辑回家只发一次；一次规划失败/超时就会把整棵树打成 FAILURE。
   - 新逻辑持续重试，直到成功或被上层反应式条件切走。

#### C. 3.0 包内中间备份：不要误读成最终版本

3.0 包内新增了 12 个 `.bak-*`，其中 XML 备份记录了以下中间状态（按内容整理）：

| 备份/中间状态 | 行为 |
|---|---|
| `bak-20261019-exttimeout` | 45 秒 + 回家重试，但还没有外层 Timeout |
| `bak-20261019-goalchange` | 加入外层 Timeout |
| `bak-door-20261019-171721` | 巡逻点改为 `-1.10, 5.06` |
| `bak-home-20261019-2.376` | 回家点改为 `-1.10, 0.00`，目标点仍在试错 |
| `bak-coordfix-20261019-1815` | 回家点曾误设为 `2.376, 0.066` |
| `bak-hp580-20261019-1835` | 巡逻/回家点已恢复为 `2.376,0.066` / `0,0`，但高血仍是 350 |
| 最终 XML | 再把 `high_hp` 改为 580，形成 `2.376,0.066` + `0,0` + 580 的最终组合 |

这些备份文件的名字不完全等于内容，例如 `bak-home-20261019-2.376` 的内容里 home 实际是 `-1.10,0.0`。**版本结论一律以 diff 和最终文件内容为准，不能只看备份文件名。**

#### 2.1 → 3.0 的实车测试发现

| 时间/测试 | 当时配置/代码状态 | 日志现象 | 说明 |
|---|---|---|---|
| 10-19 17:19 | 近目标判定已生效；`high_hp=350`；巡逻/回家仍为 `-1.10,5.06` / `-1.10,0.0`；GICP 当天曾临时回退 timer | 4 次目标启动、1 次成功、1 次 `Goal failed`；HP=301 进 LOW 后 4 秒，HP 到 352 立刻切回巡逻；近目标在 `0.10 m` 连续 1 秒被判成功 | 近目标兜底和回家重试确实生效；但 `high_hp=350` 仍造成抖动。该场 `odom→map` 最大陈旧约 113.6 秒，所以一次回家目标被 TF 冻结打断 |
| 10-19 18:16 | `high_hp=350`；点位已是 `2.38,0.07` / `0,0`；GICP 仍是临时 timer 版 | 12 次目标启动、6 次成功、1 次失败；HP 在 301→352 之间反复触发回家/巡逻；`odom→map` 最大陈旧约 727.9 秒 | 证明“低血后不回家”不是决策树完全没发目标，而是 TF 冻结和 `high_hp=350` 双重问题 |
| 10-19 18:36–18:55 | 最终 3.0：`high_hp=580`；`2.38,0.07` / `0,0`；GICP 恢复为独立线程；近目标 + 45 秒外层 Timeout | 10 次目标启动、10 次成功；5 次巡逻↔回家往返；`odom→map` 冻结报错 0 次；仅 6 条约 13 ms 的 `map→gimbal_yaw_fake` future 警告和 1 次碰撞停车；HP=301 后一直保持 LOW，直到 HP=582 才切回巡逻 | 这是 3.0 最终闭环的直接证据，但不是“TF 零告警” |

补充说明：3.0 包内 GICP 备份显示，10-19 16:42 曾把线程版备份后回退成 timer，18:30 又把 timer 备份后恢复成线程版；最终 `small_gicp` 与 2.1 逐字节相同。从备份时间、17:19/18:16 的日志以及 18:36 恢复线程版后的对比看，前两场的长时陈旧与当天的 timer 中间版本相符；18:36 最终线程版没有再出现该类 `odom→map` 冻结报错。

#### D. 3.0 的 GICP 说明

3.0 包内新增：

- `small_gicp_relocalization.cpp.bak-20261019-164200`
- `small_gicp_relocalization.cpp.bak-20261019-1830`
- 以及对应 hpp 备份

但最终 3.0 的 `small_gicp` 是线程版，和 2.1 逐字节相同。  
因此：**3.0 的净功能修复不包含“新 GICP 逻辑”；3.0 的净功能变化是决策树 + `SendGoal`。**

### 4.6 四组相邻版本一页结论

| 升级 | 修改文件 | 改了什么 | 针对的 bug | 测试结论 |
|---|---|---|---|---|
| 1.0 → 1.1 | `fake_vel_transform.hpp/.cpp` | 增加 `odom→base_footprint` TF 查询，速度改用底盘角旋转 | 试图修云台/底盘同时转时的速度方向偏差 | **失败实验**：前进仍右偏；随后出现碰撞和进度失败，2.0 回退 |
| 1.1 → 2.0 | `fake_vel_transform.cpp`、`omni_pid_pursuit_controller.cpp` | 统一节点时钟；旧 `local_plan` 旁路；控制器位姿重置时间戳；恢复 odom/gimbal 速度角 | 旧计划时间戳让同步一直等待、速度不下发；混合时钟与 future extrapolation | 修复方向正确，但 2.0 实车仍有碰撞/进度失败，尚需 2.1 |
| 2.0 → 2.1 | `small_gicp_relocalization.hpp/.cpp`，另有仿真资源 | `map→odom` 从 ROS timer 改为独立 20 Hz 线程；mutex 保护结果 | 主执行器繁忙导致 `map→odom` 冻结、全局代价地图位姿变旧、目标规划失败 | 13:04 干净运行 4/4 成功；`odom→map` 冻结报错 0；仍有少量其他 TF future 警告 |
| 2.1 → 3.0 | `send_goal.cpp`、`full_decision.xml` | 0.20 m 持续 1 秒到点；45 秒内部/外部超时；回家重试；`high_hp=580`；新巡逻点 | 近目标不返回、无反馈时 timeout 不触发、回家一次失败打穿树、351 血抖动 | 18:36–18:55 长跑 10/10 成功，5 次往返；HP 到 582 才回巡逻 |

---

## 5. 大版本归并

### V1 系列（1.0–1.1）：基础链路基线 + 方向补偿实验

1.0 中已经生效的基础修复：

- 恢复行为空集导致 NaN 速度；
- 翻译官 tf2 死锁导致 `obstacle_scan` 断流、建图冻结；
- `map→odom` 使用旧点云时间戳；
- GICP 单次修正过大；
- 雷达 TF 挂底盘而不是云台；
- 云台角进入 joint_states；
- 决策树从顺序结构改成反应式结构；
- 虚拟裁判 mode 4 完整循环；
1.1 则是在这个基线上做了一次底盘角方向实验，最终被 2.0 回退。

没有解决：

- 旧 `local_plan` 时间戳导致的同步挂起；
- `map→odom` 发布回调被主执行器饿死；
- 回家分支不重试；
- 30 秒超时；
- 351 血提前切回巡逻；
- 近目标不判定成功。

### V2.0：速度链时间戳稳定

解决/针对：

- `fake_vel_transform` 的旧 `local_plan` 旁路；
- 统一节点时钟；
- 控制器位姿重新打当前时间戳；
- 回退 1.1 的底盘角实验，恢复 gimbal/odom 速度基准。

这版的直接目标是让速度链在旧 `local_plan`/混合时钟下仍能继续下发速度。  
但 2.0 仍可能出现 `map→odom` 长时间不更新，因此还需要 2.1。

### V2.1：定位 TF 发布稳定

解决/针对：

- GICP 的 `map→odom` 从 ROS timer 改成独立 20 Hz 线程；
- 用 mutex 保护 GICP 结果；
- 避免执行器繁忙时全局定位 TF 冻结；
- 增加 RMUL2027 3V3 仿真资源和地图。

这是“点目标后全局代价地图位姿还是旧的”这一类问题的关键版本。

### V3.0：任务闭环和到点判定

解决/针对：

- HP 300 回家、351 立刻回巡逻的抖动；
- 回家分支一次失败就打断整棵树；
- Nav2 停在目标附近反复微调；
- 内部 timeout 依赖 feedback 的问题；
- 目标点与当前地图不匹配；
- 最终巡逻点 `2.376,0.066`、回家点 `0,0`。

---

## 6. 方向链演进（最容易看混的一条）

| 版本 | `yaw_diff` 实际取值 | 说明 |
|---|---|---|
| 1.0 之前/1.0 | `current_robot_base_angle_` | 里程计基准角 |
| 1.1 | `current_chassis_angle_`（优先） | 直接查 `odom→base_footprint`，实验性 |
| 2.0–3.0 | `current_robot_base_angle_` | 回到 gimbal/odom 基准；注释明确控制板处理 gimbal→chassis |

所以：

- 1.1 不是“2.0 的升级版”，而是被 2.0 回退的分支。
- 如果只根据“文件名版本号更大”判断，会误以为 1.1 比 2.0 更先进；实际相反。

---

## 7. 哪些东西在这 5 个包里没有变

为避免把旧修复错误归到新版本，特别列出：

| 项目 | 结论 |
|---|---|
| `CCUTVision/src/rm_serial_driver` | 不在导航压缩包内，未能从这 5 个包审计 |
| 串口 `/dev/rm_serial`、USB 口 1-6 屏蔽、udev | 不在导航压缩包内 |
| 视觉 `CCUTVision`、相机、armor/detector/tracker | 不在导航压缩包内 |
| `pointcloud_to_laserscan` 死锁修复 | 1.0 已包含，1.0→3.0 无净变化 |
| `recovier_behavior` 的 NaN 保护 | 1.0 已包含，后续版本无净变化 |
| `nav2_params.yaml` 的 GICP 限幅/翻译官高度/强度 | 1.0 已包含，后续版本无净变化 |
| 真实地图 `lab6/lab8/lab9` 文件本身 | 这 5 个包的实车地图没有随版本变化 |
| 虚拟裁判 mode 4 | 1.0 已包含，后续版本无净变化 |

因此，如果以后发现“串口/视觉/USB”问题，不应拿这 5 个导航包的版本差异来解释；需要单独对比 `CCUTVision` 的备份。

---

## 8. 实车验证证据

### 8.1 2026-10-19 18:36–18:55 的决策树长跑

从 `~/.ros/log` 中 `component_container_isolated_202711_1792406144637.log` 统计：

- 运行时间：约 19 分钟（18:36–18:55）。
- `Begin navigating from current location`：10 次。
- `Goal succeeded`：10 次。
- `Goal failed`：0 次。
- 巡逻/回家往返：5 次完整循环。
- `GICP accepted`：2305 次。
- `GICP did not converge`：23 次。
- `GICP rejected`：0 次。

同一时段的 `rm_behavior_tree_207329_1792406184059.log` 显示：

- `PatrolGoal` 和 `HomeGoal` 交替下发；
- HP 低于 300 时进入回家；
- 日志尾部 HP 从 354 回到 412 的过程中一直保持 `state=LOW`，没有在 351 立刻切回巡逻；本场在 HP=412 时由人工结束，所以没有完整跑到 580。

结论：**3.0 的决策树 + 2.1 的 TF 发布 + 2.0 的速度链，在这一次长跑中形成了完整闭环。**

### 8.2 这次长跑不能证明的事情

同一场日志仍有少量：

```text
Exception in transformPose: Lookup would require extrapolation into the future ...
```

它们没有导致本场 10 个目标失败，但说明：

- TF 时间戳边界抖动仍然存在；
- 目前是“目标能成功”的工程状态，不是“TF 零告警”的理论完美状态。

另外，这次长跑没有持续记录 `map→odom` 的 20 Hz 探针，所以不能把“10 个目标成功”单独等价为“所有时段都没有冻结”。它证明的是当前链路在这一次实车场景下可用。

---

## 9. 仍然存在的风险

1. **窄门/膨胀层**
   - 目标点通过窄门时，门框可能被当作障碍；机器人半径、膨胀半径、门宽会直接影响是否卡住。

2. **距离依赖性**
   - 目标距离越远，GICP、TF、局部规划、窄门通过的累计误差越大；目前成功场次主要是中短距离往返。

3. **45 秒超时**
   - 外层 Timeout 可以防止无限 RUNNING，但如果某个目标确实需要超过 45 秒，会被取消并重试。

4. **目标点依赖地图**
   - `2.376,0.066` 和 `0,0` 必须与当前实际地图一致；换地图后必须重新核对，不能沿用。

5. **TF 边界抖动**
   - 18:36 长跑仍有 future extrapolation 告警；目前不影响成功，但需要继续观察长时间运行。

6. **包内无 git 历史**
   - 这 5 个包主要靠文件 CRC、内部 mtime 和 `.bak-*` 还原，不是完整 Git 历史。以后备份建议保留 `git diff` 或生成 manifest。

---

## 10. 建议的版本选择

| 场景 | 建议 |
|---|---|
| 实车跑当前导航/决策树 | 用 3.0 |
| 只排查速度方向/底盘角 | 对比 1.1 和 2.0 的 `fake_vel_transform` |
| 排查 GICP 局部定位/TF 冻结 | 对比 2.0 和 2.1 的 `small_gicp` |
| 排查决策树回家/巡逻/窄门 | 用 3.0 的 `full_decision.xml` 和 `send_goal.cpp` |
| 排查串口/USB/视觉 | 不要用这 5 个导航包，另做 `CCUTVision` 备份审计 |

最终建议：

```text
当前保留：3.0
1.0：历史基线
1.1：方向实验记录，不建议独立使用
2.0：速度链时间戳修复节点
2.1：map→odom 发布稳定节点
3.0：决策树/到点判定最终版
```

---

## 附录 A：源包内关键文件时间

| 版本 | `fake_vel_transform.cpp` | `omni_pid_pursuit_controller.cpp` | `small_gicp_relocalization.cpp` | `full_decision.xml` | `send_goal.cpp` |
|---|---|---|---|---|---|
| 1.0 | 2026-10-16 18:25:48 | 2026-03-16 11:01:58 | 2026-09-22 06:55:24 | 2026-10-17 12:37:56 | 2026-09-19 20:32:28 |
| 1.1 | 2026-10-17 13:16:12 | 2026-03-16 11:01:58 | 2026-09-22 06:55:24 | 2026-10-17 12:37:56 | 2026-09-19 20:32:28 |
| 2.0 | 2026-10-17 19:35:32 | 2026-10-18 08:22:44 | 2026-09-22 06:55:24 | 2026-10-17 12:37:56 | 2026-09-19 20:32:28 |
| 2.1 | 2026-10-17 19:35:32 | 2026-10-18 08:22:44 | 2026-10-18 12:49:12 | 2026-10-17 12:37:56 | 2026-09-19 20:32:28 |
| 3.0 | 2026-10-17 19:35:32 | 2026-10-18 08:22:44 | 2026-10-19 18:30:02（内容同 2.1） | 2026-10-19 18:31:56 | 2026-10-19 15:47:54 |

## 附录 B：复现审计

工作区已解出文本源码：

```text
/home/robot/Documents/Codex/2026-09-26/new-chat/outputs/qy_versions/
```

相邻版本 diff：

```bash
diff -ruN --exclude='*.bak-*' --exclude='*.bak' \
  outputs/qy_versions/1.0 outputs/qy_versions/1.1

diff -ruN --exclude='*.bak-*' --exclude='*.bak' \
  outputs/qy_versions/1.1 outputs/qy_versions/2.0

diff -ruN --exclude='*.bak-*' --exclude='*.bak' \
  outputs/qy_versions/2.0 outputs/qy_versions/2.1

diff -ruN --exclude='*.bak-*' --exclude='*.bak' \
  outputs/qy_versions/2.1 outputs/qy_versions/3.0
```

注意：`diff -ruN` 会显示文件内容差异；如果要审计全部条目（包括未解压的二进制资源），应回到 zip 层按 CRC/SHA-256 比较，本文的净差异表就是这样得到的。
