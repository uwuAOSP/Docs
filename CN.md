# 后台管理

`uwuBackGroundManager` 为指定应用配置后台策略，包括空闲时冻结进程，以及提高应用在后台的保留级别。

## 应用模式

| 模式 | 行为 |
| --- | --- |
| 默认 | 使用 Android 原生后台管理策略 |
| 墓碑 | 应用进入空闲状态后冻结进程，保留内存状态并停止执行 CPU 指令 |
| Full | 不冻结应用，将 OOM 优先级限制在可感知应用级别，并加入 Device Idle 白名单 |

策略以 Android 用户和包名为单位保存在 `Settings.Secure`。`system_server` 监听策略变化，并将指定模式应用到同一 UID 下的进程。

Full 可以减少部分进程回收和 Doze 限制，但不能保证应用持续运行。应用仍可能被强行停止、崩溃、主动退出，或在严重内存压力下被回收。

## 墓碑冻结

冻结前，系统检查应用的活动状态和豁免条件，包括可见 Activity、前台服务、广播、正在执行的服务、Instrumentation、音频播放与录制、定位、VPN、Binder 活动以及 Android 自带 freezer 豁免。

受保护状态结束后，符合空闲条件的应用可以再次进入冻结队列。冻结 UID 收到 Binder 请求时，系统先解冻该 UID 的进程；请求处理结束并重新空闲后，再执行冻结。此路径不会因收到 Binder 请求就直接终止冻结进程。

## 冻结后端

设置中可以选择自动、CGroup1、CGroup2 或混合后端：

- 自动模式根据 cgroup 挂载布局及 freezer 能力选择可用后端。
- 当前内核不支持的手动选项会被禁用。
- 手动选择的后端失效时，framework 会尝试回退到可用后端。
- 没有可用 freezer 时，不执行墓碑冻结。

### 内核条件

- CGroup1：可写的 freezer controller。
- CGroup2：可写的 `cgroup.freeze`。
- Binder：实现 `BINDER_FREEZE`、`BINDER_GET_FROZEN_INFO` 及冻结事务跟踪，且 UAPI 与用户空间匹配。
- Android 用户空间 freezer 已启用，能够访问相应 cgroup 层级。

仅在头文件中声明 Binder ioctl 编号不足以满足要求。Full 模式不依赖上述 Binder 冻结接口。

## 最近任务

「忽略启动器任务卡移除」仅对墓碑和 Full 模式生效。划掉最近任务卡片后，卡片仍会消失，但相关任务和进程可继续保留。强行停止应用不受这一设置保护。

## 诊断

设置页可以在本机导出后台管理日志，内容包括应用策略、请求及实际冻结后端、cgroup 挂载和控制器、Binder 节点、内核 freezer 状态、framework 事件及构建信息。

日志使用 `[INFO]`、`[WARN]` 和 `[ERROR]` 等等级。导出操作不自动上传数据。

本功能的墓碑设计参考 [Cirno](https://github.com/Freezer-Team/Cirno)。