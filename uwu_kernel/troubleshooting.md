# uwu_kernel 故障排查

## 修改 kernel 源码后没有重新编译

首先确认修改的文件位于 `kernel_dir` 中。如果修改的是 external module，请确认它位于 `external_module_root` 中。

`uwu_kernel` 会跟踪这些目录中的源码变化。只有位于这些目录之外、但仍需要作为 kernel build 输入的文件，才应通过 `srcs` 显式声明。

不要使用：

```bp
srcs: ["**/*"],
```

这种方式将整个源码树展开为 Soong 输入会显著增加 Soong 分析和生成 build graph 的开销。遇到源码依赖没有被正确跟踪的问题时，应首先检查 `kernel_dir`、`external_module_root` 和实际源码位置。

## External module 构建失败

首先确认 `external_module_root` 和 `external_modules` 指向正确的位置。

普通 external module 会通过其自身的 Makefile 构建。如果该 module 需要使用主 kernel Kbuild 的 `M=` 模式，应使用 `:kbuild`：

```bp
external_modules: [
    "vendor/example:kbuild",
],
```

如果 module 可以被编译但无法正确安装，还应检查其生成的 `.ko` 是否位于 `uwu_kernel` 能够收集的 module 输出中。

## Kernel module 没有安装到预期分区

检查该 module 是否包含在对应分区的 install list 中。`uwu_kernel` 支持 `system_dlkm`、`vendor_dlkm`、`vendor_ramdisk` 和 `recovery`。

Install list 决定 module 是否安装到该分区，load list 只决定其中哪些 modules 需要加载。不要为了安装一个 module 而将它加入 load list。

如果启用了 `auto_collect_deps`，依赖 module 会根据 install list 自动补充；如果未启用，则应确保所需依赖已经包含在安装集合中。

## Kernel module 没有加载

首先确认 module 已安装到预期分区，然后检查它是否存在于该分区的 load list 中。

`uwu_kernel` 要求：

```text
load list ⊆ install list
```

如果 load list 中包含未安装到对应分区的 module，构建会直接失败。

如果 module 已正确安装并存在于 load list，但设备启动后仍未加载，应继续检查生成的 `modules.load`、module dependencies、blocklist 和设备启动日志。此时问题通常已经不属于 `uwu_kernel` 的 module layout 配置本身。

## DTB 或 DTBO 构建失败

确认对应输出已经启用，并检查 `target`、`input_globs` 和实际 Kbuild 输出是否一致。

使用 `qcom_merge` 时，`dtb.enabled` 和 `dtbo.enabled` 必须同时启用。该模式使用 `dtb.target` 构建设备树，并从生成的 DTS 输出中完成后续 DTB/DTBO 合并。

如果设备使用了不同于标准流程的设备树布局，应先确认能否通过 `target` 或 `input_globs` 描述；只有标准流程无法覆盖时才应使用 `custom_command`。

## Kernel configuration 与预期不一致

检查最终生成的 `.config`，不要只检查 defconfig、fragment 或 `overrides` 的源文件。

Kconfig 在合并 fragment、应用 LTO 配置和追加 override 后还会执行默认值处理，因此最终 `.config` 才是实际参与 kernel build 的配置。