# CCUTVision 串口 / 导航 对接说明（给导航 + 电控）

来源：`/home/robot/CCUTVision`（只读分析，未修改任何文件）
字段偏移由 `g++ -std=c++14` 实测 `sizeof/offsetof` 得到（x86-64 + `__attribute__((packed))`）。

---

## 一、结论速览

1. 全仓库只有 **一个** 包跟电控打交道：`src/rm_serial_driver/`。视觉数据、导航速度、下位机姿态、裁判系统数据全部走这**一条串口**。
2. 导航在本仓库里只有 **一处落点**：串口驱动订阅 `/red_standard_robot1/cmd_vel`，打包成 `label = 2` 的 Twist 帧发给电控（`rm_serial_driver.cpp:72`、`:299`）。
3. 本仓库 **没有** nav2 / SLAM / 定位 / 代价地图 / 路径规划代码。`armor_tracker/Planner` 是「击打策略」(平移/小陀螺跟随)，不是导航规划，别被名字骗了。
4. 视觉节点有**看门狗**：1 秒收不到电控的视觉帧就自杀重启 → 电控必须持续（>1Hz）发姿态帧，这是硬性约定。

---

## 二、串口相关文件清单

| 文件 | 作用 |
| --- | --- |
| `src/rm_serial_driver/include/rm_serial_driver/packet.hpp` | **协议定义，对接电控必发这份** |
| `src/rm_serial_driver/src/rm_serial_driver.cpp` | 收发主逻辑、CRC 拼帧、看门狗、重开串口 |
| `src/rm_serial_driver/include/rm_serial_driver/rm_serial_driver.hpp` | 节点类接口、订阅/发布声明 |
| `src/rm_serial_driver/src/crc.cpp` + `include/crc.hpp` | CRC16（裁判系统同款，初值 0xFFFF，小端追加） |
| `src/rm_serial_driver/config/serial_driver.yaml` | 波特率等参数（注意 device_name 实际不生效） |
| `src/rm_serial_driver/launch/serial_driver.launch.py` | 单节点启动 |
| `src/rm_vision-main/.../config/node_params.yaml` | `/serial_driver` 段参数（timestamp_offset 0.0057 等） |
| `src/rm_vision-main/.../launch/vision_bringup.launch.py` | 总启动，会延迟 1.5s 拉起串口节点 |

## 三、物理层参数

| 项 | 值 | 来源 |
| --- | --- | --- |
| 设备名 | `/dev/rm_serial`（**代码强制写死**，udev 绑定） | `rm_serial_driver.cpp:342-346` |
| 波特率 | 115200 | `node_params.yaml:13` |
| 数据位/校验/停止位 | 8 / NONE / 1（8N1） | `getSerialParams()` |
| 流控 | none | 同上 |
| 帧头 | `0xAA 0x55` | `packet.hpp:5` |

> `serial_driver.yaml` 里写的 `/dev/ttyUSB0` 被代码覆盖，以 `/dev/rm_serial` 为准。

## 四、帧格式（所有包统一）

```
AA 55 | label(1B) | data_length(2B 小端) | data(data_length B) | CRC16(2B 小端)
```
- 整帧长度 = `5 + data_length + 2`；`data_length = sizeof(结构体) - 7`
- CRC16 校验范围：整帧去掉最后 2 字节；初值 `0xFFFF`，查表法
- 浮点：IEEE754 float32，小端；结构体全部 `packed`（电控端必须 `__packed` / `#pragma pack(1)`）

### 4.1 视觉 → 电控：自瞄帧（label = 1，共 33 字节，data_length = 26）

| 偏移 | 字段 | 类型 | 说明 |
| --- | --- | --- | --- |
| 0-1 | header | u8×2 | `AA 55` |
| 2 | label | u8 | `1` |
| 3-4 | data_length | u16 | `26` |
| 5 | tracking | u8 | 是否跟稳 |
| 6 | bit0-2 id / bit3-5 armors_num / bit6-7 reserved | u8 | id: 0=outpost 6=guard 7=base... |
| 7-10 | yaw | f32 | 发弹解算 yaw（弧度，云台系） |
| 11-14 | pitch | f32 | 发弹解算 pitch（弧度） |
| 15-18 | fire_yaw | f32 | 开火用 yaw |
| 19-22 | v_yaw | f32 | 目标旋转角速度 rad/s |
| 23-26 | distance | f32 | 距离 m |
| 27-30 | is_fire | f32 | 1=开火 |
| 31-32 | checksum | u16 | CRC16 小端 |

### 4.2 视觉 → 电控：导航帧（label = 2，共 20 字节，data_length = 13）★ 你要用的就是这帧

