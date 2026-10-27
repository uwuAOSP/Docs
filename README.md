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

### 配置 OTA 更新 {#ota-updates}

原生更新器从维护者提供的 HTTPS 地址读取更新列表，再下载和安装设备对应的 OTA 包。本节使用 `packages/apps/Updater` 中的 `Updater`，应用包名为 `org.uwuaosp.updater`。

#### 1. 设置更新源

在设备树的 `device.mk` 或等效产品 Makefile 中加入资源覆盖目录。若已有该目录，直接复用：

```make
PRODUCT_PACKAGE_OVERLAYS += device/<vendor>/<device>/overlay
```

在 `device/<vendor>/<device>/overlay/packages/apps/Updater/app/src/main/res/values/config.xml` 中配置更新列表地址：

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="updater_server_url" translatable="false">https://ota.example.org/{device}/{type}.json</string>
</resources>
```

将 `<vendor>`、`<device>` 和示例域名替换为实际值。URL 可使用以下占位符，也可以直接填写固定的 JSON 地址：

| 占位符 | 运行时取值 |
| --- | --- |
| `{device}` | `ro.uwu.device` |
| `{type}` | `ro.uwu.releasetype`，转换为小写 |
| `{incr}` | `ro.build.version.incremental`，用于按已安装构建返回增量更新 |

例如，设备代号为 `sheng`、渠道为 `UNOFFICIAL` 时，上述地址会变成 `https://ota.example.org/sheng/unofficial.json`。更新源默认为空；未配置时不执行在线检查，本地导入仍可使用。

#### 2. 生成并发布更新包

使用同一套发布密钥为设备构建 OTA。密钥初始化和签名命令见 [uni 的 OTA 签名说明](https://uwuaosp.uwuniverse.org/docs/uni/#ota-签名)。

```sh
uni otapackage --sign-keys ~/.android-certs
```

将最终签名 OTA ZIP 上传到可直接下载的 HTTPS 地址。服务器需支持 HTTP Range 请求，以便暂停后继续下载。发布前读取该文件的元数据、SHA-256 和字节数：

```sh
ota_package=/path/to/final-signed-ota.zip
unzip -p "$ota_package" META-INF/com/android/metadata
sha256sum "$ota_package"
stat -c %s "$ota_package"
```

将 `ota_package` 改为构建输出的实际路径。不要使用 Fastboot 包或 target-files ZIP，也不要在计算校验值后重新打包或修改 OTA。

#### 3. 发布更新列表

在更新源地址提供 JSON 数组。每条更新只包含一个文件；没有可用更新时返回 `[]`，不要套用旧版接口的 `{"response": [...]}` 格式。

下面是结构示例，所有字段均需替换为目标构建和最终签名 ZIP 的实际值：

```json
[
  {
    "datetime": 1793059200,
    "type": "UNOFFICIAL",
    "version": "17.0.157",
    "files": [
      {
        "filename": "uwuAOSP-example-ota.zip",
        "size": 2147483648,
        "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
        "url": "https://ota.example.org/files/uwuAOSP-example-ota.zip",
        "os_sdk_level": 36,
        "os_patch_level": "2026-10-05"
      }
    ]
  }
]
```

| 字段 | 填写方式 |
| --- | --- |
| `datetime` | OTA 元数据的 `post-timestamp`，单位为秒，不是上传时间 |
| `type` | 目标系统的 `ro.uwu.releasetype`；仅向相同渠道提供更新，不区分大小写 |
| `version` | 目标系统的 `ro.uwu.release`，不能用已安装版本推算 |
| `filename` | 最终 OTA ZIP 的文件名，不包含目录 |
| `size` | 最终 OTA ZIP 的字节数 |
| `sha256` | 最终 OTA ZIP 的 64 位十六进制 SHA-256 |
| `url` | OTA ZIP 的 HTTPS 直链，不是下载介绍页 |
| `os_sdk_level` | OTA 元数据的 `post-sdk-level`，填写为整数 |
| `os_patch_level` | OTA 元数据的 `post-security-patch-level`，格式为 `YYYY-MM-DD`，可省略 |

目标版本和渠道可从本次构建输出的 `product/etc/build.prop` 读取，不能沿用旧系统的值。

更新列表响应不能超过 1 MiB，地址不能依赖 HTTP 重定向。客户端只显示时间戳晚于已安装系统、渠道相同且 SDK 不低于当前系统的包。服务端还需按设备划分列表；使用增量 OTA 时，也要按 `{incr}` 对应的源构建返回匹配的包。

A/B 设备如需流式安装，在文件对象中增加 `ota_property_files`，原样填写最终 ZIP 元数据里的 `ota-property-files`。其中必须包含 `payload_metadata.bin`、`payload.bin` 和 `payload_properties.txt` 的字节范围，下载服务器需支持这些范围请求。不提供该字段时，使用完整下载后安装；非 A/B 设备使用 Recovery 安装。

官网更新日志与 OTA 更新源相互独立。更新日志按版本存放在 `uwuAOSP/uwu_ChangeLOG` 的 `uwu-{版本号}` 分支。更新器读取官网 `/blog/uwu-{版本号}/`，不能把该网页地址填入 `updater_server_url`。

#### 4. 验证配置

1. 重新构建并安装包含资源覆盖的系统包，在设备上读取更新器实际采用的地址：

   ```sh
   adb shell cmd overlay lookup org.uwuaosp.updater org.uwuaosp.updater:string/updater_server_url
   adb shell getprop ro.uwu.device
   adb shell getprop ro.uwu.releasetype
   adb shell getprop ro.build.version.incremental
   ```

2. 打开 **设置 > 系统 > 软件更新**，检查空列表、新更新，以及下载暂停和继续。
3. 在测试设备上完成校验、安装和重启。A/B 设备确认更新后的系统可启动；非 A/B 设备确认 Recovery 能安装该包。

更新包必须通过系统信任的 OTA 签名及设备、源构建和版本兼容性校验。SHA-256 不能代替签名。已有设备必须继续信任同一套发布密钥；密钥应备份到独立存储，不能上传到更新服务器。

参考：[AOSP OTA 包与元数据](https://source.android.com/docs/core/ota/tools)、[资源覆盖](https://source.android.com/docs/core/runtime/rros)。

### Soong-only 迁移

> [!WARNING]
> 开启 Soong only 不是启动 uwuAOSP 的必须条件。

> [!NOTE]
> 有关 Soong only能给您的设备带来的益处以及限制，请参见 [此处](https://uwuaosp.uwuniverse.org/docs/soong-only/)

为了迁移到 Soong only, uwuAOSP 建议您做如下修改：

#### 迁移到 uwu_prebuilt_image

uwuCLI 将会在第一阶段结束后询问您是否写入第一阶段修改。写入修改后，uwuCLI 将会询问您是否转换 radio image 集成方式（如果设备通过 `$(call add-radio-file,...)` 添加了固件镜像的话）。

如果要开启 Soong only, 这一步是必须的。

#### 迁移到 uwu_kernel

跟随此处教程进行迁移：[uwu_kernel migration](https://uwuaosp.uwuniverse.org/docs/soong-only/uwu_kernel/migration.html)。

有关详细信息，请参见 [uwu_kernel 文档](https://uwuaosp.uwuniverse.org/docs/soong-only/uwu_kernel/)。

#### 检查阻碍您迁移到 Soong only 的包

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
