# Uni 构建系统

Uni 是 uwuAOSP 的本机构建调度器。它仍使用 Soong、Kati 和 Ninja 生成并执行 Android 构建规则，但在执行前保存构建图状态、拆分关键阶段，并按实时资源给高内存任务分配独立资源池。

```sh
source build/envsetup.sh
lunch uwu_nabu-cp2a-userdebug
uni -j18 otapackage
```

模块增量构建使用相同入口：

```sh
uni -j18 SystemUI Settings Launcher3QuickStep
```

## 工作原理

一次完整构建分为准备、启动和最终 Ninja 阶段：

1. **准备**：检查产品、release、variant、目标参数、环境和构建系统指纹。指纹未变化时复用已保存的 Soong/Blueprint 构建图；变化时才重新生成 Android.bp 图。
2. **建立任务集**：从产品的 Ninja 图读取可构建目标，结合历史耗时、R8 目标和内核目标计算启动顺序。长任务被提前，R8 目标分散到批次中，避免最后集中排队。
3. **启动阶段**：完整构建优先处理需要先完成的内核或历史长任务。内核链接默认与 Kotlin、Java、R8 等高内存任务错开，减少同时换页。
4. **主阶段**：剩余目标交给 Ninja 执行。全局 `-j` 仍由用户指定，Java、Kotlin、Rust、R8 和其他高内存任务通过环境变量使用各自的准入池。
5. **最终阶段**：验证构建图没有在中途变化，再执行打包、分发和 OTA 目标。阶段完成后写入状态，下一次构建可从已完成产物继续。
6. **恢复**：发生中断或持续内存压力时，保留状态和已有输出。恢复运行会重新读取磁盘状态，不删除可复用产物；自动资源池只在确认压力后降低。

Uni 不替换依赖关系检查，也不会把真实的源码变化隐藏掉。它只减少重复的准备工作和无效的等待。

## 相比 Make 做了什么

原生 `make`/`m` 把整个 Ninja 图交给统一的全局并发。Uni 在此基础上增加：

- **构建图复用**：记录产品、版本、变体、目标参数、图文件状态和构建源码指纹，条件未变化时跳过重复准备。
- **关键路径前置**：按历史任务耗时优先处理长任务，并把 R8 目标分散到多个阶段，减少尾部集中等待。
- **内核阶段隔离**：完整构建先执行必要的嵌套内核目标，避免内核链接和 R8、Kotlin 同时争用内存。
- **独立资源池**：R8、Java、Kotlin、Rust 和其他高内存任务分别计算并发，Ninja 全局 `-j` 不被改写。
- **Rust 并发约束**：同时考虑 `rustc` 数量和每个进程的 codegen units，避免 LLVM 后端把一个 Ninja 任务再次放大成过多线程。
- **分段恢复**：阶段结束保存 Ninja 恢复状态。压力中止后，已完成输出继续复用。
- **编译缓存**：记录 ccache 命中，并按磁盘余量维护缓存上限。
- **可观测性**：输出阶段耗时、内存、swap、PSI、CPU、进程峰值、资源池和源码版本。
- **终端界面**：显示 Graph、Kernel、Startup、Main、R8 和内存状态；支持详情、复制模式和完整中断。

## 资源计算公式

Uni 使用 `/proc/meminfo` 的 `MemTotal` 和 `MemAvailable`。下面的量均以 GiB 计：

```text
T = MemTotal
A = MemAvailable
R = max(4, 0.25 × T)
H = min(6, max(4, 0.20 × T))
```

`R` 是留给 Soong 非 Go 堆、系统和文件缓存的内存，`H` 是构建进程启动时预留的活动进程空间。Android.bp 分析的 Go 堆上限 `G` 按下面的规则计算：

```text
L = T - R
若 A > H：G = min(L, max(A / 2, A - H))
若 A ≤ H：G = A / 2
若结果 ≤ 0：G = max(1, T / 2)
```

这不是给单个 `soong_build` 进程设置的硬 RSS 上限。Soong 的 Go 堆之外还会分配 C/C++、文件映射和工具链内存，所以必须给非 Go 内存留下空间。

