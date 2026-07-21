# uwuBackGroundManager

uwuBackGroundManager 是一个按应用管理后台策略的系统功能，可冻结空闲应用或提高应用的后台存活能力。

## 设备运行要求

- 内核支持 cgroup freezer
- Android-Binder 驱动支持进程冻结
- Android 用户空间 freezer 已启用，并且能够访问 freezer cgroup 层级

## 工作原理

应用模式按用户保存在 `Settings.Secure` 中。配置在应用重启和设备重启后仍然保留。`system_server` 监听配置变化，并将模式应用到该应用 UID 下的全部进程。

控制器会跟踪应用可见状态、音频播放、录音、定位监听、VPN 连接和 Binder 活动。这些状态会暂时阻止应用被冻结。保护状态结束后，墓碑模式会重新安排冻结。

墓碑模式会在应用空闲约 3s 后冻结符合条件的进程。音频停止后等待约 6s。进程存在可见 Activity、前台服务、正在接收广播、正在执行服务、Instrumentation、显式 CPU 能力或 AOSP freezer 豁免时不会被冻结。

被冻结的墓碑应用收到 Binder 请求时，uwuBackGroundManager 会临时解冻整个应用 UID，将不会走 AOSP 默认的冻结进程终止逻辑。Binder 空闲约 3s 后，该 UID 可以再次被冻结。

Full 模式不使用 freezer。它会将进程的 OOM 调整值限制在“可感知应用”级别，并把应用包名和 app ID 加入 Device Idle 白名单。这样可以减少进程回收和 Doze 限制，但不能保证应用永远不被终止。

可选的“忽略启动器任务卡移除”只对墓碑或 Full 模式的应用生效。从最近任务划掉应用时，启动器列表中的任务卡会消失，但任务、页面状态和进程会保留。强行停止、应用崩溃、严重内存压力或应用主动退出仍会结束进程。

## 内核侧所需支持

### 墓碑模式必需项

- `CONFIG_CGROUP_FREEZER=y`
  提供暂停和恢复应用进程所需的 cgroup freezer。当前使用的 cgroup 层级必须向 Android 用户空间提供可写的 `cgroup.freeze` 接口。

- `CONFIG_ANDROID_BINDER_IPC=y`
  提供应用进程和系统进程使用的 Android Binder IPC 驱动。

- `BINDER_FREEZE`
  Binder UAPI 和驱动必须实现该 ioctl。Android 通过它冻结目标进程的 Binder 投递，使 Binder 状态与 cgroup 冻结状态保持一致。

- `BINDER_GET_FROZEN_INFO`
  Binder UAPI 和驱动必须实现该 ioctl，并报告进程冻结期间收到的同步和异步事务。framework 依赖这些状态，在出现 Binder 活动时安全解冻墓碑应用。

- Binder 冻结事务跟踪
  驱动必须跟踪冻结期间待处理的同步事务和异步流量。只在头文件中定义 ioctl 编号不够，还需要对应的驱动实现。

- 用户空间与内核 Binder 接口一致
  内核中的 ioctl 结构体和命令编号必须与调用它们的 Android 用户空间匹配。

Full 模式不需要专用内核hook

### 墓碑模式

保留内存状态，但空闲时不继续占用 CPU。

- 保护延时结束后冻结符合条件的后台进程
- 进程存活期间保留内存和应用状态
- 音频、录音、定位、VPN、可见状态和 Binder 活动会临时唤醒或保护应用
- 内存压力较高时，应用仍可能被系统回收
- 前台进程或明确豁免的进程可能不会被冻结

> Tip：离开聊天应用后，符合条件的进程会被冻结。收到 Binder 事件时应用临时解冻并处理事件，空闲后再次冻结。

### Full

适合需要持续执行后台任务的应用。

- uwuBackGroundManager 不会冻结该应用
- 进程 OOM 优先级至少提高到“可感知应用”级别
- 应用加入 Device Idle 白名单，减少 Doze 限制
- 在其他 Android 权限和限制允许的范围内，应用可以继续执行后台任务
- 应用仍可能主动退出、崩溃、被强行停止，或在严重内存压力下被终止

> Tip：下载器可以继续执行后台任务，并获得比默认模式更强的进程保留能力。

### 默认

默认模式会移除该应用的 uwuBackGroundManager 策略，恢复 Android 原生进程管理。

### 感谢

非常感谢Cirno项目给本功能的启发：https://github.com/Freezer-Team/Cirno.git
