# Soong-only 构建验证

## 构建前

在源码根目录初始化环境并选择产品：

```sh
source build/envsetup.sh
lunch uwu_device-cp2a-user
```

产品名称以设备的 `AndroidProducts.mk` 为准。验证不同设备时，应使用该设备自己的
分区和 kernel 配置。

## 目标验证

先构建 Soong-only 的主要目标：

```sh
m --soong-only bootimage
m --soong-only dtboimage
m --soong-only superimage
m --soong-only vbmetaimage
```

完整产品构建可以使用：

```sh
m --soong-only
```

如果产品配置已经设置 `PRODUCT_SOONG_ONLY := true`，也可以省略
`--soong-only`。

## 产物检查

检查主输出是否存在且非空：

```sh
for image in kernel dtb.img dtbo.img boot.img init_boot.img vendor_boot.img \
        system.img vendor.img product.img super.img vbmeta.img \
        vbmeta_system.img vbmeta_vendor.img; do
    test -s "out/target/product/<device>/$image" || exit 1
done
```

检查产品配置记录的实际分区和 AVB 参数：

```sh
grep -E '^(avb_|building_|dynamic_partition_list=|has_dtbo=|init_boot=|vendor_boot=)' \
    out/target/product/<device>/misc_info.txt
```

镜像格式检查应使用与产品配置匹配的 host 工具。例如，已配置 AVB 时可以检查：

```sh
avbtool info_image --image out/target/product/<device>/boot.img
avbtool info_image --image out/target/product/<device>/dtbo.img
```

## kernel 输出检查

`uwu_kernel` 的 Soong 中间输出使用模块标签，而不是设备侧硬编码路径。调试时可以
查看当前构建的模块目录：

```text
out/soong/.intermediates/device/<vendor>/<device>/kernel/<variant>/
```

常见输出标签对应：

| 标签 | 实际内容 |
| --- | --- |
| 默认输出 | 内核镜像，例如 `kernel/Image` |
| `.config` | `kernel_build/.config` |
| `.dtb` | `dtb/dtb.img` |
| `.dtbo` | `dtbo/dtbo.img` |
| `.modules` | 内核模块 zip，启用并被目标消费时生成 |
| generated headers | `headers/usr/include` 等 UAPI 目录 |

检查最终配置时，应查看生成的 `.config`：

```sh
grep -E 'CONFIG_(<device-specific-options>|LTO_)' \
    out/soong/.intermediates/device/<vendor>/<device>/kernel/<variant>/kernel_build/.config
```

## Soong-only 差异测试

Google 的 Soong 文档建议比较 Soong-only 和 Soong+Make 的结果。产品配置完成后运行：

```sh
build/soong/scripts/soong_only_diff_test.py <product>
```

该脚本会分别执行两种模式，构建 target-files 和镜像目标，并比较产物差异。差异应
先按分区、安装路径、文件列表和镜像 metadata 分类，再决定是设备配置问题还是 Soong
迁移缺口。

## 增量依赖检查

修改内核源码后，至少验证以下两点：

1. `kernel_image` action 被重新执行；
2. 未修改的 Android 分区不被无条件重建。

`uwu_kernel` 使用 `source_deps/source.d` 跟踪 kernel 源码目录，不要通过
`srcs: ["**/*"]` 替代目录依赖。

修改 UAPI header、defconfig、fragment、DTB/DTBO 配置或模块清单后，应确认相应的
headers、config、device-tree 或 modules action 重新执行。
