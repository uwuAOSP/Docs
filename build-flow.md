# Soong-only 构建流程

## 处理阶段

一次普通的 Soong-only 产品构建可以分为以下阶段：

```text
产品配置 Makefile
        |
        v
runMakeProductConfig
        |
        +--> soong_config.mk 导出 JSON 产品变量
        |
        v
soong_ui 选择 Soong-only
        |
        v
soong_build 分析 Android.bp 和生成的模块
        |
        +--> uwu_kernel 生成 kernel、DTB、DTBO、headers、modules
        |
        +--> fsgen 生成 filesystem、boot、super、vbmeta 模块
        |
        v
Ninja 执行构建动作
        |
        v
android_device 复制镜像和安装文件到 PRODUCT_OUT
```

## 跳过 Kati 的位置

产品配置完成后，Soong-only 会跳过 Kati 的两个主要阶段：

- `skipKati`：不运行 Kati 生成常规 Make 构建目标；
- `skipKatiNinja`：不使用 Kati 生成的 Ninja 文件作为主构建输入。

`soong_ui` 仍会运行产品配置，因为 Soong 需要从产品 Makefile 获得设备、分区、AVB
和产品包变量。跳过的是目标生成，不是产品变量读取。

这样做是为了避免每次构建重新分析和转换 Android.mk。按照 Soong 的说明，
Soong-only 的目标是将构建分析时间降低到约一半并提高开发效率。实际编译时间仍取决
于 Ninja 依赖图、源码变化、缓存命中率和主机资源。

## 产品配置阶段

`build/soong/ui/build/build.go` 仍先调用产品配置。产品配置阶段会解析：

- `PRODUCT_SOONG_ONLY`；
- `PRODUCT_PACKAGES` 和 `PRODUCT_COPY_FILES`；
- `BoardConfig.mk` 中的分区、AVB、boot 和 kernel 变量；
- `SOONG_CONFIG_*` 命名空间变量。

因此，Soong-only 产品仍必须保持产品 Makefile 可解析。Soong-only 主要替代的是
Make 生成构建目标的阶段，而不是产品配置语言本身。

## 模式选择

`build/soong/ui/build/config.go` 处理三种入口：

| 入口 | 作用 |
| --- | --- |
| `PRODUCT_SOONG_ONLY := true` | 产品默认使用 Soong-only |
| `SOONG_ONLY=true` | 环境变量请求 Soong-only |
| `--soong-only` | 当前构建请求 Soong-only |

Soong-only 模式会设置 `skipKati` 和 `skipKatiNinja`。产品配置已经读取的 Make 变量
仍会被用于生成 Soong 配置和确定目标设备。

## Make 到 Soong 的配置桥接

`build/make/core/soong_config.mk` 将 BoardConfig 和产品变量写入 Soong 的产品变量
JSON。与镜像和内核相关的内容主要包括：

- `PartitionQualifiedVariables`：每个分区的构建开关、文件系统、大小和 AVB 参数；
- `BoardUsesSoongKernel`：是否使用 Soong kernel；
- `BoardAvbMakeVbmetaImageArgs`：vbmeta 的额外参数；
- `BoardKernelPagesize`：boot/DTBO 相关 page size；
- `PartitionVarsForSoongMigrationOnlyDoNotUse`：迁移期间仍由 Make 提供的产品变量。

这些值在 Soong 侧通过 `android.Config().ProductVariables()` 读取。设备代码不应直接
读取 `out/soong/.intermediates` 中的文件来建立依赖。

## Soong 模块分析

`soong_build` 分析所有相关 `Android.bp`，然后由模块的 load hook 创建派生模块。
`build/soong/fsgen/filesystem_creator.go` 中的 `filesystemCreator` 会：

1. 根据分区变量创建 filesystem 模块；
2. 创建 boot、init_boot、vendor_boot 和 vendor_kernel_boot 模块；
3. 创建 DTBO、super 和 vbmeta 模块；
4. 创建 `android_device`，汇总镜像和 target-files 依赖；
5. 在 Kati 未启用时创建传统镜像目标对应的 phony alias。

## 输出阶段

`android_device` 在 Soong-only 模式下将 filesystem 和镜像复制到
`$(PRODUCT_OUT)`，并把主设备的 all-images stamp 接到 `droidcore-unbundled`。这样
`m droid` 仍然可以构建产品所需的默认输出。

镜像 alias 例如：

```text
systemimage
bootimage
initbootimage
vendorbootimage
dtboimage
superimage
vbmetaimage
```

alias 只在对应模块和分区实际存在时创建。`vendorkernelbootimage` 等目标不能在不具备
对应分区配置时硬编码到验证脚本中。
