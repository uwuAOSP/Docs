# uwu_kernel 输出

`uwu_kernel` 通过 Soong output label 向其他模块提供 kernel 构建产物。

## 输出标签

| 标签 | 输出 |
| --- | --- |
| 默认标签 | kernel image |
| `.config` | 最终 kernel configuration |
| `.dtb` | DTB image |
| `.dtbo` | DTBO image |
| `.modules` | kernel modules archive |

例如，名为 `kernel` 的 `uwu_kernel` 可以通过以下方式引用：

```text
:kernel
:kernel{.config}
:kernel{.dtb}
:kernel{.dtbo}
:kernel{.modules}
```

除默认的 kernel image 外，其他标签只有在对应输出存在时才可用。例如，只有启用 DTBO 输出后才会提供 `.dtbo`。

设备配置和其他 Soong 模块应通过这些标签引用 kernel 输出，请勿依赖 `out/soong/.intermediates` 中的具体路径。

## Kernel headers

Kernel UAPI headers 不通过 output label 暴露。`uwu_kernel` 会执行 Kbuild `headers_install` 并清理生成的 headers，然后通过 Generated headers 接口提供给其他模块。

`generated_kernel_includes` 会使用 `SOONG_KERNEL_MODULE` 指向的 `uwu_kernel` 提供 headers。这保留了对于目前 Android tree 中依赖 `generated_kernel_includes` 的库的兼容性。

## Kernel modules

启用 kernel modules 后，`.modules` 提供构建生成的 kernel modules archive。`uwu_kernel` 会根据 `modules` 中声明的 install list、load list 和 blocklist 创建对应的分区 module installer。

设备不需要直接添加这些内部 installer。将主 `uwu_kernel` module 加入产品即可：

```device.mk
PRODUCT_PACKAGES += kernel
```

Kernel module 的分区和加载配置请参阅 [配置参考](configuration.md)。