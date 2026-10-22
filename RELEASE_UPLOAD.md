# 上传清单与发布步骤

## 1. 上传到 Git 仓库

把本目录中的以下内容上传到 GitHub 仓库根目录：

- `README.md`
- `.gitignore`
- `docs/`
- `maps/`
- `media/`
- `RELEASE_UPLOAD.md`

不要上传 ZIP、MP4、PCD 等大文件到普通 Git 历史。

## 2. 创建 GitHub Release

进入仓库页面：

```text
Releases -> Draft a new release
```

- Tag：`v3.0`
- Title：`哨兵导航 v3.0 - 建图、导航与决策树联调`
- 只上传以下附件：
  - `robomaster-sentry-nav-v3.0-public.zip`
  - `robomaster-sentry-nav-v3.0-public.zip.sha256`
  - 演示视频（如果有）

不要上传：

- `nav-master-ccut-QY-newcccutNav2-3.0-clean.zip`
- 其他版本的导航压缩包
- `build/`、`install/`、`log/` 或原始 PCD/视频大文件

## 3. Release Notes

```markdown
## 主要变化

- 修复建图翻译官 tf2 死锁，`obstacle_scan` 不再静默归零。
- 修复 `map -> odom`、local plan 和控制器输出的时间戳问题。
- small-GICP 使用独立发布线程并增加结果互斥保护。
- 决策树支持低血回家、回血恢复巡逻、45 秒超时、失败重试和近目标到达判定。
- 增加实验室 `lab9` 二维地图、三维点云和完整启动文档。

## 附件

- `robomaster-sentry-nav-v3.0-public.zip`：v3.0 公开源码快照，包含导航源码、决策树、串口桥和依赖接口。
- `robomaster-sentry-nav-v3.0-public.zip.sha256`：压缩包 SHA256 校验值。
```

## 4. 校验

上传完成后，在下载目录执行：

```bash
sha256sum -c robomaster-sentry-nav-v3.0-public.zip.sha256
```

应显示：

```text
robomaster-sentry-nav-v3.0-public.zip: OK
```

## 说明

本仓库是公开仓库。不要上传真实账号、密码、Token、比赛密钥或队伍内部敏感资料。雷达配置中的内网 IP 只作为现场配置参考。
