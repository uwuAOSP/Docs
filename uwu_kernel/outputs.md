# uwu_kernel 输出与依赖

## 输出标签

`uwu_kernel` 通过 Soong 输出标签暴露产物：

| 标签 | 输出 | 主要消费者 |
| --- | --- | --- |
| 默认标签 `""` | kernel image | fsgen bootimg、android_device |
| `.config` | 最终 Kconfig 文件 | 调试和配置验证 |
| `.dtb` | DTB image | boot image、设备验证 |
| `.dtbo` | DTBO image | fsgen DTBO、vbmeta |
| `.modules` | kernel modules zip | 分区模块安装器 |

源码 kernel 会按启用的属性生成 `.config`、headers、DTB、DTBO 和 modules。prebuilt
kernel 不执行 Kbuild；只有配置了 `prebuilt_config`、`prebuilt_headers`、`dtb.src`、
`dtbo.src` 或 `prebuilt_modules` 时，才会提供对应输出。

设备配置应通过模块引用标签，例如：

```text
:kernel
:kernel{.dtb}
:kernel{.dtbo}
:kernel{.modules}
```

不要把 `out/soong/.intermediates` 中的具体路径写入 Android.bp 或 Makefile。

## Kernel action

源码构建会创建独立的 `kernel_build` 输出目录，并建立以下依赖关系：

1. 创建 kernel 源码和外部模块的目录依赖 stamp；
2. 由配置输入生成 `.config`，并由源码输入运行 `headers_install` 和 UAPI headers 清洗；
3. 让 kernel image action 依赖 `.config` 和源码依赖 stamp；
4. 让 DTB/DTBO action 依赖 kernel image、DTS 输出和源码依赖 stamp；
5. 让 modules action 依赖 kernel/DT 输出，编译、安装并打包 kernel modules。

每个 action 都把前一阶段输出作为显式依赖，因此 DT、modules 和 headers 不会脱离
kernel image 的构建图单独漂移。

如果启用了 `autofdo_profile`，profile 也会作为 kernel、DT 和 modules action 的显式
输入。启用 `rbe_wrapper` 时，wrapper 脚本同样作为 action 输入。

## 工具链

默认 Kbuild 调用使用 tree 内工具：

- Clang：`prebuilts/clang/host/linux-x86/<version>/bin`；
- build tools：`prebuilts/build-tools/linux-x86/bin`；
- kernel tools：`prebuilts/kernel-build-tools/linux-x86/bin`；
- Lineage 工具：`prebuilts/tools-lineage/linux-x86/bin`；
- Perl 基础模块：`prebuilts/tools-lineage/common/perl-base`。

调用中会显式设置 `LLVM=1`、`LLVM_IAS=1`、`DTC_EXT`、`LZ4`、`LEX`、`YACC`、`M4`、
`PAHOLE`、`LIBCLANG_PATH`、`CC` 和 `LD`。这避免 kernel 构建依赖宿主发行版中恰好
安装的工具版本。

## 源码依赖

`source_deps/source.d` 由 Soong 目录依赖规则生成。kernel 源码和配置文件作为 action
输入，源码目录作为目录依赖；因此新增或修改 kernel 文件时可以触发对应 action，
同时避免把整个源码树展开到每条 Ninja 规则。

`srcs` 只用于目录依赖之外的额外文件。不要使用：

```bp
srcs: ["**/*"],
```

这种写法会增加 Soong 分析内存、Ninja 文件体积和重新分析成本。

## UAPI headers

`uwu_kernel` 执行 Kbuild `headers_install`，然后运行
`vendor/uwu/build/tools/clean_headers.sh`。公共的 `generated_kernel_includes` 模块
会转发 `SOONG_KERNEL_MODULE` 指向的 headers；未迁移设备才回退到
`generated_kernel_includes_legacy`。

预编译 kernel 可以通过 `prebuilt_headers` 使用相同的 headers provider；压缩包必须是
gzip 压缩的 tar archive。

原生模块继续依赖 `generated_kernel_includes`，不需要改成具体设备的中间目录。

## Modules zip 和分区安装器

`.modules` 输出是一个包含安装结果、load list 和 blocklist 的 zip。`uwu_kernel` 会
根据四类分区的清单生成内部的 `PrebuiltKernelModules` 模块，fsgen 再把这些模块接入
对应 filesystem 或 ramdisk。

这些内部模块不是给设备产品配置直接添加的公共产品模块。设备只需要在
`uwu_kernel.modules` 声明清单，并通过 `PRODUCT_PACKAGES += kernel` 安装主模块。

## 中间输出

设备构建的主要 kernel 中间目录通常是：

```text
out/soong/.intermediates/device/<vendor>/<device>/kernel/<variant>/
```

其中可能包含 `kernel/<image_name>`、`kernel_build/.config`、`dtb/<image_name>`、
`dtbo/<image_name>`、`headers/` 和 `source_deps/source.d`。设备侧应把这些作为验证
线索，而不是稳定 API。
