# QEMU 训练营 2025 专业阶段实验解析

!!! note "主要贡献者"

    - 作者：[@LordaeronESZ](https://github.com/LordaeronESZ)
    - 原文：[QEMU 训练营 2025 专业阶段实验解析 - Chaoqun Zheng][qemu-camp-2025-note-link]

笔者的研究方向主要是操作系统与虚拟化，平时接触最多的开源项目除了 Linux 内核外，便是 QEMU 了，不只是将它作为一个全系统模拟器和 VMM 来使用，还涉及到基于它的增量开发的工作。但这些基本都还是边做边学的，主要原因在于市面上一直缺乏系统性介绍 QEMU 的书籍和资料（笔者知道的仅有一本《QEMU/KVM 源码解析与应用》，但其主要专注于 QEMU 的 VMM 用途，对模拟器的介绍不是重点）。得知 OpenCamp 将举办 QEMU 训练营，笔者也是在持续关注，并且在第一时间报名并开展了学习。

截至现在，笔者已经完成了基础阶段和专业阶段的学习，正在等待项目阶段的开放。本文将分享笔者专业阶段实验的解题思路和方法，作为个人的一个记录，也为后续参与训练营的同学提供参考。

## 实验概述

专业阶段的实验平台是由训练营主办方专门设计的一个简易的教学用板卡 G233，对它的介绍可以参考文档：[b. G233 Datasheet - Learning QEMU Docs][qemu-datasheet-link]。

实验共有十个，归类下来总共有三类：lab1 主要是根据手册的说明，补全对 G233 板卡的模拟代码；lab2 ~ lab5 则是为 RISC-V 指令集新增自定义指令；lab6 ~ lab10 是为 G233 新增一个自定义的 SPI 控制器设备。下面笔者也将按照这样的归类，分三个部分来一一介绍。


## lab1 G233 板卡模拟

lab1 目标是实现对 G233 板卡的模拟，但其实绝大部分的代码已经写好了，需要做的只是将部分缺失的代码补充完整，因此本实验的主要目的还是熟悉 QEMU 设备模拟相关流程和一些 API 接口的使用，实验过程中可以参考 QEMU 代码目录下其他板卡（比如笔者参考了 `xiangshan_kmh.c`）的代码实现，后面的 SPI 设备模拟也是同理。

G233 板卡的组件构成如下：

```c
typedef struct G233SoCState {
    /*< private >*/
    DeviceState parent_obj;

    /*< public >*/
    RISCVHartArrayState cpus;
    DeviceState *plic;
    DeviceState *uart0;
    DeviceState *pwm0;
    SIFIVEGPIOState gpio;
    MemoryRegion mask_rom;
} G233SoCState;

typedef struct G233MachineState {
    /*< private >*/
    MachineState parent_obj;

    /*< public >*/
    G233SoCState soc;

} G233MachineState;
```

需要自定义的类型有两个：G233 主板（`G233MachineState`）和 G233 的 SoC（`G233SocState`），其他的设备如 PLIC 和 UART 等都直接采用了 QEMU 代码库中的现有实现。我们下面直接以 SoC 为例介绍 QEMU 自定义类型的代码实现流程。

下列代码就是常见的添加一个 QEMU 新类型的基本操作，基本上对于绝大部分的类型都适用，而其底层实现原理不在本文的讨论范围内。

```c
static const TypeInfo g233_soc_type_info = {
    .name = TYPE_RISCV_G233_SOC,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(G233SoCState),
    .instance_init = g233_soc_init,
    .class_init = g233_soc_class_init,
};

static void g233_soc_register_types(void)
{
    type_register_static(&g233_soc_type_info);
}

type_init(g233_soc_register_types)
```

`.name`、`.parent`、`.instance_size` 这些字段想必不用过多介绍了，我们主要关注 `class_init` 和 `instance_init` 接口的实现。

这两个函数接口的区别顾名思义，一个用于类的初始化，一个用于实例（对象）的初始化。如果用 C++ 这样面向对象的编程语言的概念来套的话，可以理解为：`class_init` 用于完成 **类的每个对象所共有属性和状态** 的初始化，如成员函数、静态成员变量等；而 `instance_init` 则用于完成 **类的每个对象所私有属性和状态** 的初始化，如非静态成员变量，更直接一点说，`instance_init` 接口其实就相当于类的构造函数。

相信有了上面的认识，要理解 G233 SoC 的初始化代码也不那么困难了。G233 的 `class_init` 接口完成的工作就是设置其父类 `DeviceClass` 的 `realize` 接口为 G233 SoC 类特定的接口 `g233_soc_realize`，这样运行时动态绑定的过程是不是很熟悉？没错，这其实就是 QOM 实现的类似 **C++ 虚函数** 的功能，子类在程序运行时动态绑定父类的函数接口，从而实现 **运行时多态** 。因此，我们在理解代码时不妨将 `class_init` 接口看作是一个间接层，它只是进行一系列的动态绑定操作，实际需要关注的则是子类对父类接口的特定实现，在 G233 SoC 中，它只实现了 `realize` 接口。

```c
static void g233_soc_class_init(ObjectClass *oc, const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(oc);

    dc->realize = g233_soc_realize;
}
```

我们接下来具体分析 `realize` 接口的实现，其中新增的代码用 `//+` 来进行标识。

我们需要新增的代码是 CPU 的实例化，实例化的过程是先设置设备一些属性（property）：

- `num-harts` 表示 CPU 的核心数量，通常通过 QEMU 的启动参数 -smp 来指定，由于 g233_machine_class_init 中将最大 CPU 数量设置为了 1，因此这里我们直接指定为 1。

- `hartid-base` 表示 CPU 核心（HART）的起始编号，通常指定为 0 即可。

- `cpu-type` 表示 CPU 的类型，实验代码已经在 `target/riscv/cpu-qom.h` 中声明好了，直接添加即可。

- `resetvec` 需要重点关注，它表示 CPU 在上电或复位后 PC 指向的指令地址，`g233_soc_init` 中的注释也提到了它为 `0x1004`，即 ROM 中的固件代码，用于实际跳转到程序中开始执行。

!!! tip

    关于 `RISCVHartArrayState` 有哪些属性可以查看代码 `hw/riscv/riscv_hart.c:riscv_harts_props` 处的内容。

最后调用 `qdev_realize` 的封装函数 `sysbus_realize` 完成实例化。

后面的一系列代码通过调用 plic 和 uart 等设备的特定接口，进行设备的创建，并将设备寄存器映射到物理地址空间中，也就是 MMIO。

```c
static void g233_soc_realize(DeviceState *dev, Error **errp)
{
    MachineState *ms = MACHINE(qdev_get_machine());
    G233SoCState *s = RISCV_G233_SOC(dev);
    MemoryRegion *sys_mem = get_system_memory();
    const MemMapEntry *memmap = g233_memmap;
    //+ uint32_t num_harts = ms->smp.cpus;

    /* CPUs realize */
    //+ qdev_prop_set_uint32(DEVICE(&s->cpus), "num-harts", num_harts);
    //+ qdev_prop_set_uint32(DEVICE(&s->cpus), "hartid-base", 0);
    //+ qdev_prop_set_string(DEVICE(&s->cpus), "cpu-type",
    //+                 TYPE_RISCV_CPU_GEVICO_G233);
    //+ qdev_prop_set_uint64(DEVICE(&s->cpus), "resetvec",
    //+                 memmap[G233_DEV_MROM].base + 0x4);
    //+ sysbus_realize(SYS_BUS_DEVICE(&s->cpus), &error_fatal);

    /* Mask ROM */
    memory_region_init_rom(&s->mask_rom, OBJECT(dev), "riscv.g233.mrom",
                           memmap[G233_DEV_MROM].size, &error_fatal);
    memory_region_add_subregion(sys_mem, memmap[G233_DEV_MROM].base,
                                &s->mask_rom);

    /* MMIO */
    s->plic = sifive_plic_create(memmap[G233_DEV_PLIC].base,
                                 (char *)G233_PLIC_HART_CONFIG, ms->smp.cpus, 0,
                                 G233_PLIC_NUM_SOURCES,
                                 G233_PLIC_NUM_PRIORITIES,
                                 G233_PLIC_PRIORITY_BASE,
                                 G233_PLIC_PENDING_BASE,
                                 G233_PLIC_ENABLE_BASE,
                                 G233_PLIC_ENABLE_STRIDE,
                                 G233_PLIC_CONTEXT_BASE,
                                 G233_PLIC_CONTEXT_STRIDE,
                                 memmap[G233_DEV_PLIC].size);
    riscv_aclint_swi_create(memmap[G233_DEV_CLINT].base,
                            0, ms->smp.cpus, false);
	[...]
}
```

`instance_init` 接口通常只用于完成在设备实例化前的必要工作，比如这里通过调用 `object_initialize_child` 初始化其内嵌的设备对象，并建立子对象与父对象之间的依赖关系。

```c
static void g233_soc_init(Object *obj)
{
    /*
     * You can add more devices here(e.g. cpu, gpio)
     * Attention: The cpu resetvec is 0x1004
     */
    //+ G233SoCState *s = RISCV_G233_SOC(obj);

    //+ object_initialize_child(obj, "cpus", &s->cpus, TYPE_RISCV_HART_ARRAY);
    //+ object_initialize_child(obj, "gpio", &s->gpio, TYPE_SIFIVE_GPIO);
}
```

最后总结，下面是笔者根据个人理解，用 C++ 编写的一个简单的程序，用于表示 QOM 设备模型，只用于辅助理解，语义上并不一定和 QOM 等同。

简单一点来说，上述三个函数接口执行的先后顺序为 `class_init -> instance_init -> realize`。`class_init` 完成类对象共享成员的初始化，由于本设备只涉及成员函数，而不涉及到类似静态成员变量的概念，因此这部分在 C++ 中直接通过虚函数在语言层面便完成了实现。`instance_init` 可以看作是类的构造函数，且该构造函数应该 尽可能精简，精简到只调用其内嵌类型成员的无参构造函数。`realize` 成员函数则用于完成设备的完整实例化，包括动态创建指针类型成员和设置内嵌类型成员的成员变量（属性）。

```c
#include <iostream>

class GpioController {
private:
    std::string name;

public:
    GpioController() {
        std::cout << "GPIO 在地址 " << this << " 处被构造" << std::endl;
    }

    ~GpioController() {
        std::cout << "GPIO 在地址 " << this << " 处被析构" << std::endl;
    }

    void set_name(const std::string &name) {
        this->name = name;
        std::cout << "GPIO 名称被设置为 " << name << std::endl;
    }
};

class CpuCluster {
private:
    int num_cores;

public:
    CpuCluster() {
        std::cout << "CPU 集群在地址 " << this << " 处被构造" << std::endl;
    }

    ~CpuCluster() {
        std::cout << "CPU 集群在地址 " << this << " 处被析构" << std::endl;
    }

    void set_num_cores(int num_cores) {
        this->num_cores = num_cores;
        std::cout << "CPU 核心数被设置为 " << num_cores << std::endl;
    }
};

class SiFivePLIC {
public:
    SiFivePLIC() {
        std::cout << "SiFivePLIC 在地址 " << this << " 处被构造" << std::endl;
    }

    ~SiFivePLIC() {
        std::cout << "SiFivePLIC 在地址 " << this << " 处被析构" << std::endl;
    }
};

class Device {
public:
    Device() {
        std::cout << "Device 在地址 " << this << " 处被构造" << std::endl;
    }

    ~Device() {
        std::cout << "Device 在地址 " << this << " 处被析构" << std::endl;
    }

    virtual void realize() {
        std::cout << "virtual realize" << std::endl;
    }
};

class G233SoC : public Device {
private:
    // 内嵌类型成员
    CpuCluster cpus;
    GpioController gpio;
    // 指针类型成员
    SiFivePLIC *plic;

public:
    // 构造函数相当于 instance_init
    G233SoC() : cpus(), gpio(), plic(nullptr) {
        // instance_init 应尽可能精简，且不允许失败
        std::cout << "G233SoC 构造函数主体" << std::endl;
    }

    ~G233SoC() {
        delete plic;
    }

    // 相当于 class_init，C++ 在语言层面实现了动态绑定
    void realize() override {
        // QOM 设备模型通常在 realize 中进行属性设置等具体的实例化操作
        std::cout << "开始 realize" << std::endl;
        plic = new SiFivePLIC();
        gpio.set_name("gpio0");
        cpus.set_num_cores(1);
        std::cout << "realize 完成" << std::endl;
    }
};

int main() {
    G233SoC my_soc;
    my_soc.realize();
    return 0;
}
```

!!! tip

    有关 `instance_init` 和 `realize` 的区别，以及如何实现它们的 best practice，可以参考文章：[QEMU’s instance_init() vs. realize()][qemu-instance-vs-realize-link]

## lab2 ~ lab5 新增自定义指令

lab2 ~ 5 和 G233 板卡基本没什么关联，主要是理解 QEMU RISC-V 模拟器指令从格式声明到功能定义的全过程。只要完成了一个，后面几个大部分的工作就只是复制粘贴了，下面我们以 lab2 的 dma 指令为例进行介绍。

首先需要声明指令的格式，这部分需要首先理解 QEMU RISC-V 引入的 Decodetree 的使用方式，可以参考在线讲义中的内容：[Decodetree - Learning QEMU Docs][qemu-decodetree-link]。由于我们新增指令为 32 位，因此在 `target/riscv/insn32.decode` 中进行添加。

```c
diff --git a/target/riscv/insn32.decode b/target/riscv/insn32.decode
index cd23b1f..230f6ce 100644
--- a/target/riscv/insn32.decode
+++ b/target/riscv/insn32.decode
@@ -111,6 +111,9 @@
 # Formats 128:
 @sh6       ...... ...... ..... ... ..... ....... &shift shamt=%sh6 %rs1 %rd

+# *** Learning QEMU ***
+dma        0000110  ..... ..... 110 ..... 1111011 @r
+
 # *** Privileged Instructions ***
 ecall       000000000000     00000 000 00000 1110011
 ebreak      000000000001     00000 000 00000 111001
```

接下来，Decodetree 脚本将会根据指令的声明，自动生成对应的译码代码和相应的指令功能函数 `trans_<insn_name>` 的声明，同时在译码完成后对其进行调用，本指令名为“dma”，因此它会生成 `trans_dma` 的声明，我们需要将其实现。

!!! tip

    若想要观察 Decodetree 生成代码的具体内容，可以查看 `build/libqemu-riscv64-softmmu.a.p/decode-insn32.c.inc`。

```diff
diff --git a/target/riscv/insn_trans/trans_rvi.c.inc b/target/riscv/insn_trans/trans_rvi.c.inc
index b9c7160..7283e02 100644
--- a/target/riscv/insn_trans/trans_rvi.c.inc
+++ b/target/riscv/insn_trans/trans_rvi.c.inc
@@ -18,6 +18,17 @@
  * this program.  If not, see <http://www.gnu.org/licenses/>.
  */

+/* Learning QEMU */
+static bool trans_dma(DisasContext *ctx, arg_dma *a)
+{
+    TCGv dst = get_gpr(ctx, a->rd, EXT_NONE);
+    TCGv src = get_gpr(ctx, a->rs1, EXT_NONE);
+    TCGv grain = get_gpr(ctx, a->rs2, EXT_NONE);
+
+    gen_helper_dma(tcg_env, dst, src, grain);
+    return true;
+}
+
```

> 这里的 trans_rvi.c.inc 是笔者随意选取的位置，并非指令实际的类型。

由于 dma 指令的功能相对复杂，难以直接用 QEMU 的 TCG IR 来实现，因此需要借助于 helper 函数。helper 函数也分为声明和定义两部分，声明的格式可以参考 `target/riscv/helper.h` 中的其他实现。

```diff
diff --git a/target/riscv/helper.h b/target/riscv/helper.h
index f712b1c..0b2294a 100644
--- a/target/riscv/helper.h
+++ b/target/riscv/helper.h
@@ -1,3 +1,6 @@
+/* Learninig QEMU */
+DEF_HELPER_4(dma, void, env, tl, tl, tl)
+
```

最后便是对 helper 函数进行实现，实现部分就很简单了，当然若不想自己手动实现的话，也可以直接参考测试程序的写法。

```diff
diff --git a/target/riscv/op_helper.c b/target/riscv/op_helper.c
index 110292e..b0e2407 100644
--- a/target/riscv/op_helper.c
+++ b/target/riscv/op_helper.c
@@ -28,6 +28,30 @@
 #include "exec/tlb-flags.h"
 #include "trace.h"

+/* Learning QEMU */
+void helper_dma(CPURISCVState *env, uintptr_t dst,
+                uintptr_t src, target_ulong grain)
+{
+    int n;
+    int i, j;
+    float val;
+    uintptr_t src_p, dst_p;
+
+    if (grain > 2) {
+        riscv_raise_exception(env, RISCV_EXCP_ILLEGAL_INST, GETPC());
+    }
+
+    n = 1 << (grain + 3);
+    for (i = 0; i < n; ++i) {
+        for (j = 0; j < n; ++j) {
+            src_p = src + (i * n + j) * sizeof(float);
+            dst_p = dst + (j * n + i) * sizeof(float);
+            val =  make_float32(cpu_ldl_data(env, src_p));
+            cpu_stl_data(env, dst_p, float32_val(val));
+        }
+    }
+}
+
 /* Exceptions processing helpers */
 G_NORETURN void riscv_raise_exception(CPURISCVState *env,
                                       RISCVException exception,
```

事实上，对于完整的指令新增流程而言，还需要在反汇编器 `disas/riscv.c` 中添加对应指令的逻辑，以便于调试。但是测试程序只关注指令的功能，因此笔者在此没有实现。

剩余的几条指令实现方式和 dma 基本一致，在此就不过多赘述了。

## lab6 ~ lab10 扩展 SPI 设备

lab6 ~ lab10 需要新增一个设备的模拟，并将其与 G233 主板连接起来，这也是三部分实验中最有挑战性的一个，这部分建议参考其他 SPI 设备比如 SiFiveSPI 的实现。由于涉及到的代码较多，下面笔者将选取一些关键点以及笔者在做实验的过程中感到有些困惑的地方进行介绍。

### 与 SoC 和从设备的连接

G233 SPI 设备作为一个独立的设备实现，要想起作用，必然要与我们的 G233 SoC 进行连接，具体分为以下几个操作：

- 将 G233 SPI 的设备寄存器映射到系统内存（MMIO）中，通过调用 `sysbus_mmio_map` 函数实现。

- 将 G233 SPI 的中断信号线连接到 G233 SoC 的中断控制器 PLIC，通过调用 `sysbus_connect_irq` 函数实现。

```c
static void g233_soc_realize(DeviceState *dev, Error **errp)
{
	[...]

    /* G233 SPI device */
    sysbus_realize(SYS_BUS_DEVICE(&s->spi), errp);
    sysbus_mmio_map(SYS_BUS_DEVICE(&s->spi), 0, memmap[G233_DEV_SPI].base);
    sysbus_connect_irq(SYS_BUS_DEVICE(&s->spi), 0,
                       qdev_get_gpio_in(DEVICE(s->plic), G233_SPI_IRQ));
}
```

除了与 SoC 相连外，G233 SPI 设备还需要与两个 flash 芯片相连，具体分为以下几个操作：

- 创建题目指定的 flash 设备，并与命令行参数中传入的 flash 文件后端相关联。
- 将 flash 设备 与 SSI 总线相连，用于后续 G233 SPI 向该总线中写入数据与 flash 设备通信。
- 将 G233 SPI 的中断信号线连接到 flash，从而实现片选。

```c
typedef struct G233SPIState {
	[...]

    SSIBus *spi;

    qemu_irq irq;
    qemu_irq *cs_lines;
} G233SPIState;
```

需要注意的是手册中并没有给出运行测试程序时 QEMU 的启动参数，我们可以在执行 make 时启用详细输出（Verbose Mode）来看到：`make check-gevico-tcg V=1`，其参数如下：

```bash
build/qemu-system-riscv64 \
    -M g233 \
    -m 2G \
    -display none \
    -semihosting \
    -serial stdio \
    -d int \
    -device loader,file=test-xxx \
    -blockdev driver=file,filename=disk0.img,node-name=flash0 \
    -blockdev driver=file,filename=disk1.img,node-name=flash1
```

flash 文件后端通过 `node-name` 来标识，我们可以在代码中使用 `blk_by_name` 找到对应的 `BlockBackend` 对象，并通过 `qdev_prop_set_drive_err` 与将其与新建的 flash 设备对象进行关联。具体的代码如下所示：

```c
static void g233_machine_init(MachineState *machine)
{
	[...]
    BlockBackend *blk0, *blk1;
    DeviceState *flash_dev1, *flash_dev2;
    qemu_irq flash_cs1, flash_cs2;

	[...]

    /* Connect first flash to SPI */
    flash_dev1 = qdev_new("w25x16");
    qdev_prop_set_uint8(flash_dev1, "cs", 0);
    blk0 = blk_by_name("flash0");
    qdev_prop_set_drive_err(flash_dev1, "drive", blk0, &error_fatal);
    qdev_realize_and_unref(flash_dev1, BUS(s->soc.spi.spi), &error_fatal);
    flash_cs1 = qdev_get_gpio_in_named(flash_dev1, SSI_GPIO_CS, 0);
    sysbus_connect_irq(SYS_BUS_DEVICE(&s->soc.spi), 1, flash_cs1);

    /* Connect second flash to SPI */
    flash_dev2 = qdev_new("w25x32");
    qdev_prop_set_uint8(flash_dev2, "cs", 1);
    blk1 = blk_by_name("flash1");
    qdev_prop_set_drive_err(flash_dev2, "drive", blk1, &error_fatal);
    qdev_realize_and_unref(flash_dev2, BUS(s->soc.spi.spi), &error_fatal);
    flash_cs2 = qdev_get_gpio_in_named(flash_dev2, SSI_GPIO_CS, 0);
    sysbus_connect_irq(SYS_BUS_DEVICE(&s->soc.spi), 2, flash_cs2);
}
```

最终在创建完成后，可以通过 QEMU monitor 进行查看：

```bash
QEMU 10.0.93 monitor - type 'help' for more information
(qemu) info qtree
bus: main-system-bus
  type System
  dev: g233.spi, id ""
    gpio-out "sysbus-irq" 5
    mmio 0000000010018000/0000000000001000
    bus: spi
      type SSI
      dev: w25x32, id ""
        gpio-in "WP#" 1
        gpio-in "ssi-gpio-cs" 1
        write-enable = false
        nonvolatile-cfg = 36863 (0x8fff)
        spansion-cr1nv = 0 (0x0)
        spansion-cr2nv = 8 (0x8)
        spansion-cr3nv = 2 (0x2)
        spansion-cr4nv = 16 (0x10)
        drive = ""
        cs = 1 (0x1)
      dev: w25x16, id ""
        gpio-in "WP#" 1
        gpio-in "ssi-gpio-cs" 1
        write-enable = false
        nonvolatile-cfg = 36863 (0x8fff)
        spansion-cr1nv = 0 (0x0)
        spansion-cr2nv = 8 (0x8)
        spansion-cr3nv = 2 (0x2)
        spansion-cr4nv = 16 (0x10)
        drive = ""
        cs = 0 (0x0)
[...]
```

### 全双工同步数据传输的实现

数据传输部分的实现，`sifive_spi.c` 和 `xilinx_spi.c` 等都纷纷采用了 QEMU 实现的一种数据结构 `Fifo8`，顾名思义，它是一个先进先出的缓冲区，很适合用来实现 SPI 这样的全双工串行数据通信。但是 G233 SPI 的 FIFO 容量只有 1（8-bit），因此直接用一个整型变量来表示更加简洁，本文接下来也直接采用这种方式。

要实现 CPU 对 SPI 设备寄存器的访问，需要创建一个 SPI 设备的 `MemoryRegion`，并且实现该 MR 的 `read` 和 `write` 接口，根据传入的偏移量 `offset` 决定对哪个设备寄存器进行读写，最后将 MR 添加到系统内存中，实现 MMIO。MMIO 上一节已经介绍过，下面主要介绍如何实现 `read` 和 `write` 接口。

```c
typedef struct G233SPIState {
    /*< private >*/
    SysBusDevice parent_obj;

    /*< public >*/
    MemoryRegion iomem;

    uint32_t cr1;
    uint32_t cr2;
    uint32_t sr;
    uint32_t dr;
    uint32_t csctrl;

	[...]
} G233SPIState;
```

`read` 接口的实现基本没有什么需要注意的地方，主要是 DR 寄存器在读取数据后，需要设置 SR 寄存器，表示此时读缓冲区清空。

!!! tip

    这里笔者没有考虑读之前读缓冲区已为空的情况，而是认为读之前检查 SR 状态是设备驱动程序（即测试程序）考虑的事情。

```c
static uint64_t g233_spi_read(void *opaque, hwaddr offset,
                              unsigned int size)
{
    G233SPIState *s = opaque;
    uint32_t val = 0;

    switch (offset) {
	[...]
    case G233_SPI_DR:
        val = s->dr;
        s->sr &= ~SPI_SR_RXNE;
        break;

	[...]

    return val;
}
```

`write` 接口相对来说要复杂一些。CR1 和 CR2 寄存器的写没有什么特别的地方，直接写入即可。

DR 寄存器在写入之前，需要判断缓冲区是否已满，如果是则需要设置 SR 的溢出位表示触发溢出错误；否则将数据写入到 DR 寄存器中，并设置 SR 寄存器，表示此时写缓冲区已满。在此之后，SPI 控制器的作用便体现出来了：它将写缓冲区中的数据通过 SSI 总线送到片选打开的从设备中，再将接收到的返回数据存入读缓冲区中。这部分调用的是 `ssi_transfer` 函数，参数为 SSI 总线和发送的数据，返回值为返回的数据。由于缓冲区的大小只有 1，因此我们不需要循环，直接单次调用即可，在将返回的数据存入 DR 寄存器前，同样需要进行溢出检查，如果此时读缓冲区已满，那么设置 SR 的溢出位表示触发溢出错误。此外，需要在整个传输过程的前后设置和清除 BSY 位。

> 关于 BSY 位设置的时机笔者也不是很清楚。

最后，对 CSCTRL 寄存器的写入需要更新片选信号，这部分我们下一节再详细介绍。

```c
static void g233_transfer_data(G233SPIState *s)
{
    uint32_t retval;

    s->sr |= SPI_SR_BSY;

    retval = ssi_transfer(s->spi, s->dr);
    if (s->sr & SPI_SR_RXNE) {
        s->sr |= SPI_SR_OVERRUN;
    } else {
        s->dr = retval;
    }
    s->sr |= SPI_SR_TXE;
    s->sr |= SPI_SR_RXNE;

    s->sr &= ~SPI_SR_BSY;
}

static void g233_spi_write(void *opaque, hwaddr offset,
                           uint64_t value, unsigned int size)
{
    G233SPIState *s = opaque;

    switch (offset) {
	[...]
    case G233_SPI_SR:
        if (value & SPI_SR_OVERRUN)
            s->sr &= ~SPI_SR_OVERRUN;
        if (value & SPI_SR_UNDERRUN)
            s->sr &= ~SPI_SR_UNDERRUN;
        break;
    case G233_SPI_DR:
        if (!(s->sr & SPI_SR_TXE)) {
            s->sr |= SPI_SR_OVERRUN;
        } else {
            s->dr = (uint32_t)value;
            s->sr &= ~SPI_SR_TXE;
            g233_transfer_data(s);
        }
        break;
    case G233_SPI_CSCTRL:
        s->csctrl = (uint32_t)value;
        g233_spi_update_cs(s);
        break;
    }

	[...]
}
```

### 中断传递

CPU 与 I/O 设备通信的方式通常有两种：轮询和中断。对本实验的 G233 SPI 设备而言，轮询方式就是 CPU 不断读取 SR 寄存器，确认读缓冲区和写缓冲区的状态，并决定是否从 DR 寄存器中读取数据或向其写入数据。而中断方式则是 CPU 注册相应的中断处理函数，SPI 设备当读缓冲区满或写缓冲区空时向 CPU 发起中断，CPU 自动跳转到对应的中断处理函数执行相应的操作：如从读缓冲区中读取数据，或向写缓冲区中写入数据。此外，某些错误发生时，SPI 设备也要向 CPU 发起中断请求处理，lab10 的溢出检测就是这种情况。

发起中断的函数为 `qemu_set_irq`，我们可以专门编写一个函数，根据此时控制寄存器和状态寄存器的值决定是否要发起中断，代码如下所示，并在对设备寄存器进行读写操作时调用。

!!! tip

    在 `read`、`write` 接口处调用中断更新是因为对 G233 SPI 设备而言，CPU 对设备寄存器的读写是 唯一可能 使得 SPI 设备状态发生变化的情况。

```c
static void g233_spi_update_irq(G233SPIState *s)
{
    int level = 0;

    /* Should trigger TX int ? */
    if ((s->cr2 & SPI_CR2_TXEIE) && (s->sr & SPI_SR_TXE))
        level = 1;

    /* Should trigger RX int ? */
    if ((s->cr2 & SPI_CR2_RXNEIE) && (s->sr & SPI_SR_RXNE))
        level = 1;

    /* Should trigger ERR int ? */
    if ((s->cr2 & SPI_CR2_ERRIE) && (s->sr & (SPI_SR_OVERRUN | SPI_SR_UNDERRUN)))
        level = 1;

    qemu_set_irq(s->irq, level);
}
```

片选操作的实现也是类似，根据 CSCTRL 寄存器的值决定启用哪块 flash 芯片，代码如下所示，这个更新过程可以在 CSCTRL 寄存器被写入时进行，需要注意的是片选信号为 0 代表有效。

```c
#define CSi_EN(sr, i) \
    ( ( (sr) >> (i) ) & 1 )

#define CSi_ACT(sr, i) \
    ( ( (sr) >> ( (i) + 4 ) ) & 1)

static void g233_spi_update_cs(G233SPIState *s)
{
    int i;

    for (i = 0; i < NUM_CS; i++) {
        if (!CSi_EN(s->csctrl, i)) {
            qemu_set_irq(s->cs_lines[i], 1);
            continue;
        }

        if (CSi_ACT(s->csctrl, i)) {
            qemu_set_irq(s->cs_lines[i], 0);
        } else {
            qemu_set_irq(s->cs_lines[i], 1);
        }
    }
}
```

[qemu-camp-2025-note-link]: https://lordaeronesz.top/2025/10/28/QEMU%E8%AE%AD%E7%BB%83%E8%90%A52025%E4%B8%93%E4%B8%9A%E9%98%B6%E6%AE%B5%E5%AE%9E%E9%AA%8C%E8%A7%A3%E6%9E%90/
[qemu-datasheet-link]: https://gevico.github.io/learning-qemu-docs/ch4/g233-board-datasheet/#_3
[qemu-instance-vs-realize-link]: https://people.redhat.com/~thuth/blog/qemu/2018/09/10/instance-init-realize.html
[qemu-decodetree-link]: https://gevico.github.io/learning-qemu-docs/ch2/sec2/qemu-accel-tcg/tcg-add-guest-insn/

