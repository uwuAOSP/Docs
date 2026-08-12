# Moment

Moment 是 uwuAOSP 的轻量多任务窗口。它可以把应用任务转换为可移动、可缩放的小窗，并提供多种快速启动和恢复方式。

## How It Works

Moment 在 Android WindowManager 中使用独立的窗口模式。应用仍然运行在原来的 Android 任务中，系统通过任务 Surface 的缩放、裁剪、圆角和位置变换，把任务显示为 Moment，而不是在应用内部模拟一层悬浮界面。

`system_server` 中的 Moment 控制器负责窗口状态、动画、位置约束、缩放、折叠模式和全屏转换。SystemUI 负责导航条、通知和 MomentArc 等入口，Launcher 负责最近任务手势，uwuSettingsExt 提供总开关、入口设置和应用选择页面。

Moment 设置按用户保存在 `Settings.Secure` 中。关闭总开关会停止新的 Moment 启动，并将该用户已有的 Moment 恢复到全屏。工作资料等其他用户拥有独立设置和任务状态。

应用内容仍由应用自身绘制。Moment 会保留同一任务内的 Activity 栈、返回行为和页面状态，并为跨 Activity 的预测式返回动画保持小窗圆角。横屏时，系统会根据可用空间使用更小的默认比例，并约束窗口避免遮挡系统栏和操作区域。

## 功能

- [启动应用](./launching-apps.md)：从 Moment 设置或应用列表直接启动小窗。
- [导航条双击](./navigation-handle.md)：将当前应用转换为 Moment，或从主屏幕打开 Moment 应用列表。
- [MomentArc](./moment-arc.md)：从屏幕底角快速选择应用或快捷方式。
- [通知打开](./notifications.md)：点按通知进入 Moment，并保留全屏打开按钮。
- [最近任务上滑手势](./recents-gesture.md)：继续上滑，将当前任务以 Moment 打开。
- [多窗口与焦点](./multiple-windows.md)：同时保留多个 Moment，并在应用之间切换。
- [移动与缩放](./move-and-resize.md)：拖动窗口并从四角调整大小。
- [操作菜单](./controls.md)：全屏、关闭和进入折叠模式。
- [折叠模式](./compact-mode.md)：把 Moment 缩小、吸边、隐藏或拖动关闭。
- [横屏适配](./landscape.md)：在横屏下使用适合可用空间的尺寸和布局。
- [返回操作](./back.md)：使用底部操作条返回，并支持预测式返回动画。
- [设置与适用范围](./settings.md)：总开关、入口开关、方向选项和设备范围。
- [调试命令](./debugging.md)：通过 `wm moment` 查看和控制 Moment。

## 适用范围

Moment 主要面向手机上的快速多任务。uwuSettingsExt 当前不会在平板设备上显示 Moment 设置入口。

部分应用依赖固定尺寸、特定屏幕方向、全屏系统 UI、受保护内容或厂商窗口行为，可能无法在 Moment 中获得完整体验。相机、游戏、投屏、安装器、身份认证和紧急操作等场景通常更适合全屏使用。

Moment 不会绕过 Android 的锁屏、工作资料验证、Activity 启动权限或应用自身限制。
