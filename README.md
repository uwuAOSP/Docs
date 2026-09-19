# Kotj

Kotj 是 uwuAOSP 随系统编译的本地备忘录应用。它使用 Kotlin、Jetpack Compose 和 Material 3，包名为 `com.lopleec.kotj`。

## 系统集成

`vendor/uwu/config/common.mk` 将 `Kotj` 加入通用产品包，因此采用 uwuAOSP 通用配置的手机、平板和折叠设备都会从源码构建并预装它。它不是下载的预编译 APK。

只编译 Kotj：

```sh
uni -j$(nproc) Kotj
```

## 功能

- 富文本、标题、列表、复选框、表格和图片。
- 分类、置顶、搜索、最近删除和自动清理。
- 导入 TXT、Markdown、RTF、DOCX。
- 导出纯文本、Markdown、DOCX。
- 独立密码或 Android 系统身份验证保护加密笔记。

## 数据与隐私

Kotj 不申请网络权限。笔记、附件、分类和设置保存在应用私有空间中，不上传遥测或笔记内容。

密码模式使用 PBKDF2-HMAC-SHA256 派生 AES-256 密钥，并以 AES-GCM 加密笔记和附件。系统解锁模式使用 Android Keystore 包装随机密钥。加密内容打开时会阻止截图和最近任务预览。

忘记独立密码、丢失系统解锁密钥或清除应用数据后，加密笔记无法恢复。ROM 更新不会主动清除应用数据，但刷机前仍应导出重要内容。

## 上游

Kotj 的独立源码和完整使用说明位于 [UwUniverse/Kotj](https://github.com/UwUniverse/Kotj)。uwuAOSP 只负责系统构建集成，不改变其本地存储与加密边界。
