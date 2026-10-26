# 后台管理

后台管理可为不同应用指定后台运行策略。

## 应用模式

| 模式 | 行为 |
| --- | --- |
| 默认 | 使用 Android 原有后台管理 |
| 墓碑 | 应用空闲后冻结进程，保留内存状态 |
| Full | 不冻结进程，限制 OOM 优先级并加入 Device Idle 白名单 |

策略按 Android 用户和应用包名保存。

## 墓碑冻结

应用进入空闲状态后，系统检查前台界面、服务、音频、定位及 Binder 等活动状态，再决定是否冻结。冻结期间收到 Binder 请求时，系统会解冻应用处理请求，空闲后再次冻结。

## 冻结后端

可选择**自动、CGroup1、CGroup2 或混合**模式。自动模式根据设备内核的 freezer 支持情况选择后端。

CGroup1 使用 freezer controller，CGroup2 使用 `cgroup.freeze`。Binder 冻结需要对应的内核驱动支持。

## 最近任务

「忽略启动器任务卡移除」适用于墓碑和 Full 模式。划掉最近任务卡片时可保留应用进程；手动强行停止仍会结束应用。

## 诊断

设置页支持导出后台管理日志，包括应用策略、冻结后端、cgroup 和 Binder 状态以及 framework 事件。

墓碑实现参考 [Cirno](https://github.com/Freezer-Team/Cirno)。
