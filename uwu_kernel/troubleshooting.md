# uwu_kernel 故障排查

## 找不到 kernel 模块

确认 `SOONG_KERNEL_MODULE` 在 BoardConfig 阶段设置，并且模块名称和 Android.bp
一致：

```make
BOARD_USES_SOONG_KERNEL := true
SOONG_KERNEL_MODULE := //device/<vendor>/<device>:kernel
```

同时确认产品包含：

```make
PRODUCT_PACKAGES += kernel
```

不要只在产品 Makefile 后期设置 `SOONG_KERNEL_MODULE`。Soong mutator 和 fsgen 需要在
模块分析阶段读取它。

## 找不到配置文件

无斜杠的配置名按以下路径解析：

```text
<kernel_dir>/arch/<config_arch>/configs/<name>
```

包含路径的配置名按源码根目录解析，例如 `vendor/common.config` 对应源码根目录下的
`vendor/common.config`。`x86_64` 的 defconfig 目录会转换为 `arch/x86/configs`。
检查 `config.defconfig`、所有 fragment 是否存在，并确认
fragment 的顺序没有依赖旧 Make 的隐式变量展开。

如果使用 `prebuilt` kernel，源码配置属性不会执行；应分别设置
`prebuilt_config`、`prebuilt_headers` 和 `prebuilt_modules` 来提供对应输出。

## 修改源码后没有重新编译

检查：

1. 修改文件是否位于 `kernel_dir` 或 `external_module_root`；
2. `source_deps/source.d` 是否包含对应目录；
3. 是否手工修改了 `out/soong` 中间文件；
4. 是否把源码放在了目录依赖之外且没有加入 `srcs`；
5. action 的输出时间戳是否被外部脚本覆盖。

不要通过 `srcs: ["**/*"]` 解决问题。应修正源码根目录或补充最小的额外输入。

## Headers 没有更新

确认 `generated_kernel_includes` 依赖的是当前 `uwu_kernel`，而不是
`generated_kernel_includes_legacy`。然后检查：

```text
out/soong/.intermediates/device/<vendor>/<device>/kernel/
  <variant>/headers.timestamp
out/soong/.intermediates/device/<vendor>/<device>/kernel/
  <variant>/source_deps/source.d
```

headers action 会执行 Kbuild `headers_install`，随后运行
`vendor/uwu/build/tools/clean_headers.sh`。如果 action 已执行但结果不完整，应检查
kernel 的 UAPI 导出规则，而不是手工复制 headers。

## DTB 或 DTBO 失败

依次检查：

- `dtb.enabled` 和 `dtbo.enabled` 是否符合 `qcom_merge` 的要求；
- Kbuild `target` 是否真的生成了对应 DT 文件；
- `input_globs` 是否匹配实际输出；
- DTBO `page_size` 是否与 BoardConfig 和 bootloader 要求一致；
- `custom_command` 是否使用了合法的 `$(kernelDir)`、`$(kernelOut)` 和 `$(out)`；
- 使用 QCOM 合并时，`merge_dtbs.py` 的输入目录是否包含基础 DTB 和 techpack DT。

不要直接编辑生成的 `.dtb` 或 `.dtbo`。

## Modules 构建成功但没有安装

确认模块同时出现在对应的 install list 和 load list。还要检查：

- 模块所属分区是否是 `system_dlkm`、`vendor_dlkm`、`vendor_ramdisk` 或 `recovery`；
- `external_module_root` 是否正确；
- 普通 external module 是否应该改为 `path:kbuild`；
- `module_aliases` 是否使用 `old.ko:new.ko`；
- `auto_collect_deps` 是否需要启用；
- blocklist 是否误用了模块 basename 以外的路径。

不要把内部生成的 `kernel_modules_*` 模块手工添加到 `PRODUCT_PACKAGES`。

## 工具链或 Perl 错误

检查实际 action 是否使用 tree 内工具链和工具路径。常见问题包括：

- `clang_version` 不存在；
- 自定义 `clang_path` 缺少 `bin` 或 `lib`；
- `DTC_EXT` 使用了错误的 host 输出；
- 外部模块依赖宿主系统 Perl；
- `make_command` 指向的工具不支持当前 Kbuild 参数。

优先修正模块属性，不要在设备 Makefile 中重新拼接一套 PATH。

如果启用 `rbe_wrapper`，确认它是完整的 rewrapper 命令，并且构建环境设置了 `TOP`。
