# uwuBackGroundManager

[English](./README.md) | **简体中文**

uwuBackGroundManager 按应用管理后台行为。它可以冻结空闲应用，也可以提高需要持续工作的应用的后台保留级别。

## 应用模式

- **默认**：不添加 uwuAOSP 策略，使用 Android 原生后台管理。
- **墓碑模式**：应用空闲后冻结进程，保留内存状态并停止 CPU 执行。
- **Full**：不冻结应用，将 OOM 优先级限制在“可感知应用”级别，并加入 Device Idle 白名单。

策略按用户和包名保存在 `Settings.Secure`。`system_server` 监听配置变化，并把模式应用到同一 UID 下的进程。

墓碑模式会跟踪可见 Activity、前台服务、广播、正在执行的服务、Instrumentation、音频播放与录制、定位、VPN、Binder 活动和 AOSP freezer 豁免。受保护状态结束后，符合条件的应用会重新进入冻结队列。Binder 请求到达时，系统先解冻整个 UID，处理完成并再次空闲后再冻结，而不是按 AOSP 默认路径终止冻结进程。

Full 模式减少进程回收和 Doze 限制，但不能保证应用永远存活。强行停止、崩溃、主动退出和严重内存压力仍然可以结束进程。

## 冻结后端

设置页提供自动、CGroup1、CGroup2 和混合后端。系统读取 cgroup 挂载布局和 freezer 能力：

- **自动**根据设备实际布局选择可用后端。
- 不受当前内核支持的手动选项会被禁用。
- 手动选择失效或布局改变时，framework 会回退到当前可用后端；没有可用 freezer 时不会执行墓碑冻结。

墓碑模式还要求 Binder 驱动支持 `BINDER_FREEZE`、`BINDER_GET_FROZEN_INFO` 和冻结事务跟踪。只定义 ioctl 编号而没有驱动实现不够。Full 模式不需要这些冻结接口。

## 最近任务

“忽略启动器任务卡移除”只对墓碑和 Full 应用生效。从最近任务划掉卡片后，卡片会消失，但任务和进程可继续保留。强行停止仍会结束应用。

## 诊断日志

设置页可以导出后台管理日志。日志使用 `[INFO]`、`[WARN]` 和 `[ERROR]` 标注，包含构建信息、应用策略、请求与实际冻结后端、cgroup 控制器和挂载、Binder 节点、内核 freezer 状态及 framework 事件。导出只读取本机状态，不会上传文件。

## 内核要求

- CGroup1 需要可写的 freezer controller。
- CGroup2 需要可写的 `cgroup.freeze`。
- Android Binder 驱动需要与用户空间一致的冻结 UAPI。
- Android 用户空间 freezer 必须启用并可访问对应 cgroup 层级。

该功能受 [Cirno](https://github.com/Freezer-Team/Cirno.git) 启发。
