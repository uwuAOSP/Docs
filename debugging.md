# Moment 调试命令

可以使用 `adb shell wm moment` 查看或操作当前 Moment 状态。

```sh
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

| 命令 | 行为 |
| --- | --- |
| `enable` / `disable` | 临时启用或停用 Moment |
| `status` | 显示开关、默认比例和当前活动任务数 |
| `list` | 列出已有 Moment 任务 |
| `start` | 将指定 Activity 作为 Moment 启动 |
| `convert` | 转换指定任务；缺省任务 ID 时转换当前聚焦任务 |
| `stop` / `fullscreen` | 将指定任务恢复到全屏 |
| `close-all` | 将所有 Moment 恢复到全屏 |
| `set-scale` | 设置调试用默认缩放比例 |

