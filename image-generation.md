# Soong-only 镜像生成

## 组件职责

| 组件 | 主要职责 |
| --- | --- |
| `build/make/core/soong_config.mk` | 导出分区、boot 和 AVB 参数 |
| `build/soong/fsgen` | 根据产品变量创建镜像模块和依赖 |
| `build/soong/filesystem` | 实现 filesystem、bootimg、DTBO、vbmeta 等模块 |
| `android_device` | 汇总设备依赖并复制结果到 `PRODUCT_OUT` |
| `ninja` | 按 Soong 生成的规则执行实际命令 |

## Filesystem 分区

fsgen 从 `PartitionQualifiedVariables` 判断分区是否需要生成。常见分区包括
`system`、`system_ext`、`product`、`vendor`、`odm`、`userdata`、`system_dlkm`、
`vendor_dlkm`、`odm_dlkm` 和 `recovery`。

每个 filesystem 模块先生成 staging directory，再根据分区的文件系统和 AVB 配置
生成镜像。动态分区产品还会把有效分区交给 `super` 镜像模块；vbmeta 模块则消费
分区模块提供的输出和公钥信息。

## Boot 镜像

fsgen 根据以下输入创建 boot 类镜像：

- `SOONG_KERNEL_MODULE` 指向的 kernel 模块默认输出；
- kernel 模块的 `.dtb` 输出；
- `ramdisk`、`vendor_ramdisk` 或 recovery filesystem；
- boot header version、partition size、kernel cmdline 和 bootconfig；
- 对应分区的 AVB key、algorithm、rollback index 和 security patch。

`android_device` 再将模块输出复制为：

```text
$(PRODUCT_OUT)/boot.img
$(PRODUCT_OUT)/init_boot.img
$(PRODUCT_OUT)/vendor_boot.img
$(PRODUCT_OUT)/vendor-bootconfig.img
```

`vendor_kernel_boot.img` 只有在 `vendor_kernel_boot` 分区启用并且对应模块被创建时
才会出现。

## DTB 和 DTBO

`uwu_kernel` 可以提供 `.dtb` 和 `.dtbo` 输出。fsgen 在 Soong-only 模式下可以从
kernel 模块标签读取 DTBO：

```text
:kernel{.dtbo}
```

设备也可以通过 `BOARD_PREBUILT_DTBOIMAGE` 提供预编译 DTBO。DTBO 的 page size、AVB
参数和输出名称来自分区配置。DTBO 模块将输出提供给 vbmeta，同时安装到 Soong 的
`etc` 输出目录，`android_device` 将最终镜像复制到 `PRODUCT_OUT/dtbo.img`。

QCOM 设备的常见路径是：

1. Kbuild 生成多个 DTB/DTBO；
2. `merge_dtbs.py` 合并和整理设备树；
3. `mkdtboimg create --page_size=<size>` 生成 DTBO；
4. fsgen 将 DTB/DTBO 接入 boot、DTBO 和 vbmeta 依赖。

## AVB 和 super

每个启用 AVB 的分区从 `PartitionQualifiedVariables` 获取自己的 key、algorithm、
rollback index 和 footer 参数。fsgen 负责创建分区 AVB 信息，vbmeta 模块按
`vbmeta_*` 的分区列表建立链式依赖。

动态分区产品的依赖关系通常是：

```text
system/product/vendor/odm/... filesystem
                |
                v
              super
                |
                v
          vbmeta_system/vendor
                |
                v
              vbmeta
```

不要在设备配置中手工复制这个依赖关系。应该设置正确的 BoardConfig 变量，让 fsgen
从产品配置自动创建模块。

## 产品输出

产品输出目录通常为 `out/target/product/<device>`。常见输出包括：

| 输出 | 说明 |
| --- | --- |
| `kernel` | `uwu_kernel` 的默认内核镜像输出 |
| `dtb.img` | 设备树镜像，是否合并取决于设备配置 |
| `dtbo.img` | DTBO 镜像，page size 取决于产品配置 |
| `boot.img` | boot 镜像 |
| `init_boot.img` | init_boot 镜像 |
| `vendor_boot.img` | vendor boot 镜像 |
| `system.img`、`vendor.img`、`product.img` | 动态分区成员镜像 |
| `super.img` | 动态分区容器镜像 |
| `vbmeta*.img` | AVB 链镜像 |

`boot_16k.img`、`vendor_kernel_boot.img` 等特殊镜像只有在对应产品分区和 kernel
配置启用时才会生成。缺少未启用的镜像不代表 Soong-only 构建失败。