| 偏移 | 字段 | 类型 | 说明 |
| --- | --- | --- | --- |
| 0-1 | header | u8×2 | `AA 55` |
| 2 | label | u8 | `2` |
| 3-4 | data_length | u16 | `13` |
| 5-8 | linear_x | f32 | 前进速度 m/s（车体系） |
| 9-12 | linear_y | f32 | 左移速度 m/s（车体系） |
| 13-16 | angular_z | f32 | 自转角速度 rad/s |
| 17 | reached | u8 | 到达标志（**语义待确认，见第七节**） |
| 18-19 | checksum | u16 | CRC16 小端 |

### 4.3 电控 → 视觉：姿态帧（label = 0x01，payload 13 字节）

| 偏移(data 内) | 字段 | 类型 | 说明 |
| --- | --- | --- | --- |
| 0 | bit0 detect_color / bit1 reset_tracker / bit6-7 reserved | u8 | 0=红 1=蓝；reset=1 重置跟踪 |
| 1-4 | roll | f32 | 云台横滚（弧度） |
| 5-8 | pitch | f32 | 云台俯仰（弧度） |
| 9-12 | yaw | f32 | 云台偏航（弧度） |

用途：广播 TF `odom → gimbal_link`，发布 `/robot_yaw`、`/robot_pitch`，跟踪节点用它判断云台是否到位（开火判据之一）。

### 4.4 电控 → 视觉：裁判系统帧（label = 0x02，payload 5 字节）

| 偏移(data 内) | 字段 | 类型 |
| --- | --- | --- |
| 0-1 | self_hp | u16 |
| 2 | game_process | u8 |
| 3-4 | remain_time | u16 |

现状：代码只打印 `Judge packet CRC ok`，**没有实际使用**（`rm_serial_driver.cpp:219`）。

## 五、时序 / 心跳 / 安全（必须和电控敲死）

1. **心跳方向：电控 → 视觉**。看门狗 1s 超时（`rm_serial_driver.cpp:432-445`），超时直接 `exit(1)` 靠 systemd 重启。所以电控必须以 ≥ 20~50Hz 持续发 `label=0x01` 姿态帧，否则整个视觉节点反复重启。
2. **导航帧发送节奏**：`/xxx/cmd_vel` 来一帧发一帧（nav2 通常 10~20Hz）。视觉端**没有**超时停车逻辑 → 建议电控端加 `cmd_vel` 超时（如 200ms 未收到即停车）。
3. `reached` 目前由速度分量是否全非零推导（见第七节风险），别当可靠的"到达"信号用。
4. 未知 label 的帧会被打印成 hex 日志（`processUnKnowPacket`），不报错，可用来在联调时确认线序通了。

## 六、导航相关文件与链路

- 唯一入口：`src/rm_serial_driver/src/rm_serial_driver.cpp:72-75`（订阅）与 `:299-327`（打包发送）
  - 话题：`/red_standard_robot1/cmd_vel`，类型 `geometry_msgs/msg/Twist`，QoS 深度 10，**硬编码，不在 yaml 里**
- 与导航间接相关的 TF：
  - `vision_bringup.launch.py:82-87` 静态发布 `odom → base_link`（零偏移）
  - `rm_serial_driver` 发布 `odom → gimbal_link`（云台姿态）
  - `armor_tracker` 用 `target_frame: odom` 做目标坐标变换
- 不相关但易混淆：`armor_tracker/src/Planner/Planner.cpp` = 击打策略（`SHOOT_TRANSLATION` / `SHOOT_TRACK`），非路径规划。

## 七、待与电控确认清单（逐条打勾）

- [ ] 串口设备：车里 `/dev/rm_serial` 对应哪个物理口？udev 规则是否已按哨兵的电控板 ID 更新？（现规则很可能是标准车的）
- [ ] 接线：TX/RX 交叉、共地、TTL 电平（3.3V/5V）
- [ ] 波特率 115200 / 8N1 / 无流控，双端一致
- [ ] 电控端结构体必须 packed + 小端 + float32
- [ ] **心跳**：电控 ≥1Hz（建议 50Hz）持续发 `label=0x01`，否则视觉节点会重启
- [ ] **`reached` 语义**：现在实现为 `vx && vy && wz 全非零 → 0，否则 → 1`，语义基本是坏的；确认电控是否使用该字节，若使用需重新定义
- [ ] 坐标系：`linear_x/linear_y/angular_z` 是否为车体系（nav2 默认 `base_link`），电控如何解算麦轮/全向轮
- [ ] **话题名**：哨兵导航栈实际发布的话题名是否为 `/red_standard_robot1/cmd_vel`；若不是（例如 `/red_sentry_robot1/cmd_vel`），视觉端订阅不到 → 车不会动
- [ ] 停车保护：电控端加 cmd_vel 超时（建议 200ms）
- [ ] 一个串口同时跑自瞄帧 + 导航帧，电控必须按 label 分流解析
- [ ] 开火标志 `is_fire` 是否与电控发射机构逻辑对齐（含安全条件）

## 八、代码里发现的现有风险（仅提示，未改动）

