# uwu_kernel 配置参考

## 顶层属性

| 属性 | 说明 |
| --- | --- |
| `kernel_dir` | 相对 Android 源码根目录的 kernel 源码目录，源码构建必填 |
| `prebuilt` | 预编译 kernel 路径；设置后跳过源码 Kbuild |
| `prebuilt_config` | 预生成 `.config` 文件；仅用于 `prebuilt` |
| `prebuilt_headers` | gzip 压缩的 kernel UAPI headers archive；仅用于 `prebuilt` |
| `prebuilt_modules` | 预生成 kernel modules zip；仅用于 `prebuilt` |
| `kernel_arch` | Kbuild 架构，例如 `arm64` |
| `image_name` | `arch/<arch>/boot` 下的内核文件名，源码构建必填 |
| `clang_version` | `prebuilts/clang/host/linux-x86` 下的工具链版本 |
| `clang_path` | 自定义 Clang 目录，优先于 `clang_version` |
| `rust_version` | 可选的 Rust 工具链版本 |
| `clang_triple` | 覆盖默认的 `CLANG_TRIPLE` |
| `cross_compile` | 传给 Kbuild 的 `CROSS_COMPILE` |
| `cc` / `ld` | 覆盖默认 C 编译器或链接器 |
| `autofdo_profile` | AutoFDO profile 路径；默认查找 GKI profile，设置为 `none` 禁用 |
| `rbe_wrapper` | 完整的 `rewrapper` 命令；设置后使用 `kernel_rbe_cc.sh` 进行实际编译 |
| `make_command` | 覆盖默认的 tree 内 `make` |
| `build_jobs` | 覆盖 Kbuild 的 `-j` 并行度 |
| `make_flags` | 传给所有 Kbuild 调用的变量或参数 |
| `additional_flags` | 设备额外的 Kbuild 配置参数 |
| `environment` | 传给 Kbuild action 的额外环境赋值 |
| `srcs` | 额外的分析期输入，不用于代替源码目录依赖 |

默认 Kbuild 并发度为整数计算的 `(逻辑 CPU 数 + 2) * 3 / 2`。只有主机内存不足或
kernel 存在并发问题时才需要降低 `build_jobs`。

未显式设置 `clang_version` 和 `rust_version` 时，优先使用 envsetup 导出的
`LLVM_AOSP_PREBUILTS_VERSION` 和 `RUST_AOSP_PREBUILTS_VERSION`，再分别回退到
`clang-stable` 和不添加 Rust 工具链路径。

`environment` 中的每一项必须是 `NAME=value`，会作为 Kbuild action 的 shell 环境变量；
需要传递 Make 变量时应使用 `make_flags` 或 `additional_flags`。

## Kconfig

```bp
config: {
    defconfig: "gki_defconfig",
    fragments: [
        "vendor/common.config",
        "vendor/device.config",
    ],
    merge_at_once: true,
    overrides: [
        "CONFIG_EXAMPLE=y",
    ],
    lto: "thin",
},
```

配置处理顺序是：

1. 以 `defconfig` 初始化 `.config`；
2. 执行 `olddefconfig`；
3. 按顺序或一次性合并 `fragments`；
4. 根据 `lto` 修改 LTO 选项；
5. 追加 `overrides`；
6. 最后再次执行 Kconfig 的默认值处理。

`lto` 支持 `none`、`thin` 和 `full`。fragment 和 override 的最终结果应通过生成的
`.config` 验证，而不是只检查源文件。

不包含 `/` 的配置名以及 `vendor/` 开头的配置名按以下路径解析：

```text
<kernel_dir>/arch/<config_arch>/configs/<name>
```

其他包含 `/` 的配置名按 Android 源码根目录解析；例如 `kernel/vendor/foo.config`
可以直接指向源码根目录下的文件。`x86_64` 的 defconfig 目录仍转换为
`arch/x86/configs`。

## DTB 和 DTBO

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

