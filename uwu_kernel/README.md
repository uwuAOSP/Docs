# uwu_kernel

`uwu_kernel` 是 uwuAOSP 用于将 kernel build 接入 Soong 的模块类型，旨在替代 legacy kernel build task。

`uwu_kernel` 不替代 kernel 自身的 build system。Kbuild、Bazel 等仍负责实际的 kernel build，`uwu_kernel` 负责将其构建过程和产物接入 Android build system。

## 为什么需要 uwu_kernel

Legacy kernel build task 是普通设备切换 Soong only 最大的阻碍，切换到 Soong Only 可以极大减小 build graph 生成时间。

Legacy kernel module 配置将 module 的安装位置和加载列表分散在多个 Make 变量和文件中，两者之间还存在交叉引用，因此以下问题很难直接从配置中确认：

- 一个 module 最终安装到了哪个分区；
- 一个 module 为什么被安装；
- 哪些 module 会在启动过程中实际加载。

`uwu_kernel` 不保留这些隐式行为。设备应直接声明需要的 kernel 配置、输出和 module 布局，使最终状态能够从设备配置中直接确认。

## 迁移

我们推荐使用 uwuCLI 完成初始迁移。它会读取现有设备配置，并生成对应的 `uwu_kernel` 配置。

对于 kernel modules，uwuCLI 会解析 legacy module 配置及其引用的静态 module list，并据此生成对应的 install 和 load list 配置。如果现有配置无法被可靠转换，uwuCLI 会报告 blocker，而不是猜测设备需要的 module 布局。

建议您对自动转换的结果进行检查。

详细步骤请参阅 [从 legacy kernel build 迁移](migration.md)。

## 文档

- [迁移指南](migration.md)：将现有设备迁移到 `uwu_kernel`
- [配置参考](configuration.md)：`uwu_kernel` 支持的配置项
- [输出](outputs.md)：可供其他 Soong 模块使用的 kernel 输出
- [故障排查](troubleshooting.md)：常见构建和配置问题

## 已验证设备

`uwu_kernel` 已在以下具有代表性的 kernel 配置上完成构建和启动验证：

| 设备 | Kernel | 类型 | Kernel build system |
| --- | --- | --- | --- |
| OnePlus 6T (`fajita`) | 4.19 | non-GKI | Kbuild |
| OnePlus Ace 3 / 12R (`aston(c)`) | 5.15 | GKI | Kbuild |
| POCO F7 / Redmi Turbo 4 Pro (`onyx`) | 6.6 | GKI | Bazel |