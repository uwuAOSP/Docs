# 从 Make 内核构建迁移到 uwu_kernel

## 迁移原则

旧版 `vendor/uwu/build/tasks/kernel.mk` 使用全局变量、隐式 Make 规则和多个共享的
中间目录。`uwu_kernel` 将 Android 侧的编排改为一个声明式模块，但底层 Kconfig、
Kbuild、DTS 和模块编译仍由 kernel 源码执行。

迁移不是把变量名称机械替换成小写属性。应先确定旧变量实际控制的是：

- kernel 源码和工具链；
- Kconfig 输入；
- DTB/DTBO 输出；
- kernel modules 的构建或安装；
- Android boot/分区镜像。

## 配置位置转换

| 旧 Make 配置 | `uwu_kernel` 配置 | 说明 |
| --- | --- | --- |
| `TARGET_KERNEL_SOURCE` | `kernel_dir` | 源码目录，相对 Android 根目录 |
| `KERNEL_ARCH` / `TARGET_KERNEL_ARCH` | `kernel_arch` | Kbuild 架构 |
| `BOARD_KERNEL_IMAGE_NAME` | `image_name` | `arch/<arch>/boot` 下的输出文件名 |
| `TARGET_PREBUILT_KERNEL` | `prebuilt` | 使用预编译 kernel，跳过源码构建 |
| 预编译 kernel config | `prebuilt_config` | 提供 `.config` 输出 |
| 预编译 kernel headers archive | `prebuilt_headers` | 提供 UAPI headers provider |
| 预编译 kernel modules zip | `prebuilt_modules` | 提供 `.modules` 输出和分区 installer |
| `TARGET_KERNEL_CONFIG` 第一个条目 | `config.defconfig` | 基础 defconfig |
| `TARGET_KERNEL_CONFIG` 其余条目 | `config.fragments` | 按顺序合并的 fragment |
| `TARGET_KERNEL_CONFIG_EXT` | `config.fragments` | 迁移为源码树中的显式配置路径 |
| `KERNEL_CONFIG_OVERRIDE` | `config.overrides` | 最终追加的配置行 |
| `MERGE_ALL_KERNEL_CONFIGS_AT_ONCE` | `config.merge_at_once` | 一次性或逐个合并 fragment |
| `KERNEL_LTO` | `config.lto` | `none`、`thin` 或 `full` |
| `TARGET_KERNEL_CLANG_VERSION` | `clang_version` | tree 内 Clang 版本 |
| `TARGET_KERNEL_CLANG_PATH` | `clang_path` | 自定义 Clang 路径 |
| `KERNEL_CLANG_TRIPLE` | `clang_triple` | 覆盖目标 triple |
| `KERNEL_CROSS_COMPILE` | `cross_compile` | 交叉工具链前缀 |
| `KERNEL_CC` | `cc`、`ld` 或 `make_flags` | 旧值若同时包含 `CC=` 和 `LD=`，应拆分后再填写 |
| `KERNEL_MAKE_CMD` | `make_command` | Kbuild 使用的 make |
| `KERNEL_MAKE_FLAGS` | `make_flags` | 传给每次 Kbuild 调用 |
| `TARGET_KERNEL_ADDITIONAL_FLAGS` | `additional_flags` | 设备附加 Kbuild flags |
| `CLANG_AUTOFDO_PROFILE` | `autofdo_profile` | AutoFDO profile；默认可使用 GKI profile |
| kernel rewrapper 配置 | `rbe_wrapper` | 仅包装实际编译动作 |
| `TARGET_KERNEL_MIXED_MODE` | 设备和分区配置 | 不再作为 kernel action 的全局开关 |

`TARGET_KERNEL_VERSION` 通常不属于 kernel 编译参数。QCOM 平台和 HAL 选择逻辑可能
仍然使用它，迁移时应保留并按平台需要设置。

## DTB/DTBO 转换

| 旧 Make 配置 | `uwu_kernel` 配置 |
| --- | --- |
| `BOARD_DTB_CFG` | `dtb.config` |
| `BOARD_DTBO_CFG` | `dtbo.config` |
| `BOARD_KERNEL_SEPARATED_DTBO` | `dtbo.enabled` 和设备 DT 配置 |
| `TARGET_MERGE_DTBS_WILDCARD` | `dtb.input_globs` |
| `TARGET_DTB_LIST_WILDCARD` | `dtb.input_globs` |
| `TARGET_MERGE_DTBOS_WILDCARD` | `dtbo.input_globs` |
| `TARGET_DTBO_LIST_WILDCARD` | `dtbo.input_globs` |
| `BOARD_CUSTOM_DTBIMG_MK` | `dtb.custom_command` 或通用 Soong 能力 |
| `BOARD_CUSTOM_DTBOIMG_MK` | `dtbo.custom_command` 或通用 Soong 能力 |
| QCOM merge dtbs 脚本 | `dtb.qcom_merge: true`，同时启用 DTB/DTBO |
| `BOARD_KERNEL_PAGESIZE` | Android boot/DTBO 分区相关配置；DTBO 使用 `page_size` |