| 属性 | 说明 |
| --- | --- |
| `enabled` | 启用该类设备树输出 |
| `qcom_merge` | 使用 uwuAOSP 的 QCOM DT 合并流程 |
| `src` | 预编译 DTB/DTBO 输入 |
| `target` | 传给 Kbuild 的目标，默认是 `dtbs` 或 `dtbo.img` |
| `image_name` | 输出名称，默认是 `dtb.img` 或 `dtbo.img` |
| `config` | `mkdtboimg cfg_create` 使用的配置文件 |
| `input_globs` | Kbuild 未直接生成最终镜像时收集 DT 文件的 glob |
| `page_size` | DTBO `mkdtboimg create` 的 page size，默认 4096 |
| `custom_command` | 标准流程无法覆盖时使用的命令 |

`dtb.src` 和 `dtbo.src` 只适用于 `prebuilt` kernel。源码构建中应通过 Kbuild target
或 `input_globs` 收集设备树输出。`dtb.config` 和 `dtb.page_size` 当前不支持；QCOM
merge 模式使用 `dtb.target`，`dtbo.target` 不生效但会被忽略。

`custom_command` 可使用 `$(kernelDir)`、`$(kernelOut)` 和 `$(out)`。它只能作为最后
手段；重复出现的流程应实现为通用能力。

启用 `qcom_merge` 时，`dtb.enabled` 和 `dtbo.enabled` 必须同时为 true。QCOM 合并
流程从 kernel 的 DTS 输出建立独立的 merge 工作目录，再分别生成 DTB 和 DTBO；Kbuild
本身仍然使用共享的 `kernel_build` 输出目录。

## Modules

```bp
modules: {
    enabled: true,
    build_targets: ["modules"],
    external_module_root: "kernel/vendor/device-modules",
    external_modules: [
        "vendor/example",
        "vendor/kbuild-module:kbuild",
    ],
    install_strip: true,
    auto_collect_deps: true,
    system_dlkm_module_install_list: ["modules.include.system_dlkm"],
    system_dlkm_module_load_list: ["modules.load.system_dlkm"],
    vendor_dlkm_module_install_list: ["modules.include.vendor_dlkm"],
    vendor_dlkm_module_load_list: ["modules.load.vendor_dlkm"],
    recovery_module_install_list: ["modules.include.recovery"],
    recovery_module_load_list: ["modules.load.recovery"],
},
```

模块安装集合支持 `system_dlkm`、`vendor_dlkm`、`vendor_ramdisk` 和 `recovery`。
每类都必须同时提供 install list 和 load list；load list 必须是 install list 的
子集。每类还可以配置对应的 blocklist。

普通 `external_modules` 条目按照外部模块自己的 Makefile 构建；带 `:kbuild` 后缀的
条目使用 kernel Kbuild 的 `M=` 模式。`module_aliases` 使用
`old_name.ko:new_name.ko` 格式。`auto_collect_deps` 会根据生成的模块依赖补充安装
集合。

启用 `auto_collect_deps` 时，构建还需要源码树中的
`lineage/scripts/collect-kernel-module-deps/collect-kernel-module-deps.py`。每类分区
只有在配置 install list 或 load list 时才创建内部 installer；一旦配置其中一个，另一个
也必须配置。`vendor_dlkm` 在同时启用 `system_dlkm` 时会依赖 system_dlkm installer。

RBE wrapper 只包装实际的 C/汇编编译，预处理、汇编、依赖分析等其他 clang 调用仍在本地
执行。`rbe_wrapper` 应包含所需的 rewrapper 参数，例如：

```bp
rbe_wrapper: "prebuilts/remoteexecution-client/live/rewrapper --labels=type=compile,lang=cpp,compiler=clang",
```

## 设备配置

完整配置应位于设备树的 `Android.bp`。本文只列出配置模型，不复制设备专属的
模块清单、DT 输入和内核路径。不同设备可以选择不同的架构、工具链、DT 流程和
模块分区，但都应通过同一个 `uwu_kernel` 模块接口表达。
