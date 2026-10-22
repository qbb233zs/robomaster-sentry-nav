# 建图一直失败的根因：pointcloud_to_laserscan（翻译官）进程内 tf2 死锁

日期：2026-10-13 晚。结论已用"隔离域台架复现 + gdb 锁主实证"验证。

## 一句话结论
翻译官的进程里有两个线程（点云执行器线程 / TF 监听线程），在 tf2/tf2_ros 0.25.23（Humble jammy）内部
**两把锁互相等待（AB-BA 死锁）**：一旦踩中，进程还活着、CPU 也在耗，但 `obstacle_scan` 永久断流
→ slam_toolbox 地图冻结 → 人继续走、点云继续飘 → 看起来就是"建图失败"。
**死锁不会自愈，只有重启进程才能恢复**（与生产现象完全一致）。

## 死锁双方的调用链
- 线程A（点云执行器，主线程）：
  `MessageFilter::add → tf2_ros::Buffer::waitForTransform`（持 `timer_to_request_map_mutex_`）
  `→ tf2::BufferCore::addTransformableRequest`（等 `transformable_requests_mutex_`，卡死）
- 线程B（TransformListener 自带线程）：
  `TF → setTransformImpl → BufferCore::testTransformableRequests`（持 `transformable_requests_mutex_`）
  `→ 请求完成回调（libtf2_ros）`（等 `timer_to_request_map_mutex_`，卡死）

## 锁主实证（2026-10-13 19:13:21，进程未杀）
- TF 线程等的锁 0x...2ee0：owner = 18（主线程 TID）
- 主线程等的锁 0x...2e30：owner = 29（TF 线程 TID）
- 两把锁地址差恒为 0xB0 = Buffer+0x210（`timer_to_request_map_mutex_`）与 BufferCore+0x150
  （`transformable_requests_mutex_`）之差；BufferCore 是 tf2_ros::Buffer 的基类子对象（Buffer+0x10）。
- 现场文件：`deadlock_owner_191321.txt`（本目录）；另有 5 份台架全线程栈见 /tmp/rig_captures/*（签名一致）。

## 触发条件与规律
- 必要条件：`target_frame` 非空（=base_footprint）→ 翻译官走 MessageFilter/futures 路径；
  TransformListener 自带 spin 线程（默认 spin_thread=true）。
- 竞态概率随：点云频率/单帧点数、TF 频率、TF 短暂断供或堆积而升高。不需要 TF 断供也能触发。
- 台架复现：10 万点/20Hz 下最快"开跑 3 秒内"就中；30k 点基线 10 分钟 0 次；生产实例寿命随机 0.4–28 分钟。
- 现象判别：`obstacle_scan` 0Hz，而上游 point_lio/loam/terrain_ext 全部正常；重启翻译官即好。

## 修复选项（按推荐排序，都需先备份）
A. 【推荐，治根】改翻译官源码，绕开 MessageFilter/futures：
   直接订阅 `cloud_in`，回调内 `tf2_->transform(...)` 失败就丢这一帧（现有代码已自带 try/catch，20Hz 丢一帧无感）。
   改动点：`src/rm_perception/pointcloud_to_laserscan/src/pointcloud_to_laserscan_node.cpp`
   构造函数 84–95 行附近（target_frame 非空时不再创建 MessageFilter，仍保留 TransformListener）。
   效果：进程内不再调用 waitForTransform → 死锁两个环都不存在。
B. 【保命，立即见效】看门狗常开（已有 `p2l_auto_watchdog.py` + `p2l_restart.sh`），
   建图/比赛档期自动盯 `obstacle_scan`，静默 10s 就重启翻译官；代价是建图最多中断 10 秒。
C. 【不紧急】TransformListener 改 spin_thread=false 与节点单线程化（串行执行），可消死锁但可能增加 TF 延迟。
D. 【不建议】升级/重编系统 tf2_ros（影响面大，赛前风险高）。

## 复现办法（隔离域，不碰正式系统）
RIG_DOMAIN=80 bash outputs/p2l_repro/run_rig.sh 300 --points 100000
