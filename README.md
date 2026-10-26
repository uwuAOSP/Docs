# Kotj

Kotj 是随系统预装的本地备忘录应用，使用 Kotlin、Jetpack Compose 和 Material 3。包名为 `com.lopleec.kotj`。

## 系统集成

`vendor/uwu/config/common.mk` 将 `Kotj` 加入通用产品包，并从源码构建。

单独构建模块：

```sh
uni Kotj
```

## 功能

- 富文本、标题、列表、复选框、表格和图片
- 分类、置顶、搜索、最近删除和自动清理
- 导入 TXT、Markdown、RTF、DOCX
- 导出纯文本、Markdown、DOCX
- 以独立密码或 Android 系统身份验证保护加密笔记

## 数据与加密

Kotj 不申请网络权限，笔记和附件保存在本机。

独立密码模式通过 PBKDF2-HMAC-SHA256 派生密钥，使用 AES-GCM 加密笔记和附件。系统身份验证模式使用 Android Keystore。加密页面会阻止截图和最近任务预览。

**加密笔记无法在忘记密码或清除应用数据后恢复。** 刷机前请备份重要笔记。

源码及使用说明：[UwUniverse/Kotj](https://github.com/UwUniverse/Kotj)。