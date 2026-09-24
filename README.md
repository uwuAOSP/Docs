# Soong-only

Android build graph 的生成过程中包含一个串行执行的 Kati 阶段。Kati 负责处理 Make
构建规则，并生成相应的 Ninja 构建规则。

LineageOS 在 23.2 的 release blog 中提到：

> LineageOS is now nearly Android.mk free! Google announced their move from make to
> soong many years ago, pushing developers to migrate from Android.mk to Android.bp,
> and has started blocking Android.mk in many locations of the source tree.

但是，一些核心构建组件仍然依赖 Make，因此 Kati 仍然是构建流程中不可跳过的一环。

uwuAOSP 继续完成这项迁移的最后一步，使现代设备可以跳过 Kati 的主构建图生成阶段。
在我们的测试中，这使 build graph 的生成时间近乎减半。产品配置（例如 BoardConfig）仍然使用 Make；
Soong-only 并不意味着设备树中不能再使用 Makefile。

## 迁移设备

对于现代设备，uwuAOSP 已经处理了大部分 Soong-only 所需的构建组件。设备 bringup
时通常只需要迁移以下部分：

- [`uwu_kernel`](uwu_kernel/)：替代 legacy kernel build task；
- `uwu_prebuilt_image`：替代 Make 层的 `$(call add-radio-file, ...)`。

为简化设备 bringup，uwuCLI 已经提供相应的迁移脚本。脚本机械转换后，您仍应
检查并验证自动生成的构建规则。

要临时启用 Soong-only，请配置环境变量：

```bash
export SOONG_ONLY=true
```

也可以在产品配置中设置：

```make
PRODUCT_SOONG_ONLY := true
```

> [!NOTE]
> A-only 设备依赖的 `//bootable/deprecated-ota:updater` 尚未完成迁移，因此目前不能
> 使用 Soong-only。如果您需要支持此类设备，请在 issue tracker 中提交 issue。

## 验证

迁移完成后，请至少完成一次完整构建，并确认设备可以正常开机和使用主要功能。
请勿通过固定的镜像列表判断迁移是否成功。

以下设备已经完成 Soong-only 构建和启动验证：

- OnePlus 6T (`fajita`)
- OnePlus Ace 3 / 12R (`aston(c)`)
