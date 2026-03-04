# QEMU RISC-V 上游双周报

以下内容汇总自 [qemu-devel 邮件列表](https://lore.kernel.org/qemu-devel/) 中与 RISC-V 相关的 patchset。

## 新提交的 Patchset

| 日期 | Subject | 作者 | 简要说明 |
|------|---------|------|----------|
| 02-25 | [[PATCH v2 00/16] Add RISC-V big-endian target support](https://lore.kernel.org/qemu-devel/20260225102016.200654-1-djordje.todorovic@htecgroup.com/) | Djordje Todorovic | 新增 riscv32be/riscv64be softmmu 与 linux-user 目标，为 RISC-V 提供大端序模拟支持 |
| 03-04 | [[PATCH v4 0/9] Add Zvfbfa extension support](https://lore.kernel.org/qemu-devel/20260304132514.2889449-1-max.chou@sifive.com/) | Max Chou | 实现 Zvfbfa 向量 BFloat16 算术扩展，包括 altfmt VTYPE 字段与 BF16 转换指令 |
| 03-04 | [[PATCH v4 00/14] Add OCP FP8/FP4 and RISC-V Zvfofp8min/Zvfofp4min extension support](https://lore.kernel.org/qemu-devel/20260304134006.2908449-1-max.chou@sifive.com/) | Max Chou | 实现 OCP FP8/FP4 浮点格式及 Zvfofp8min/Zvfofp4min 向量窄/宽转换指令 |
| 03-04 | [[PATCH 0/3] RISC-V Zbr0p93 extension](https://lore.kernel.org/qemu-devel/20260304121113.117299-1-james.wainwright@lowrisc.org/) | James Wainwright | 添加未批准的 Zbr0p93 CRC32 扩展指令支持与反汇编器更新 |
| 03-04 | [[PATCH 0/2] hw/riscv/riscv-iommu: Bug fixes and IPSR.PMIP support](https://lore.kernel.org/qemu-devel/20260304040959.47267-1-jay.chang@sifive.com/) | Jay Chang | 修复 RISC-V IOMMU HPM irq_overflow_left 残留值 bug，并添加 IPSR.PMIP RW1C 支持 |
| 02-27 | [[PATCH 0/3] riscv: AIA: Add in-kernel irqchips save and restore function support](https://lore.kernel.org/qemu-devel/20260227180104794YvW9Rb2I_kAGzUruZL11Q@zte.com.cn/) | Liu Xuemei | 为 KVM RISC-V AIA 中断控制器添加保存/恢复功能，支持热迁移场景 |
| 02-26 | [[PATCH] MAINTAINERS: Add myself as a reviewer for RISC-V TCG CPUs](https://lore.kernel.org/qemu-devel/20260226102008.146928-1-chao.liu.zevorn@gmail.com/) | Chao Liu | 将 Chao Liu 添加为 RISC-V TCG CPU 子系统的 reviewer |
| 02-27 | [[PATCH] hw/riscv: Remove deprecated 'riscv, delegate' device-tree property](https://lore.kernel.org/qemu-devel/20260227232838.23392-1-philmd@linaro.org/) | Philippe Mathieu-Daudé | 移除已弃用的 `riscv,delegate` 设备树属性 |
| 02-20 | [[PATCH qemu 00/11] Introduce ot-earlgrey machine](https://lore.kernel.org/qemu-devel/177159976712.8279.7732381632410882915-0@git.sr.ht/) | Lex Bailey | 引入 OpenTitan EarlGrey RISC-V 机器，包含 LowRisc Ibex hart 与 alert handler 外设 |

---

## Single-binary 相关（涉及 RISC-V 重构）

| 日期 | Subject | 作者 | 简要说明 |
|------|---------|------|----------|
| 02-18 | [[PATCH v6 0/7] single-binary: Drop TARGET_PHYS_ADDR_SPACE_BITS](https://lore.kernel.org/qemu-devel/20260218-phys_addr-v6-0-a603bf363218@rev.ng/) | Anton Johansson | 移除 TARGET_PHYS_ADDR_SPACE_BITS，改用属性设置 RISC-V IOMMU 物理地址空间大小 |
| 02-19 | [[PATCH v2 00/50] gdbstub: Build once on various targets (single-binary)](https://lore.kernel.org/qemu-devel/20260219191955.83815-1-philmd@linaro.org/) | Philippe Mathieu-Daudé | 将 gdbstub 统一为单次编译，其中包含多个 RISC-V 补丁（提取 monitor 代码、移除 target_ulong 依赖等） |
| 02-25 | [[PULL 00/70] Single binary patches for 2026-02-26](https://lore.kernel.org/qemu-devel/20260225231411.96482-1-philmd@linaro.org/) | Philippe Mathieu-Daudé | 合并 70 个 single-binary 补丁，包含上述 RISC-V IOMMU PAS、gdbstub、monitor 重构等 |

---

## 仍在活跃 Review 的早期 Patchset

| Subject | 作者 | Review 动态 |
|---------|------|-------------|
| [[PATCH v6 0/4] hw/riscv/virt: Add acpi ged and powerdown support](https://lore.kernel.org/qemu-devel/20260209102128930gN_7xu7bNIZcEawnIz3c7@zte.com.cn/) | Liu Xuemei | Michael S. Tsirkin 于 02-20 回复讨论 |
| [[PATCH v5 0/6] riscv: implement Ssqosid extension and CBQRI controllers](https://lore.kernel.org/qemu-devel/20260201-riscv-ssqosid-cbqri-v5-0-273ea4a21703@kernel.org/) | Drew Fustini | Radim Krcmar 于 03-02 逐 patch 回复 review 意见 |
| [[PATCH v2] target/riscv: Support Smpmpmt extension](https://lore.kernel.org/qemu-devel/20251107023136.89181-1-jay.chang@sifive.com/) | Jay Chang | Jay Chang、Chao Liu 于 03-04 回复讨论 PMP 内存类型扩展 |
| [[PATCH] hw/dma: sifive_pdma: Set done bit upon completion](https://lore.kernel.org/qemu-devel/20260304030816.33209-1-jay.chang@sifive.com/) | Jay Chang | Alistair Francis 于 03-04 回复确认 |

---

## QEMU 训练营 2026

QEMU 训练营是在清华大学陈渝老师和李明老师的倡议下，由格维开源社区 (GTOC) 发起的公益性技术训练营。[QEMU 训练营 2026](https://qemu-camp.org/tutorial/2026/) 由格维开源社区（GTOC）和华中科技大学开放原子开源俱乐部主办，课程以 RISC-V 为主要目标架构，围绕 QEMU 展开四个阶段的递进式教学。

### 各阶段进展

| 阶段 | 主题 | 当前状态 |
|------|------|----------|
| ch0 导学 | 编程基础、体系结构基础、开发环境搭建 | 已完成 |
| ch1 基础 | QOM、MemoryRegion、启动流程、调试技巧 | 已完成（6 篇） |
| ch2 专业 | TCG、CPU 建模、外设建模、GPGPU、Rust for QEMU | 编写中（15 篇，持续更新） |
| ch3 项目 | CXL、K230、Wine CE、QEMU Agent | 编写中（4 篇） |

下面重点介绍专业阶段 GPGPU 实验。

---

### RISC-V GPGPU 实验

ch2 专业阶段新增了 **GPGPU 体系结构与建模**专题（2 篇），是本期训练营的重点实验项目。该实验基于 QEMU 实现了一个 RISC-V GPGPU 设备原型，目前已完成原型开发。

简化架构图（参考 Vortex 设计）：

``` {data-ppt-lines="64"}
+----------------------------------------------------------------------------------------+
| Guest OS (RISC-V or Other Arch)    GPGPU App/Kernel --> GPGPU Driver (MMIO + DMA)      |
+--------------------------------------+-------------------------------------------------+
                                       | PCIe
=======================================|==================================================
                         QEMU          | Device Model
=======================================|==================================================
+--------------------------------------v-------------------------------------------------+
| PCIe Frontend (gpgpu.c)                                                                |
|                                                                                        |
| BAR0 (CTRL 1MB)             BAR2 (VRAM 64MB)             BAR4 (DOORBELL 64KB)          |
| +----------------------+    +----------------------+      +----------------------+     |
| | Kernel Dispatch      |    |                      |      | DMA Engine           |     |
| |   kernel_addr/args   |    | BAR map (PCIe window)+--+   |   src/dst/size/ctrl  |     |
| |   grid_dim  (X,Y,Z)  |    |                      |  |   | MSI-X (4 vectors)    |     |
| |   block_dim (X,Y,Z)  |    |                      |  |   | IRQ enable/pending   |     |
| | Global Control       |    +----------------------+  |   +----------------------+     |
| | IRQ Status           |                              |                                |
| +----------+-----------+                              |                                |
|            | dispatch                                 | map                            |
+------------+------------------------------------------+--------------------------------+
             |                                          |
+------------v------------------------------------------+--------------------------------+
| SIMT Backend (gpgpu_core.c)                           |                                |
|                                                       |                                |
| +----------------------+                              |                                |
| | VRAM (64MB)          | <-- PCIe BAR2 maps here +----+                                |
| | GPU Local Memory     |                                                               |
| +----------^-----------+                                                               |
|            | ld/st                                                                     |
|                                                                                        |
| Grid --> Block(0,0)  Block(1,0)  Block(2,0) ...                                        |
|               |                                                                        |
|               v                                                                        |
|       +--- Block ------------------------------------------------------------+         |
|       |                                                                      |         |
|       |  +--- Warp 0 --------+  +--- Warp 1 --------+  +--- Warp 2 --+       |         |
|       |  | Lane 0 .. Lane 31 |  | Lane 0 .. Lane 31 |  | Lane 0..31  |       |         |
|       |  | +----+     +----+ |  | +----+     +----+ |  | +----+      |       |         |
|       |  | | PC |     | PC | |  | | PC |     | PC | |  | | PC |      |  ...  |         |
|       |  | | x0 |     | x0 | |  | | x0 |     | x0 | |  | | x0 |      |       |         |
|       |  | |... |     |... | |  | |... |     |... | |  | |... |      |       |         |
|       |  | |x31 |     |x31 | |  | |x31 |     |x31 | |  | |x31 |      |       |         |
|       |  | +----+     +----+ |  | +----+     +----+ |  | +----+      |       |         |
|       |  | active_mask (32b) |  | active_mask (32b) |  | active_mask |       |         |
|       |  +-------------------+  +-------------------+  +-------------+       |         |
|       |                                                                      |         |
|       | barrier / sync            mhartid = [block|warp|thread]              |         |
|       +----------------------------------------------------------------------+         |
|                                                                                        |
+----------------------------------------------------------------------------------------+
```

---

**主要特性**：

- SIMT 执行模型：支持 Thread/Block/Grid 层级的线程组织与 Warp 调度
- PCIe 设备实现：作为标准 PCIe 设备挂载，支持 BAR/MMIO、DMA、MSI-X
- QTest 测试框架：集成 QEMU QTest 基础设施进行设备级自动化测试
- 前后端分层架构：PCIe 前端负责命令队列与寄存器交互，cmodel 后端执行 kernel 计算

**考核方式**：

- 基于 Qtest 框架搭建 GPGPU 测题集，用于验证功能完备性，根据测题 Pass 数目计算学员得分
- 开放题目：基于该 GPGPU 设计一个简单的 AI 软件栈（编程模型+驱动），类 cuda 风格
- 开放题目：直接将 Vortex 的 simx 集成到 QEMU 当中，并将其 AI 软件栈适配 ArceOS