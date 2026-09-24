# 从 legacy kernel build 迁移到 uwu_kernel

## 使用 uwuCLI

我们推荐使用 uwuCLI 完成初始转换：

```bash
uwu
```

请按照提示选择设备和 kernel migration。

uwuCLI 会读取现有设备配置，并尽可能转换已有的 kernel build 设置。自动生成的配置未必是最终结果。迁移完成后，请您按照以下说明检查 kernel configuration、DTB/DTBO 和 kernel modules 是否符合设备实际情况。

## Kernel

Legacy kernel build 通常使用以下变量指定 kernel：

```make
TARGET_KERNEL_SOURCE := kernel/<vendor>/<kernel>
TARGET_KERNEL_ARCH := arm64
BOARD_KERNEL_IMAGE_NAME := Image
```

迁移后，这些信息由 `uwu_kernel` 模块直接声明：

```bp
uwu_kernel {
    name: "kernel",

    kernel_dir: "kernel/<vendor>/<kernel>",
    kernel_arch: "arm64",
    image_name: "Image",
}
```

工具链、额外的 Kbuild flags 和其他特殊配置只在设备确实需要时声明。完整属性请参阅 [配置参考](configuration.md)。

## Kernel configuration

Legacy kernel build 可以通过 `TARGET_KERNEL_CONFIG`、`TARGET_KERNEL_ADDITIONAL_FLAGS` 等变量指定 kernel configuration 和额外的 build flags。

例如：

```make
TARGET_KERNEL_CONFIG := \
    gki_defconfig \
    vendor/device.config

TARGET_KERNEL_ADDITIONAL_FLAGS := \
    CONFIG_EXAMPLE=y
```

迁移后应直接表达相同的配置关系：

```bp
additional_flags: [
    "CONFIG_EXAMPLE=y",
],

config: {
    defconfig: "gki_defconfig",
    fragments: [
        "vendor/device.config",
    ],
},
```

## DTB 和 DTBO

常规设备可以直接声明对应输出：

```bp
dtb: {
    enabled: true,
    target: "dtbs",
    image_name: "dtb.img",
},

dtbo: {
    enabled: true,
    target: "dtbs",
    image_name: "dtbo.img",
    page_size: 4096,
},
```

若要使用 Qualcomm DT merge，请向 `dtb` 指定 `qcom_merge`。启用此选项时，`dtb.enabled` 和 `dtbo.enabled` 必须同时为 `true`：

```bp
dtb: {
    enabled: true,
    qcom_merge: true,
    target: "dtbs",
    image_name: "dtb.img",
},

dtbo: {
    enabled: true,
    target: "dtbs",
    image_name: "dtbo.img",
    page_size: 4096,
},
```

## Kernel modules

Kernel modules 是迁移时最需要您人工检查的部分。

Legacy kernel build 将 module 的安装位置和 load list 分散在多个 Make 变量和外部列表中，两者之间还存在交叉引用。例如设备可能同时存在：

```make
BOARD_SYSTEM_KERNEL_MODULES_LOAD
BOARD_VENDOR_KERNEL_MODULES_LOAD
BOARD_VENDOR_RAMDISK_KERNEL_MODULES_LOAD
BOOT_KERNEL_MODULES
SYSTEM_KERNEL_MODULES
```

`uwu_kernel` 不使用这些 legacy 变量。对于每个需要安装 kernel modules 的分区，您必须分别指定：

1. install list，即哪些 modules 应安装到该分区；
2. load list，即哪些 modules 需要从该分区加载。

例如：

```bp
modules: {
    enabled: true,

    system_dlkm_module_install_list: [
        "modules.include.system_dlkm",
    ],
    system_dlkm_module_load_list: [
        "modules.load.system_dlkm",
    ],

    vendor_dlkm_module_install_list: [
        "modules.include.vendor_dlkm",
    ],
    vendor_dlkm_module_load_list: [
        "modules.load.vendor_dlkm",
    ],
},
```

Install list 描述 **这个分区包含什么**，load list 描述 **需要加载什么**。

因此必须满足：

```text
load list ⊆ install list
```

