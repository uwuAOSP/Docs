# 系统小图标

系统小图标用于切换系统界面的图标资源。目前提供系统默认图标和 PUI 两种样式。

## 使用

进入 **uwuAOSP Plus → 界面设置 → 系统小图标**，选择：

- **默认**：关闭 uwuAOSP 提供的图标覆盖，恢复各组件原始资源。
- **PUI**：启用 PUI 风格的系统图标。

选择后，系统通过一次 OverlayManager 事务批量切换所有兼容的资源覆盖，并保存当前选择。设置、SystemUI、Launcher 等正在运行的界面可能需要重新打开或重启对应进程后才会全部刷新。

## 工作原理

PUI 图标以源码形式放在 `vendor/uwu/overlay/rro_packages/PUI`，随 ROM 编译为可变 RRO。设置页只控制这些已安装覆盖层，不会下载、安装或执行第三方 APK。

设备只会启用当前系统中实际存在且可兼容的覆盖层。如果产品没有包含 PUI 资源，或 OverlayManager 拒绝事务，页面会给出明确的应用失败提示并保留原有样式。

PUI 资源来源于 **PUI Theme For Stock Android v17.0.218**，原作者为天伞桜；uwuAOSP 将其整理为可从源码编译的系统资源覆盖。
