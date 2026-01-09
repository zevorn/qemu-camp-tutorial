# QEMU 启动参数分析

!!! note "主要贡献者"

    - 作者：[@zevorn](https://github.com/zevorn)

QEMU 是一款功能强大的开源虚拟化和仿真软件，支持多种处理器架构（x86、ARM、RISC-V、MIPS、Loongarch 等）。通过 QEMU，开发者可以在不同的硬件平台上测试操作系统、运行应用程序、进行内核开发和调试。

本章节将聚焦 QEMU 常用启动参数，以启动 OpenEuler RISC-V 系统为实例，重点讲解主板选型、CPU 配置、设备添加等核心操作，进而介绍如何精准查找适配自身需求的 QEMU 启动参数，抛砖引玉助力读者掌握相关应用方法。

了解 QEMU 启动参数，可以帮助我们：

!!! tip

    - 精确控制虚拟机硬件配置：CPU 核心数、内存大小、外设类型
    - 调试和开发需求：GDB 调试、串口输出、内核启动跟踪
    - 性能优化：选择合适的虚拟化加速技术
    - 多架构支持：快速切换不同的硬件平台进行测试

## 常用参数速查

大体上 QEMU 常用的启动参数可以分为五类：主板配置参数、引导参数、存储设备参数、网络参数、显示和交互参数。考虑 QEMU 的启动参数众多，不建议大家死记硬背，而是常用常新，可以将本文作为手册，需要的时候查询一下即可。

下面我们通过表格来介绍，主要罗列适用于 `qemu-system-*` 的启动参数。

**主板配置参数**：

|       parameter       |   example   |     Description    |
|          ---          |     ----    |        ----        |
|  `-machine` or `-M`   | `-M virt`   | 选择机器或者主板的类型， `-M help` 可查询支持的主板 |
|  `-cpu`               | `-cpu rv64` | 选择 CPU 模型， `-cpu help` 可查询支持的 CPU 模型 |
|  `-m`                 | `-m 2G`     | 设置内存大小，支持 M/G 单位，如 2048M |
|  `-smp`               | `-smp 4`    | 配置 CPU 核心数/线程数 |
|  `-device`            | `-device virtio-blk-device`    | 添加新设备，只要当前主板支持 |

**引导参数**：

|       parameter       |          example          |     Description    |
|          ---          |           ----            |        ----        |
|  `-bios`              | `-bios opensbi.bin`       | 加载自定义 BIOS 或 OpenSBI 固件或裸机程序 |
|  `-kernel`            | `-kernel vmlinux`         | 加载 Linux 内核镜像（通常是 vmlinux 或 Image） |
|  `-initrd`            | `-initrd initrd.img`      | 指定初始化内存盘，加载 initramfs 或 initrd 文件系统 |
|  `-append`            | `-append "console=ttyS0"` | 传递给内核的命令行参数，需主板支持 fw_cfg |
|  `-dtb`               | `-dtb kernel.dtb`         | 传递给内核的 DTB 镜像文件 |


**存储设备参数**：

|       parameter       |          example          |     Description    |
|          ---          |           ----            |        ----        |
|  `-drive`             | `-drive file=image.qcow2,format=qcow2,if=virtio` | 添加块设备（硬盘、光盘等）|


**网络参数**：

|   parameter  |          example          |     Description    |
|      ---     |           ----            |        ----        |
| `-netdev`    | `-netdev user,id=net0,hostfwd=tcp::2222-:22` | 定义网络后端，用户模式网络、TAP 设备 |

**显示和交互参数**：

|   parameter  |     example     |     Description    |
|      ---     |      ----       |        ----        |
| `-nographic` |        -        | 禁用图形环境，所有输出重定向到终端，可 `CTRL+A,C` 唤起控制台 |
| `-serial`    | `-serial stdio` | 指定串口输出位置，标准输出是 `stdio`，输出到文件 `file:run.log` |
| `-monitor`   | `-monitor none` | QEMU 内部监控接口，用于运行时管理虚拟机，参数 `none` 合并到串口 |
| `-s`         |        -        | 启用 GDB 服务器（默认端口 1234） |
| `-S`         |        -        | 启动时冻结 CPU |

---

## OpenEuler 启动示例

我们以 OpenEuler RISC-V 24.03 为例进行介绍，获取相关镜像的方式如下：

```bash
wget https://mirror.nyist.edu.cn/openeuler/openEuler-24.03-LTS-SP2/virtual_machine_img/riscv64/RISCV_VIRT_CODE.fd
wget https://mirror.nyist.edu.cn/openeuler/openEuler-24.03-LTS-SP2/virtual_machine_img/riscv64/RISCV_VIRT_VARS.fd
wget https://mirror.nyist.edu.cn/openeuler/openEuler-24.03-LTS-SP2/virtual_machine_img/riscv64/start_vm.sh
wget https://mirror.nyist.edu.cn/openeuler/openEuler-24.03-LTS-SP2/virtual_machine_img/riscv64/openEuler-24.03-LTS-SP2-riscv64.qcow2.xz
```

其中 `start_vm.sh` 里面包含了 QEMU 启动 OpenEuler 的参数，我们摘出来详细介绍一下：

```bash

## Configuration
vcpu=8
memory=8
drive="$(ls *.qcow2)"
fw1="RISCV_VIRT_CODE.fd"
fw2="RISCV_VIRT_VARS.fd"
ssh_port=12055

cmd="qemu-system-riscv64 \
  -nographic -machine virt,pflash0=pflash0,pflash1=pflash1,acpi=off \
  -smp "$vcpu" -m "$memory"G \
  -blockdev node-name=pflash0,driver=file,read-only=on,filename="$fw1" \
  -blockdev node-name=pflash1,driver=file,filename="$fw2" \
  -drive file="$drive",format=qcow2,id=hd0,if=none \
  -object rng-random,filename=/dev/urandom,id=rng0 \
  -device virtio-vga \
  -device virtio-rng-device,rng=rng0 \
  -device virtio-blk-device,drive=hd0 \
  -device virtio-net-device,netdev=usernet \
  -netdev user,id=usernet,hostfwd=tcp::"$ssh_port"-:22 \
  -device qemu-xhci -usb -device usb-kbd -device usb-tablet"
```

我们挑选几个前面没有介绍到的参数，来补充说明一下：

`-machine virt,pflash0=pflash0,pflash1=pflash1,acpi=off`：选择 RISC-V 的 virt 虚拟主板；把名为 pflash0/pflash1 的块设备节点挂到主板的两片 pflash；关闭 ACPI。

`-blockdev node-name=pflash0,driver=file,read-only=on,filename="$fw1"`：创建名为 pflash0 的块设备后端，来源是文件 fw1，并设为只读（固件代码）。

`-blockdev node-name=pflash1,driver=file,filename="$fw2"`：创建名为 pflash1 的块设备后端，来源是文件 fw2（变量区，默认可写）。

`-drive file="$drive",format=qcow2,id=hd0,if=none`：定义一个 qcow2 磁盘为 hd0；if=none 表示只创建驱动器对象，不自动挂到任何总线上（后面再用 -device 手动挂载）。

`-device virtio-blk-device,drive=hd0`：把 hd0 作为 VirtIO 块设备挂到虚拟机上（通常会出现在 guest 里为 /dev/vda 一类）。

`-object rng-random,filename=/dev/urandom,id=rng0`：创建随机数后端 rng0，从主机 /dev/urandom 取随机数据。

`-device virtio-rng-device,rng=rng0`：把随机数后端通过 VirtIO RNG 设备提供给虚拟机。

`-device virtio-net-device,netdev=usernet`：添加一块 VirtIO 网卡，并绑定到 usernet 这个网络后端。

`-device virtio-vga`：添加 VirtIO 显示设备（在 -nographic 下不会弹窗，但仍可能影响固件/显示相关输出路径）。

`-device qemu-xhci`：添加 xHCI 控制器。

`-usb -device usb-kbd`：开启 USB 支持，添加 USB 键盘设备。

`-device usb-tablet`：添加 USB 平板指针设备（绝对坐标，图形界面下鼠标体验更好；-nographic 下通常用处不大）。

现在我们解压镜像，启动 OpenEuler:

```bash
unxz -k openEuler-24.03-LTS-SP2-riscv64.qcow2.xz

chmod +x start_vm.sh
./start_vm.sh
```

!!! tip "如何查阅更多 QEMU 启动参数"

    这里提供两个方法。

    方法一：直接通过 `qemu-system-* --help` 可以查询到所有支持的命令。效果如下:

    ```bash
    $qemu-system-riscv64 --help
    QEMU emulator version 10.2.50 (v10.2.0-325-g582f8d593c)
    Copyright (c) 2003-2025 Fabrice Bellard and the QEMU Project developers
    usage: qemu-system-riscv64 [options] [disk_image]

    'disk_image' is a raw hard disk image for IDE hard disk 0

    Standard options:
    -h or -help     display this help and exit
    -version        display version information and exit
    -machine [type=]name[,prop[=value][,...]]
                    selects emulated machine ('-machine help' for list)
                    property accel=accel1[:accel2[:...]] selects accelerator
                    supported accelerators are kvm, xen, hvf, nvmm, whpx, mshv or tcg (default: tcg)
                    vmport=on|off|auto controls emulation of vmport (default: auto)
                    ...
    -M              as -machine
    -cpu cpu        select CPU ('-cpu help' for list)
    ...
    ```

    方法二：通过训练营提供的 QEMU 知识库，检索需要的启动参数 [ima 知识库: QEMU | 格维开源社区][qemu-ima-link]

    [qemu-ima-link]: https://ima.qq.com/wiki/?shareId=70cb647d4024402dccc94b947c210de2e5c65c68559c166da7ee1a3d9a714e5e

