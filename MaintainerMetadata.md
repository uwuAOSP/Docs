# 设备维护者信息

设备树可以声明当前产品的维护者。有效配置写入只读系统属性 `ro.uwu.maintainer`，显示在 **设置 > 系统 > 软件更新** 的应用更新项目下方。

## 设备树配置

在产品 `.mk` 文件继承 `vendor/uwu/config/common.mk` **之前**指定维护者：

```make
UWU_MAINTAINER := example
```

构建后对应属性为：

```text
ro.uwu.maintainer=example
```

维护者名称允许字母、数字、空格、点、下划线、连字符和 `@`，最多 64 个字符。未设置或值为空时，不生成该属性，设置中也不会显示空白项目。

## 使用迁移命令

设备树迁移时可通过 `--maintainer` 指定名称：

```sh
uwu product migrate lineage apply \
  --device nabu \
  --type tablet \
  --no-telephony \
  --maintainer example
```

交互模式也会询问维护者名称。使用 `plan` 或 `--dry-run` 可以在写入设备树之前检查结果。

## 显示与更新

软件更新页面读取 `ro.uwu.maintainer`，根据系统语言显示「维护者」或「Maintainer」。

该字段仅用于展示，不参与 OTA 身份校验、固件签名或更新渠道选择。