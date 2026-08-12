# 调试命令

开发者可以通过 `adb shell wm moment` 查看或控制 Moment。

```text
adb shell wm moment enable
adb shell wm moment disable
adb shell wm moment status
adb shell wm moment list
adb shell wm moment start [--user USER_ID] PACKAGE/CLASS
adb shell wm moment convert [TASK_ID]
adb shell wm moment stop TASK_ID
adb shell wm moment fullscreen TASK_ID
adb shell wm moment close-all
adb shell wm moment set-scale SCALE
```

- `enable` / `disable`：临时启用或停用 Moment。
- `status`：显示开关、默认比例和活动任务数量。
- `list`：列出当前 Moment 任务。
- `start`：以 Moment 启动指定 Activity。
- `convert`：转换指定任务；省略任务 ID 时转换当前聚焦任务。
- `stop` / `fullscreen`：将指定 Moment 恢复到全屏。
- `close-all`：将所有 Moment 恢复到全屏。
- `set-scale`：设置调试用默认缩放比例。

这些命令面向开发和调试，不替代用户设置页面。部分命令的状态不会作为持久用户配置保存。
