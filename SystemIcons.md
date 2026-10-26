# 系统小图标

系统小图标提供「默认」和「PUI」两组系统资源样式。样式切换通过 Android Runtime Resource Overlay（RRO）完成。

## 选择样式

进入 **uwuAOSP Plus > 界面设置 > 系统小图标**。

| 选项 | 行为 |
| --- | --- |
| 默认 | 关闭自定义覆盖层，恢复各组件原始图标资源 |
| PUI | 启用已预装的 PUI 风格图标覆盖层 |

系统通过一次 `OverlayManager` 事务切换兼容的 RRO，并保存所选样式。Settings、SystemUI 和 Launcher 等已启动界面可能需要重新打开或重新启动相应进程，才能全部刷新。

## 实现与兼容性

PUI 资源位于 `vendor/uwu/overlay/rro_packages/PUI`，随系统源码编译为 RRO。设置页仅控制设备中已安装的覆盖层，不下载或执行第三方 APK。

没有包含 PUI 资源的产品，以及 `OverlayManager` 拒绝应用覆盖的情况，不会强制切换；设置页显示失败原因，原有样式保持不变。

PUI 图标来源于 **PUI Theme For Stock Android v17.0.218**，原作者为**天伞桜**。项目将其整理为随系统编译的资源覆盖。