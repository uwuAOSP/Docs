# Moment

Moment 是 Android 任务级小窗功能。应用继续运行于原有任务和 Activity 栈，WindowManager 使用任务 Surface 的位置、缩放、裁剪与圆角，将任务显示为可移动和缩放的小窗。

## 实现

`system_server` 中的 Moment 控制器管理窗口状态、缩放、动画、位置限制、折叠及全屏切换。SystemUI 负责导航条、通知和 MomentArc 入口；Launcher 负责最近任务手势；uwuSettingsExt 提供启用开关和应用选择界面。

设置按 Android 用户保存在 `Settings.Secure`。关闭总开关后，不再创建新的 Moment，且会将当前用户已有的 Moment 恢复到全屏。工作资料和其他用户拥有独立设置与任务状态。

Moment 保留同一任务中的 Activity 栈和返回行为；预测式返回时保持小窗外观。横屏下，窗口会根据可用区域调整默认尺寸，避免遮挡系统栏和操作区域。

## 使用

- [启动应用](./launching-apps.md)与[导航条双击](./navigation-handle.md)
- [MomentArc](./moment-arc.md)与[通知打开](./notifications.md)
- [最近任务上滑手势](./recents-gesture.md)
- [多窗口与焦点](./multiple-windows.md)、[移动与缩放](./move-and-resize.md)
- [操作菜单](./controls.md)、[折叠模式](./compact-mode.md)
- [横屏适配](./landscape.md)、[返回操作](./back.md)
- [设置与适用范围](./settings.md)、[调试命令](./debugging.md)

## 适用范围

Moment 主要面向手机上的快速多任务。当前 uwuSettingsExt 不在平板设备中显示 Moment 设置入口。

依赖固定尺寸、特定方向、全屏系统 UI、受保护内容或厂商窗口实现的应用，可能无法完整适配小窗。相机、游戏、投屏、安装器、身份验证和紧急操作等场景通常更适合全屏。

Moment 不绕过 Android 锁屏、工作资料验证、Activity 启动权限或应用自身限制。