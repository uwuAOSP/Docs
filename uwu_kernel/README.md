# uwu_kernel 构建系统

`uwu_kernel` 是 uwuAOSP 提供的 Soong 模块类型。它不替代 Linux Kbuild，而是把
Android 侧的 kernel 编排从全局 Make 变量和隐式规则转换为 Soong 模块依赖。

## 为什么需要 uwu_kernel

Soong-only 的核心是跳过 Kati 的 Android 目标生成阶段。如果 kernel 仍然只通过旧的
`vendor/uwu/build/tasks/kernel.mk` 构建，kernel image、DTB、DTBO、modules 和 UAPI
headers 的输入输出关系仍停留在 Make 的隐式规则中，Soong 和 fsgen 就无法可靠地把
这些产物纳入自己的依赖图。

`uwu_kernel` 将 Android 侧的 kernel 构建声明为 Soong 模块，使 Soong 能够看到：

- kernel 源码、配置和工具链输入；
- kernel image、DTB、DTBO、modules 和 UAPI headers 输出；
- 外部模块、安装清单和分区之间的依赖。

因此 fsgen 可以直接消费模块输出并生成 boot、DTBO、super 和 vbmeta 镜像，而不需要
重新依赖 Kati 生成的 kernel 目标。这样才能在跳过 Kati 后保持完整的 Android 构建
依赖关系。

源码构建仍由 Soong action 调用 Linux Kbuild；`prebuilt` 模式不会调用 Kbuild。预编译
模式如需继续提供配置、headers 或 modules 输出，必须同时声明对应的
`prebuilt_config`、`prebuilt_headers` 和 `prebuilt_modules` 属性。

Soong 的官方说明指出，Soong-only 的目标是将构建分析时间降低到约一半并提高开发
效率。`uwu_kernel` 参与的是实现这一目标所需的 kernel 迁移：它跳过 Android 侧的
legacy Make 编排，但仍由 Soong action 调用 Linux Kbuild 完成实际 kernel 编译。

## 页面导航

| 页面 | 内容 |
| --- | --- |
| [配置参考](configuration.md) | 所有常用 `uwu_kernel` 属性和设备示例 |
| [输出与依赖](outputs.md) | kernel、DT、modules、headers 输出及消费者 |
| [Make 迁移](migration.md) | 旧 Make 配置到 `uwu_kernel` 的转换方法 |
| [故障排查](troubleshooting.md) | 配置、依赖、工具链和增量构建问题 |

## 职责边界

| 层 | 职责 |
| --- | --- |
| `uwu_kernel` | 声明输入输出、准备工具链、调用 Kbuild、导出 headers 和镜像 |
| Linux Kbuild | 执行 Kconfig、编译内核、生成 DT 和模块 |
| `fsgen` | 消费 kernel 输出并创建 Android boot、DTBO、super、vbmeta 镜像 |
| 设备树 | 选择源码、defconfig、fragment、DT 策略和模块安装清单 |
| 内核树 | 维护源码、Kbuild、DTS、defconfig 和 UAPI |

迁移不要求也不应该把内核源码中的 Makefile 改写成 Android.bp。Soong 只负责
Android 构建图中的调用和依赖声明。

## 最小配置

```bp
uwu_kernel {
    name: "kernel",
    kernel_dir: "kernel/vendor/device",
    kernel_arch: "arm64",
    image_name: "Image",
    clang_version: "clang-r<version>",
    config: {
        defconfig: "gki_defconfig",
        fragments: ["vendor/device.config"],
    },
}
```

未设置 `clang_version` 时会继承 envsetup 导出的 AOSP clang 版本。需要 AutoFDO 或 RBE
时分别设置 `autofdo_profile` 和 `rbe_wrapper`，二者都只影响源码 kernel。

BoardConfig 阶段选择该模块：

```make
BOARD_USES_SOONG_KERNEL := true
SOONG_KERNEL_MODULE := //device/<vendor>/<device>:kernel
```

产品配置将它加入安装集合：

```make
PRODUCT_PACKAGES += kernel
```

## 设计原则

- 设备通过模块名和输出标签消费 kernel 产物，不直接依赖 `.intermediates` 路径；
- 通用构建逻辑进入 `vendor/uwu/build/soong/kernel`，设备差异留在 Android.bp；
- kernel 源码由目录依赖跟踪，不把整个源码树展开成 `srcs`；
- kernel、DT、模块和 UAPI headers 都必须在 Soong 图中有明确依赖；
- legacy Make 路径只作为未迁移设备的兼容路径，不再添加新的设备特例。