### 任务资源池

对于每种任务，`B` 是并发准入预算，`J` 是用户的全局 `-j`，资源池并发为：

```text
P = max(1, min(J, floor((A - 3 GiB) / B)))
```

预算如下：

| 任务 | `B` |
| --- | ---: |
| Java | 2 GiB |
| Rust | 2 GiB |
| 其他高内存任务 | 4 GiB |
| R8 | 5 GiB |
| Kotlin | 5 GiB |

Rust 还要限制 LLVM codegen units。若 `C` 是 codegen units，则 Rust 池不会超过：

```text
min(PRust, ceil(J / C))
```

显式设置 `NINJA_HIGHMEM_NUM_JOBS`、`NINJA_UNI_R8_NUM_JOBS`、`NINJA_UNI_RUST_NUM_JOBS`、`NINJA_UNI_JAVA_NUM_JOBS` 或 `NINJA_UNI_KOTLIN_NUM_JOBS` 时，该池使用用户给出的值。否则每个阶段重新读取 `MemAvailable` 计算，不为所有电脑写死同一组并发。

## 构建图与增量恢复

正常复用要求以下内容保持一致：

- 产品、release、variant 和目标参数
- Android.bp、Blueprint、Make、Soong、Blueprint 和产品配置指纹
- 已生成 Ninja 图及其输入文件状态
- 构建器版本和关键环境变量

源码或构建规则变化时，Uni 会重新准备受影响的图。仅在确认磁盘上的输出与当前源码匹配时，才使用恢复参数：

| 参数 | 行为 |
| --- | --- |
| `--trust-output` | 跳过恢复输出的新鲜度校验 |
| `--assume-existing` | 接受 Ninja 日志缺失但磁盘存在的输出，同时启用 `--trust-output` |
| `--force-reuse` | 跳过源码新鲜度检查，强制复用保存的图 |

这些参数不会改变依赖关系。产品、分支或构建规则变化后，应使用普通增量构建让 Uni 更新图。

## 常用命令

```sh
# 查看调度，不运行 Ninja
uni --plan -j18 otapackage

# 本次重新生成完整 R8 索引
uni --dev -j18 otapackage

# 持久开启 R8 索引自动刷新
uni --dev-auto -j18 otapackage

# 清理 Uni/Soong 构建日志，不删除编译产物
uni --clean-logs

# 关闭本次详细调试报告
uni --no-debug -j18 SystemUI
```

## Clean build 对比

![同一主机上的 clean build 用时](assets/clean-build-time.svg)

| 构建入口 | 用时 |
| --- | ---: |
| 原生 Make | 5:19:04 |
| Uni | 3:25:43 |

时间减少公式：

```text
减少比例 = 1 - Uni 用时 / Make 用时
         = 1 - 3:25:43 / 5:19:04
         = 35.5%
```

有效吞吐指数以 Make 为 100：

```text
Uni 吞吐指数 = Make 用时 / Uni 用时 × 100
             = 5:19:04 / 3:25:43 × 100
             = 155.1
```

因此这组记录中 Uni 节省 **1:53:21**，完成速度约为 Make 的 **1.55 倍**。吞吐图是同一组时间的派生指标，不代表额外的一次构建。

测试主机为 Intel Core Ultra 5 125H，14 核 18 线程，约 32 GiB 内存、50 GiB swap 和 NVMe SSD。两组时间来自同一主机上的完整 clean build，不是跨设备承诺。源码与 manifest 版本、温度、后台负载、存储缓存、ccache、swap 和目标产品都会造成误差。

## 输出与排查

构建结束时，Uni 输出一份结果摘要，包括阶段数、最低可用内存、swap-out、包或输出目录和总时长。默认日志位于 `OUT_DIR`：

```text
uwuCli-output_<timestamp>.log
uwuCli-debug-report_<timestamp>.log
```

失败时先读取日志中的第一个真实编译错误。末尾的 `FAILED: ninja` 或 `signal: killed` 只说明最终状态，不能单独确定根因。
