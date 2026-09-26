# uwuAOSP Bringup Guide (17.0+)

本教程将指引您了解构建 uwuAOSP 所需设备树侧最小修改，以及为了获得更好的开发体验，还可以进行的额外优化。

## 必做项（第一阶段）

方便起见，请使用 uwuCLI 进行 bringup：

```bash
uwu
```

选择语言，选择“改造 LineageOS 产品设备树”后，脚本将向您询问几个问题。

其中，第三个问题的维护者名称有如下要求：名称最多 64 个字符，可使用字母、数字、空格、点、下划线、连字符和 @。您可以留空该字段。

维护者信息将被显示在 设置 -> 系统 -> 软件更新 中。

### 手动改造

请在设备树 `device.mk` 或等效 Makefile 中配置以下 Flag：

```
# Device type
# Select from phone, tablet, or foldable.
UWU_DEVICE_TYPE := phone

# Whether the device supports telephony
# Select from true or false.
UWU_SUPPORTS_TELEPHONY := true

# OPTIONAL: Device maintainer
UWU_MAINTAINER := Akaza_Akari
```

## 可选项（第二阶段）

> [!WARNING]
> 开启 Soong only 不是启动 uwuAOSP 的必须条件。

> [!NOTE]
> 有关 Soong only能给您的设备带来的益处以及限制，请参见 [此处](https://uwuaosp.uwuniverse.org/docs/soong-only/)

为了迁移到 Soong only, uwuAOSP 建议您做如下修改：

### 迁移到 uwu_prebuilt_image

uwuCLI 将会在第一阶段结束后询问您是否写入第一阶段修改。写入修改后，uwuCLI 将会询问您是否转换 radio image 集成方式（如果设备通过 `$(call add-radio-file,...)` 添加了固件镜像的话）。

如果要开启 Soong only, 这一步是必须的。

### 迁移到 uwu_kernel

跟随此处教程进行迁移：[uwu_kernel migration](https://uwuaosp.uwuniverse.org/docs/soong-only/uwu_kernel/migration.html)。

有关详细信息，请参见 [uwu_kernel 文档](https://uwuaosp.uwuniverse.org/docs/soong-only/uwu_kernel/)。

### 检查阻碍您迁移到 Soong only 的包

请在完成 lunch 后执行：

```bash
uwu inspect soong-only
```

uwuCLI 将花费一些时间来检查您的设备及所有依赖。这一时间视设备情况而定。

若脚本告知您以下结果，那么您可以切换到 Soong only:

```text
Conclusion
--------------------
  READY
  No selected package still depends on Android.mk.
```