旧 Make 变量有时只表达“是否进入某个规则”，而新属性还需要表达实际输入、输出
和 target。转换时应检查生成的命令和镜像内容，而不是只检查属性是否存在。

## Modules 转换

旧系统将模块构建、安装分区和加载清单分散在多个变量中。新系统集中到
`modules`：

| 旧 Make 配置 | `modules` 属性 |
| --- | --- |
| `BOARD_KERNEL_MODULES` / kernel `modules` target | `build_targets` |
| `TARGET_KERNEL_EXT_MODULE_ROOT` | `external_module_root` |
| 外部模块列表 | `external_modules` |
| `*_KERNEL_MODULES_LOAD` | 对应 `*_module_load_list` |
| `*_KERNEL_MODULES` 或 include 清单 | 对应 `*_module_install_list` |
| `BOARD_*_KERNEL_MODULES_BLOCKLIST` | 对应 `*_module_blocklist` |
| `TARGET_AUTO_COLLECT_KERNEL_MODULE_DEPS` | `auto_collect_deps` |
| `BOARD_KERNEL_MODULES_LOAD_ALLOW_MISSING` | `allow_missing_load` |
| 模块文件重命名规则 | `module_aliases` |
| `NEED_KERNEL_MODULE_ROOT` | 按实际分区重新声明，不直接迁移 |
| `NEED_KERNEL_MODULE_SYSTEM` | `system_dlkm` 或实际 system 安装属性 |
| `NEED_KERNEL_MODULE_VENDOR_OVERLAY` | 重新建模为对应 filesystem 依赖 |

`install_list` 和 `load_list` 必须同时存在，且 load list 中的模块必须出现在
install list。清单中可以使用路径，Soong 会按模块 basename 验证和生成结果。

## 设备转换示例

迁移时在设备的 `BoardConfig.mk` 和 `device.mk` 中接入 Soong kernel：

```make
# BoardConfig.mk
BOARD_USES_SOONG_KERNEL := true
SOONG_KERNEL_MODULE := //device/<vendor>/<device>:kernel

# device.mk
ifeq ($(BOARD_USES_SOONG_KERNEL),true)
PRODUCT_PACKAGES += kernel
endif
```

对应的 Android.bp 以声明式方式提供 kernel、DTB、DTBO、modules、config 和工具链：

```bp
uwu_kernel {
    name: "kernel",
    kernel_dir: "kernel/<vendor>/<kernel>",
    kernel_arch: "arm64",
    image_name: "Image",
    clang_version: "clang-r<version>",
    additional_flags: [
        "CONFIG_DEVICE_DTB=y",
    ],
    config: {
        defconfig: "gki_defconfig",
        fragments: [
            "vendor/common.config",
            "vendor/device.config",
        ],
    },
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
}
```

## 不能直接转换的配置

以下旧配置需要人工判断，不能简单替换：

- `TARGET_KERNEL_PLATFORM_TARGET`：这是外部 kernel platform/Kleaf 编排，不等价于
  `kernel_dir`；
- 旧 Make 的 RBE 全局变量不会自动迁移；应在模块中显式设置 `rbe_wrapper`；
- 自定义 DTB/DTBO Makefile：应优先补充通用 `uwu_kernel` 能力；
- `NEED_KERNEL_MODULE_*`：它们同时改变 Android 安装路径和分区依赖；
- 外部模块的独立 Makefile：需要决定使用普通 external module 还是 `:kbuild`；
- 通过 `$(shell)` 生成配置或清单：必须改成 Soong action 的显式输入输出；
- 依赖 `KERNEL_OUT`、`DTB_OUT` 或 `DTBO_OUT` 具体路径的脚本：应改为模块标签。

迁移完成后删除已被 `uwu_kernel` 接管的旧 kernel 编译变量，但保留仍被 Android
平台、boot image、HAL namespace 或分区配置需求的变量。

## 验证

迁移提交应至少验证：

1. kernel 默认输出和 `boot.img` 生成；
2. `.config` 包含基础配置、fragment、override 和 LTO 结果；
3. DTB/DTBO 的输入、合并方式、page size 和最终镜像正确；
4. kernel headers 能被 `generated_kernel_includes` 消费；
5. modules 的构建、安装集合、load list 和 blocklist 正确；
6. 源码、UAPI header、配置和清单修改能够触发对应增量 action；
7. Soong-only 与 Soong+Make 的 target-files 或镜像差异可解释。
