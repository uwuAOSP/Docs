# Soong-only 构建文档

本文档说明 uwuAOSP 如何在 Android 产品配置仍由 Make 解析的前提下，使用 Soong
生成构建目标、镜像和内核产物。文档同时说明 `uwu_kernel` 与旧版 Make 内核构建系统
之间的配置关系。

## 为什么使用 Soong-only

Soong 的目标是让 Android 构建不再依赖 Makefile 生成构建目标。产品配置仍可以使用
Make，但构建目标和 Ninja 规则直接由 Soong 从模块图生成，从而跳过 Kati 的主目标
生成阶段。

跳过 Kati 的主要收益是减少构建分析开销。Soong 官方说明中给出的目标是将分析时间
降低到约一半，从而提高开发效率。这里的收益主要体现在构建启动和依赖图分析阶段；
它不会跳过 Ninja 执行的编译动作，也不会跳过 `uwu_kernel` 内部调用的 Linux Kbuild。

Soong-only 不能简单地理解为“删除所有 Make”。Make 仍负责产品配置和变量展开，
但不再负责把 Android.mk 目标转换成主构建 Ninja 图。

## 文档导航

| 页面 | 内容 |
| --- | --- |
| [构建流程](build-flow.md) | `PRODUCT_SOONG_ONLY`、配置导出和 Soong 构建图 |
| [镜像生成](image-generation.md) | fsgen、boot/DTBO/vbmeta/super 镜像和输出位置 |
| [构建验证](validation.md) | 验证命令、产物检查和增量构建检查 |
| [uwu_kernel 构建系统](uwu_kernel/README.md) | `uwu_kernel` 的页面索引和职责边界 |

## 基本概念

Soong-only 不是完全移除 Make。产品配置仍需要 Make 读取产品继承关系和生成 Soong
配置变量；区别在于产品配置完成后，主构建目标和 Ninja 规则由 Soong 生成，Kati 的
主目标生成阶段不再执行。

一个产品使用 Soong-only 的必要配置是：

```make
PRODUCT_SOONG_ONLY := true
```

也可以使用命令行参数临时选择模式：

```sh
m --soong-only <target>
m --no-soong-only <target>
```

`SOONG_ONLY=true` 是等价的环境变量入口。命令行参数优先于产品默认值。

设备的关键配置通常位于：

- `device/<vendor>/<device>/BoardConfig*.mk`：启用 Soong kernel 并指定模块；
- `device/<vendor>/<device>/Android.bp`：声明 `uwu_kernel`；
- `device/<vendor>/<device>/device.mk`：将 kernel 安装到产品；
- `vendor/uwu/config/BoardConfigSoong.mk`：将 kernel 选择导出给 Soong。

某个镜像是否生成由产品的分区配置决定，不能仅根据 Soong-only 模式推断所有镜像
都存在。

## 参考实现

- Android 原生 Soong-only 说明：`build/soong/docs/soong_only.md`
- Soong 最佳实践：`build/soong/docs/best_practices.md`