`uwu_kernel` 会在构建时检查这一关系。如果 load list 中存在未包含在对应 install list 中的 module，构建将失败。`uwu_kernel` 不会因为一个 module 出现在 load list 中，就自动推断它应该被安装。

这项设计是有意的。最终 module layout 应当能够直接从设备配置中确认，而不需要重新推导规则。

uwuCLI 会解析 legacy kernel module 配置及其引用的静态 module list，并据此生成对应的 install 和 load list 配置。如果现有配置无法被可靠转换，uwuCLI 会告知您。自动转换完成后，您仍应检查各分区的 install 和 load list 是否符合设备实际情况。

### 自动收集 module dependencies

Install list 不需要手工列出其中 modules 的所有依赖。启用 `auto_collect_deps` 后，`uwu_kernel` 会根据 install list 中的 modules 自动收集其依赖，并将这些依赖加入最终的 install list：

```bp
modules: {
    enabled: true,
    auto_collect_deps: true,

    vendor_dlkm_module_install_list: [
        "modules.include.vendor_dlkm",
    ],
    vendor_dlkm_module_load_list: [
        "modules.load.vendor_dlkm",
    ],
},
```

`auto_collect_deps` 只补充 install list 所需的 module dependencies，不会根据 load list 推断哪些 modules 应当安装。您仍应明确指定设备需要的 install list 和 load list。

### External modules

如果设备使用独立于主 kernel tree 的 external modules，请指定 external module root 和需要构建的 modules：

```bp
modules: {
    enabled: true,

    external_module_root: "kernel/<vendor>/<device>-modules",
    external_modules: [
        "vendor/example",
    ],
},
```

默认情况下，`uwu_kernel` 使用 external module 自身的 build system 构建 module，并向其提供 kernel source 和 output directory 等信息。

如果 external module 是主 kernel Kbuild tree 的一部分，并需要通过 Kbuild 的 `M=` 方式构建，请在 module 路径后添加 `:kbuild`：

```bp
modules: {
    enabled: true,

    external_module_root: "kernel/<vendor>/<device>-modules",
    external_modules: [
        "vendor/example:kbuild",
    ],
},
```

这等价于通过主 kernel build system 对该目录执行 `M=<module> modules` 和对应的 `modules_install`。不要为了保留 legacy kernel build 的形式而为此复制额外的 Make rule。

具体属性和支持的 module 配置请参阅 [配置参考](configuration.md)。

## 接入 Android build

定义 `uwu_kernel` 后，需要让 Android build 使用该模块。

请在设备配置中选择 Soong kernel：

```make
BOARD_USES_SOONG_KERNEL := true
SOONG_KERNEL_MODULE := //device/<vendor>/<device>:kernel
```

并将 kernel 加入产品：

```make
PRODUCT_PACKAGES += kernel
```

其他模块应通过 `uwu_kernel` 的公开输出引用 kernel 产物。请勿依赖 `out/soong/.intermediates` 中的具体路径。

可用输出请参阅 [输出](outputs.md)。

## 无法自动迁移的配置

部分 legacy 配置表达的不只是 kernel build 参数，因此不能安全地机械转换。

常见情况包括：

- 自定义 DTB/DTBO Makefile；
- 依赖 `KERNEL_OUT`、`DTB_OUT` 或 `DTBO_OUT` 具体路径的脚本；
- 平台专用 kernel build wrapper。

遇到这些配置时，应先确认它们最终想得到什么结果，再用 `uwu_kernel` 表达这个结果。

不要为了逐行复现 legacy Make implementation，而重新在 Soong 中建立同样的隐式规则。

如果某种行为被多个设备共同需要，但当前 `uwu_kernel` 无法表达，请向 Issue Tracker 提出 Issue。

## 验证

迁移后首先进行正常构建：

```bash
uni
```

至少确认：

- kernel image 可以正常生成；
- 最终 `.config` 与设备预期一致；
- DTB/DTBO 可以生成并正常启动；
- kernel modules 安装到了正确的分区；
- modules load list 与实际启动需求一致；
- 设备可以正常启动并使用。

常见问题请参阅 [故障排查](troubleshooting.md)。