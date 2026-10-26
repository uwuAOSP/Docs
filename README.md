# 设备维护者信息

设备维护者信息显示在 **设置 > 系统 > 软件更新** 中，由设备树提供。

## 设备树配置

在产品 `.mk` 文件继承 `vendor/uwu/config/common.mk` 前添加：

```make
UWU_MAINTAINER := example
```

构建后对应属性为：

```text
ro.uwu.maintainer=example
```

名称最多 64 个字符，支持字母、数字、空格、`.`、`_`、`-` 和 `@`。

## 使用迁移命令

设备树迁移时可通过 `--maintainer` 指定名称：

```sh
uwu product migrate lineage apply \
  --device nabu \
  --type tablet \
  --no-telephony \
  --maintainer example
```

交互模式也支持填写维护者；`plan` 或 `--dry-run` 可预览更改。

## 显示

系统读取 `ro.uwu.maintainer` 显示维护者名称，未设置时隐藏该项目。