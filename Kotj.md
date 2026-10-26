# Kotj

Kotj 是随系统从源码构建的本地备忘录应用，使用 Kotlin、Jetpack Compose 和 Material 3。应用包名为 `com.lopleec.kotj`。

## 系统集成

`vendor/uwu/config/common.mk` 在通用产品包中包含 `Kotj`。采用该配置的手机、平板和折叠设备都会构建并预装 Kotj，不使用预编译 APK。

单独构建模块：

```sh
uni -j$(nproc) Kotj
```

`-j` 的数值应根据构建主机内存和处理器数量调整。

## 功能

- 富文本、标题、列表、复选框、表格和图片
- 分类、置顶、搜索、最近删除和自动清理
- 导入 TXT、Markdown、RTF、DOCX
- 导出纯文本、Markdown、DOCX
- 以独立密码或 Android 系统身份验证保护加密笔记

## 数据与加密

Kotj 不申请网络权限。笔记、附件、分类及设置保存在应用私有空间中，不上传笔记内容或遥测。

独立密码模式使用 PBKDF2-HMAC-SHA256 派生 AES-256 密钥，通过 AES-GCM 加密笔记和附件。系统身份验证模式使用 Android Keystore 包装随机密钥。打开加密内容时会阻止截图和最近任务预览。

忘记独立密码、丢失系统解锁密钥或清除应用数据后，加密笔记无法恢复。系统更新通常保留应用数据，但刷机前仍应自行导出重要内容。

Kotj 的应用源码和详细使用说明见 [UwUniverse/Kotj](https://github.com/UwUniverse/Kotj)。本系统仅负责其构建和预装集成。