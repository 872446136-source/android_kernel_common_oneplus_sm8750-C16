<div align="center">

# OnePlus 13T SM8750 Custom Kernel

基于 Android Common Kernel 6.6.118 的 OnePlus 13T 性能与内存管理定制内核

[![Device][device-badge]][device-url]
![SoC][soc-badge]
[![Kernel][kernel-badge]][kernel-url]
![Platform][platform-badge]
![Page Size][page-size-badge]
[![License][license-badge]][license-url]
![Repository][repository-badge]

</div>

[device-badge]: https://img.shields.io/badge/Device-OnePlus%2013T-EA0029?style=for-the-badge&logo=oneplus&logoColor=white
[device-url]: https://www.oneplus.com/
[soc-badge]: https://img.shields.io/badge/SoC-SM8750%20%7C%20Snapdragon%208%20Elite-red?style=for-the-badge
[kernel-badge]: https://img.shields.io/badge/Kernel-6.6.118-FCC624?style=for-the-badge&logo=linux&logoColor=black
[kernel-url]: https://www.kernel.org/
[platform-badge]: https://img.shields.io/badge/Platform-Android%20GKI-3DDC84?style=for-the-badge&logo=android&logoColor=white
[page-size-badge]: https://img.shields.io/badge/Page%20Size-4%20KiB-0078D4?style=for-the-badge
[license-badge]: https://img.shields.io/badge/License-GPL--2.0-2C3E50?style=for-the-badge
[license-url]: COPYING
[repository-badge]: https://img.shields.io/badge/Repository-Private-6E7781?style=for-the-badge&logo=github&logoColor=white

> [!IMPORTANT]
> 当前主要开发与验证设备是 OnePlus 13T（项目代号 `24821`）。仓库中存在其他
> OPLUS/realme 项目的设备树覆写配置，理论O/加/真系列8E设备通用，但这不等于对应设备已经完成启动、功能和稳定性验证。

## 免责声明

刷写自定义内核存在一定风险,可能导致**设备变砖、数据丢失或 SafetyNet / Play Integrity 校验失败**。
刷入前请务必备份重要数据。因使用本内核造成的任何损失,作者概不负责。

**刷入即代表你已知晓并愿意承担上述风险。**

### 原创实现

- **Crystal HybridSwap**：替代标准 `CONFIG_ZRAM` 的私有 zram 实现，同时保留
  `/dev/zramX`、`/sys/class/zram-control` 和常用 `/sys/block/zramX` 用户态 ABI，盘活了hybridswap。
- **ZMS packed backing store**：将压缩对象打包到 4 KiB backing blocks，支持受控写回、
  batch-in、quota、设备寿命限制和失败退避，与上游压缩回写不同的是，上游 zram 写回先将每个 slot 解压成完整页面、
  再以一页一个 backing block 落盘不同，ZMS 直接保存压缩流，并尽可能让多个对象共享同一个
  4 KiB block，以减少后端空间占用和物理写入，这也和上游未合并的压缩流回写(无论压缩后有多大，都只写入一个4 KiB block)不同，ZMS能节省更多空间；
- **SDDC 相似性压缩**：对完全相同的压缩流使用 alias，对相似数据使用 delta 表示；
  资源不足或校验失败时回退到普通压缩路径。
- **LZ4KD**：当前 4 KiB 构建默认压缩器，提供普通压缩与 SDDC delta codec。
- **SDDC 原生 ZMS 写回**：通过引用代际、slot 状态和 wire format 校验，将合法的
  alias/delta 表示写入 ZMS。
- 支持 idle/huge/incompressible page 写回、预取、batch-in、per-memcg 控制、eventfd
  压力通知和多层统计接口。
- 提供 `hybridswap_report`、`hybridswap_crystal_stat`、`sddc_stat`、`zms_stat` 等诊断节点。

### 内存回收与低内存优化

- **MGLRU 默认启用**，并包含 dirty/writeback 处理、folio 隔离、扫描批次及 refault 路径优化。
- **UKSM 默认启用**，使用 Android CPU governor，并包含 rmap walk、资源释放和异常路径修复。
- **Kcompressd-Unofficial**：将部分 kswapd swapout 压缩工作异步移交给独立内核线程。
- **le9uo working-set protection**：通过匿名页与 clean file page 水位减少低内存下的工作集抖动。
- 支持 memcg-aware swap 分配和仅回收匿名页的 proactive reclaim。
- 包含 page allocator、vmalloc、swap fault、THP 和 folio 回收路径的延迟与稳定性修复。

### CPU 调度与功耗

- **HMBIRD 调度类内建**，包含 cgroup deadline、SLIM/WALT utilization tracking、shadow tick
  及运行时控制接口。
- HMBIRD 的 DSQ、remote task pull、timeout、irq_work 和 fork 异常路径修复。
- EAS 能耗计算、schedutil/iowait boost、idle load balance 和 overutilized 判断优化。
- `SM_IDLE` 调度/idle 快速路径，降低部分空闲重入开销。
- menu 与 TEO cpuidle 决策优化，并避免存在 pending IPI 时错误进入 CPU power-down。
- power-efficient workqueue、scheduler cluster 和 Android wakelock 配置调优。
- **Boeffla Wakelock Blocker**，提供可配置的 wakelock 屏蔽接口。

### I/O 与文件系统

- **BFQ I/O scheduler** 及 cgroup 支持已编入内核。
- F2FS compression、ATGC 和 GC_MERGE；后台 GC 线程使用 idle 调度/I/O 优先级并包含
  多项 GC、卸载和压缩写回修复。
- EROFS 启用 per-CPU kthread，并修复 LZ4 解压 bounce page 分配失败时的重试路径。
- zsmalloc compact、zram memory tracking 和 backing-device 统计由 Crystal 数据面提供。
- twrp/REC可以正常进入

### 网络

- **BBRv3** TCP 拥塞控制，当前配置为默认 TCP congestion control。
- **FQ-CoDel** 为默认网络队列规则，同时保留 FQ 支持。
- 默认启用 TCP ECN 协商，并包含面向移动网络延迟的 TCP 路径调整。
- 保留 Android GKI 所需的 netfilter、IP set、XDP socket 和多路由表能力。

### SM8750 与 OPLUS 适配

- **Device-tree overwriter** 在内核启动阶段按项目配置覆写或创建设备树属性；当前包含
  OnePlus 13T (`24821`) 的充电与监控相关配置。
- 仓库还保留 OnePlus 13 (`23821`)、OnePlus Ace 5 Pro (`24811`) 以及部分 realme
  项目的覆写配置，均应视为未验证适配素材。
- **Module overlay framework** 可在模块加载时拦截目标模块，并使用内核内嵌的 zstd
  压缩版本替换用户态传入模块。
- Qualcomm SCM、充电协议和 OPLUS vendor 兼容路径包含设备专用修复。


## 致谢

感谢 Linux 内核社区、Android Common Kernel、OPLUS/OnePlus 平台开发者，以及 CrystalFrostwork、
UKSM、BBRv3、Boeffla Wakelock Blocker 等相关项目和所有提交者。具体作者、来源与
`Signed-off-by` 信息以 Git 历史和源码文件头为准。