1. `sendDataTwist` 的 `reached` 逻辑几乎肯定是笔误（`rm_serial_driver.cpp:309-316`）。
2. `sendDataVision` 里用 `std::memcpy` 读接收线程的 `m_rx_buffer` 打日志（`:266-284`），存在数据竞争，也可能打印到非视觉帧的数据；仅影响日志可读性/稳定性。
3. `unGeneralPacket` 的帧同步：CRC 失败时只丢弃 1 个字节再重找（`:148`），属于常规保守做法，但无长度上限保护（`data_length` 未做合法性校验，理论上有畸形帧导致长时间等待的风险）。
4. 视觉 bringup 静态发布 `odom → base_link`：若导航栈/定位也发布同名 TF，会出现 TF 树冲突，联调时要确认由谁发布。

---

# 附录：导航侧（nav-master-ccut-WCZ-newcccutNav2）

来源：`/home/robot/nav-master-ccut-WCZ-newcccutNav2`（RMUL 2026 仿真导航 + 决策，未修改）

## A. 这个仓库里没有串口代码

全仓库搜索 `serial / ttyUSB / ttyACM` 只命中 Livox 雷达参数注释。
**与电控的串口交互全部在视觉仓库 `rm_serial_driver`**；导航侧只负责把速度发布成 ROS 话题 `cmd_vel`。

## B. 完整运动链路（决策 → 轮子）

```
rm_behavior_tree (决策树)
  └─ action /red_standard_robot1/navigate_to_pose   目标点: 巡逻(4.78,-2.57) / 回家(-0.22,-0.97)
       └─ nav2 bt_navigator + planner_server (global: map, base: gimbal_yaw_fake)
            └─ controller_server (ccut_mpc_controller) → 发布 cmd_vel_controller
                 └─ velocity_smoother → cmd_vel_nav2_result
                      └─ fake_vel_transform（关键：把"假底盘"速度转到真实车体）
                           └─ 发布 /red_standard_robot1/cmd_vel   ★ 与视觉串口驱动订阅的话题一致
                                └─ rm_serial_driver → 串口 label=2 帧 → 电控 → 麦轮
```

关键文件：
- `nav-master-ccut/src/rm_nav_bringup/launch/rmul2026_navigation_launch.py`（仿真总入口）
- `nav-master-ccut/src/rm_nav_bringup/launch/navigation_launch.py:299`（fake_vel_transform）、`:317`/`:382-383`（cmd_vel 重映射链）
- `nav-master-ccut/src/rm_navigation/fake_vel_transform/src/fake_vel_transform.cpp`
- `nav-master-ccut/src/rm_nav_bringup/config/simulation/nav2_params.yaml:149-158`（仿真参数）
- `nav-master-ccut/src/rm_nav_bringup/config/reality/nav2_params.yaml:153-162`（真机参数）
- `rm_decision/src/rm_behavior_tree/config/full_decision.xml`（目标点与血量分支）

## C. 三个必须确认的坑

1. **命名空间不一致（最容易导致"车不动"）**
   - 仿真入口默认 `namespace=red_standard_robot1` → 话题 `/red_standard_robot1/cmd_vel`，与视觉端匹配。
   - 真机入口 `rm_navigation_reality_launch.py:50-53` 默认 `namespace=""` → 话题变成 `/cmd_vel`，视觉串口驱动**收不到**。
   - 结论：真机启动要么显式加 `namespace:=red_standard_robot1`，要么改视觉端订阅名。

2. **`angular_z` 里已经含小陀螺转速**
   - `fake_vel_transform` 恒定给 `angular.z` 加 `spin_speed_`，初值 = `init_spin_speed: 3.14` rad/s（约 180°/s）。`cmd_spin` 话题在本仓库内**没有任何发布者**，是留给外部控制的。
   - 也就是说：串口 label=2 帧里的 `angular_z` = 导航解算 + 3.14。电控不要再自己叠加自转；要停转需要外部发 `cmd_spin=0`。

3. **`reached` 字段仍不可用**（承接第七节）
   - 由于小陀螺使 `angular_z` 恒非零，`reached` 实际退化为"linear_x 与 linear_y 是否都非零"，纯直行（vy=0）时会误报到达。

## D. TF 树对照（联调时看这个）

```
导航侧： map ──(small_gicp 或静态)── odom ──(sensor_scan_generation)── base_footprint / gimbal_yaw
                                                          └──(robot_state_publisher)── gimbal_pitch ...
        fake_vel_transform 额外发布： gimbal_yaw ──> gimbal_yaw_fake   （nav2 用的"假底盘"）
视觉侧： odom ──(串口 yaw/pitch 动态发布)── gimbal_link ──> camera_link ──> camera_optical_frame
```
两边帧名不重叠（导航没有 `base_link` / `gimbal_link`），目前不会冲突；但两棵 TF 子树并存，别把它们接到同一个 URDF 里造成重名。
