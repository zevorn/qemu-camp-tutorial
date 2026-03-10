本文档是专业阶段的实验手册。

专业阶段的实验围绕着 G233 虚拟开发板展开，你需要按照 [G233 Board Datasheet][5] 手册给出的硬件参数，完成实验任务，从而帮助你更好的掌握 QEMU 建模，理解 QEMU 模拟器的工作原理。

!!! note "温馨提示"

    关于硬件建模部分，你可以尝试用 Rust 来实现（QEMU 有基本框架）；若想进一步挑战自我，也可以尝试用 Rust 模拟客户机指令（需要自己从零实现）。

## 环境搭建

第一步，以 Ubuntu 22.04 为例，介绍如何安装 QEMU 开发环境。

```bash
# 备份 sources.list 文件
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak

# 启用 deb-src 源（将所有 deb 源对应的 deb-src 源解锁）
sudo sed -i '/^# deb-src /s/^# //' /etc/apt/sources.list
sudo apt-get update
sudo apt update && sudo apt build-dep qemu

# 创建工具链安装目录
sudo mkdir -p /opt/riscv

# 下载工具链压缩包
wget https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2025.09.28/riscv64-elf-ubuntu-22.04-gcc-nightly-2025.09.28-nightly.tar.xz -O riscv-toolchain.tar.xz

# 解压到安装目录
sudo tar -xJf riscv-toolchain.tar.xz -C /opt/riscv --strip-components=1

# 设置权限
sudo chown -R $USER:$USER /opt/riscv
echo "/opt/riscv/bin" >> $GITHUB_PATH
export PATH=$PATH:/opt/riscv/bin/

riscv64-unknown-elf-gcc --version  # 验证编译器是否可用

# 安装 Rust 工具链
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable

# 验证 Rust 安装
rustup --version
rustc --version
cargo --version
```

!!! note "提示"

    安装 QEMU 开发环境，请参考导学阶段的 [Step0: 搭建 QEMU 开发环境][1]。

    安装 RISC-V 的交叉编译工具链：[下载地址][2]，尽量选择最新的版本，要求安装 `riscv64-unknown-elf-` 类型。

    安装 Rust，版本要求 >= 1.85，安装方法请参考 [Rust 官方文档][6]。


第二步，点击[这里][3]，自动 fork 作业仓库到 GTOC 组织下面，该仓库会为你开通代码上传权限。

第三步，需要 clone 刚刚 fork 好的仓库到本地：

```bash
git clone git@github.com:gevico/learning-qemu-2026-<你的 github 用户名>.git

# 比如 github 用户名是 zevorn，那么命令如下：
# git clone git@github.com:gevico/learning-qemu-2026-zevorn.git
```

第四步，添加上游远程仓库，用于同步上游的代码变更：

```bash
git remote add upstream git@github.com:gevico/gevico-classroom-learning-qemu-2026-learning-qemu.git
```

同步上游代码变更的常用命令：

```bash
git pull upstream main --rebase
```

最后一步，配置编译选项：

```bash
cd qemu
./configure --target-list=riscv64-softmmu \
            --extra-cflags="-O0 -g3" \
            --cross-prefix-riscv64=riscv64-unknown-elf- \
            --enable-rust
```

执行时，如果看到以下输出，证明交叉编译工具链配置成功：

```bash
  ...
  Cross compilers
    riscv64                         : riscv64-unknown-elf-gcc
  ...
```

## 提交代码

所有实验的测题源码，均放在仓库根目录路径： `tests/gevico/tcg/riscv64/` 。

你需要熟读每个测题源码，理解每个测题的测试意图，并实现对应的 QEMU 建模功能（需要修改 QEMU 本体源码，非测题源码），文末会给出具体实验的介绍，辅助你阅读测题源码。

每次实验完成后，需要将你的代码提交到你的 fork 仓库中。

```bash
git add .
git commit -m "feat: subject..."
git push origin main
```

!!! note

    请确保你的代码符合仓库的代码规范，包括代码格式、注释等。

## 测评验收


本地运行测题的方式：

```bash
make check-gevico-tcg
```

全部测题通过的情况下，你会看到如下输出：

```bash
  BUILD   riscv64-softmmu guest-tests
  RUN     riscv64-softmmu guest-tests
  TEST      1/10   test-board-g233 on riscv64
  TEST      2/10   test-insn-dma on riscv64
  TEST      3/10   test-insn-sort on riscv64
  TEST      4/10   test-insn-crush on riscv64
  TEST      5/10   test-insn-expand on riscv64
  ...
```

如果你想运行某个测例，比如 `test-board-g233`，可以使用如下命令：

```bash
make -C build/tests/gevico/tcg/riscv64-softmmu/  run-board-g233
```

!!! note

    当你使用 `make -C` 指定了路径以后，你可以通过输入 `run-` 和 tab 键来查看可以运行的测题

如果你想调试某个测例，比如 `test-board-g233`，可以使用如下命令启用 QEMU 的远程调试功能：

```bash
make -C build/tests/gevico/tcg/riscv64-softmmu gdbstub-board-g233
```

同理，你也可以通过 `gdbstub-` 和 tab 键来查看可以远程调试的测例。

然后需要你本地另起一个终端，使用 riscv-elf-gdb 加载被调试客户机二进制程序，进行远程调试。

!!! note

    你需要熟读 G233 Board Datasheet 和测题的源码，来理解每个实验的测试意图，这会极大地方便你调试，提高开发效率。

每道测题 10 分，一共 10 道测题，共计 100 分，评分将显示到训练营的[专业阶段排行榜][4]。

## 实验介绍

为了方便设计测题，我们设计了一个虚拟板卡 G233，并且编写了 [G233 Board Datasheet][5]，用于描述 G233 板卡的硬件规格和功能。

该阶段涉及的所有实验的硬件参数，全部记录在 [G233 Board Datasheet][5] 中。熟读手册可以帮你更好的理解每个实验的测试意图。

所有实验的测题，均在 `tests/gevico/tcg/riscv64/` 目录下，以 `test-` 开头的 `.c` 文件。

### 实验一 test-board-g233

源码路径： `tests/gevico/tcg/riscv64/test-board-g233.c`。

该实验用于验证 G233 Board 是否正常工作。你需要在 QEMU 中模拟 G233 Board。

基本代码已经存放在 `hw/riscv/g233.c` 中，需要你进一步补全它。

### 实验二 test-insn-dma

源码路径： `tests/gevico/tcg/riscv64/test-insn-dma.c`。

该实验用于验证 G233 Board 的 DMA 指令功能是否正常工作。你需要在 QEMU 中实现这条指令。

#### 实验目标

本实验测试 G233 Board 的 DMA 指令，该指令用于矩阵转置操作：

- 将源矩阵按指定粒度进行转置
- 支持不同的粒度参数（0, 1, 2）
- 实现高效的矩阵数据重排

#### 主要测试内容

**1. 8x8 矩阵转置测试** (`test_dma_grain_8x8`)

- 测试矩阵：8x8 矩阵，粒度参数为 0
- 输入数据：按行优先顺序填充 0-63
- 验证转置结果的正确性

**2. 16x16 矩阵转置测试** (`test_dma_grain_16x16`)

- 测试矩阵：16x16 矩阵，粒度参数为 1
- 输入数据：按行优先顺序填充 0-255
- 验证转置结果的正确性

**3. 32x32 矩阵转置测试** (`test_dma_grain_32x32`)

- 测试矩阵：32x32 矩阵，粒度参数为 2
- 输入数据：按行优先顺序填充 0-1023
- 验证转置结果的正确性

**4. 指令验证测试** (`custom_dma`)

- 使用内联汇编调用 DMA 指令
- 指令编码：`.insn r 0x7b, 6, 6, %0, %1, %2`
- 参数：目标地址、源地址、粒度大小

**5. 结果比较测试** (`compare`)

- 比较软件转置和硬件 DMA 指令的结果
- 验证 DMA 指令的正确性

这条指令的详细描述在 [G233 Board Datasheet][5] 中。

### 实验三 test-insn-sort

源码路径： `tests/gevico/tcg/riscv64/test-insn-sort.c`。

该实验用于验证 G233 Board 的 sort 指令功能是否正常工作。你需要在 QEMU 中实现这条指令。

#### 实验目标

本实验测试 G233 Board 的 sort 指令，该指令用于数组排序操作：

- 对指定长度的数组进行升序排序
- 支持部分数组排序功能
- 实现高效的硬件排序算法

#### 主要测试内容

**1. 数组排序测试** (`test_sort`)

- 测试数组：32 个元素的整数数组
- 输入数据：`{3, 7, 23, 9, 81, 33, 4, 607747, 13, 2451, 323, 831, 0, ...}`
- 排序长度：前 16 个元素
- 验证排序结果的正确性

**2. 软件排序实现** (`bubble_sort`)

- 使用冒泡排序算法作为参考实现
- 对数组进行升序排序
- 提供排序结果的对比基准

**3. 指令验证测试** (`custom_sort`)

- 使用内联汇编调用 sort 指令
- 指令编码：`.insn r 0x7b, 6, 22, %0, %1, %2`
- 参数：排序长度、数组地址、数组大小

**4. 结果比较测试** (`compare`)

- 比较软件排序和硬件 sort 指令的结果
- 验证 sort 指令的正确性

这条指令的详细描述在 [G233 Board Datasheet][5] 中。

### 实验四 test-insn-crush

源码路径： `tests/gevico/tcg/riscv64/test-insn-crush.c`。

该实验用于验证 G233 Board 的 crush 指令功能是否正常工作。你需要在 QEMU 中实现这条指令。

#### 实验目标

本实验测试 G233 Board 的 crush 指令，该指令用于数据压缩：

- 将 8 位数组元素的低 4 位提取出来
- 两两打包成一个 8 位数据
- 存储到目标数组中

#### 主要测试内容

**1. 数据压缩测试** (`pack_low4bits`)

- 输入数组：`{0xA, 0xB, 0xC, 0xD, 0xE, 0xF, 0x1, 0x2, 0x3, 0x4}`
- 提取低 4 位并打包：`{0xBA, 0xDC, 0xFE, 0x21, 0x43}`

**2. 指令验证测试** (`custom_crush`)

- 使用内联汇编调用 crush 指令
- 指令编码：`.insn r 0x7b, 6, 38, %0, %1, %2`
- 参数：目标地址、源地址、元素数量

**3. 结果比较测试** (`compare`)

- 比较软件实现和硬件指令的结果
- 验证 crush 指令的正确性

这条指令的详细描述在 [G233 Board Datasheet][5] 中。

### 实验五 test-insn-expand

源码路径： `tests/gevico/tcg/riscv64/test-insn-expand.c`。

该实验用于验证 G233 Board 的 expand 指令功能是否正常工作。你需要在 QEMU 中实现这条指令。

#### 实验目标

本实验测试 G233 Board 的 expand 指令，该指令用于数据扩展操作：

- 将 8 位数据的低 4 位和高 4 位分别提取
- 扩展为两个独立的 4 位数据
- 实现数据解压缩功能

#### 主要测试内容

**1. 数据扩展测试** (`split_to_4bits`)

- 输入数组：`{0xAB, 0xBC, 0xCD, 0xDE, 0xEF, 0xFA, 0x13, 0x24, 0x63, 0x74}`
- 扩展结果：每个 8 位数据拆分为两个 4 位数据
- 输出长度：原数组长度的 2 倍

**2. 指令验证测试** (`custom_expand`)

- 使用内联汇编调用 expand 指令
- 指令编码：`.insn r 0x7b, 6, 54, %0, %1, %2`
- 参数：目标地址、源地址、数据数量

**3. 结果比较测试** (`compare`)

- 比较软件实现和硬件 expand 指令的结果
- 验证 expand 指令的正确性

这条指令的详细描述在 [G233 Board Datasheet][5] 中。

### 实验六 test-spi-jedec

源码路径： `tests/gevico/tcg/riscv64/test-spi-jedec.c`。

该实验用于验证 G233 Board 的 SPI-JEDEC 功能是否正常工作。你需要在 QEMU 中实现这个外设。

#### 实验目标

本实验测试 G233 Board 的 SPI 外设基本功能：

- 验证 SPI 控制器初始化配置
- 测试 SPI 数据传输功能
- 读取 Flash 芯片的 JEDEC ID
- 验证 SPI 片选控制功能

#### 主要测试内容

**1. SPI 初始化测试** (`spi_init`)

- 配置 SPI 为主模式
- 设置波特率控制位
- 启用 SPI 功能
- 验证寄存器配置正确性

**2. JEDEC ID 读取测试** (`test_jedec_id`)

- 发送 JEDEC ID 命令 `0x9F`
- 读取 3 字节 JEDEC ID
- 验证 W25X16 Flash 返回 `0xEF 0x30 0x15`
- 测试片选控制时序

**3. SPI 传输功能测试** (`spi_transfer_byte`)

- 测试单字节发送和接收
- 验证 TXE 和 RXNE 状态位
- 测试 SPI 忙状态检测

这个外设的详细描述在 [G233 Board Datasheet][5] 中。

### 实验七 test-flash-read

源码路径： `tests/gevico/tcg/riscv64/test-flash-read.c`。

该实验用于验证 G233 Board 的 flash-read 功能是否正常工作。你需要在 QEMU 中实现这个外设。

#### 实验目标

本实验测试 G233 Board 的 SPI Flash 完整读写功能：

- 验证 Flash 状态寄存器读取
- 测试 Flash 扇区擦除功能
- 验证 Flash 页编程功能
- 测试 Flash 数据读取功能

#### 主要测试内容

**1. Flash 状态管理测试** (`flash_read_status`, `flash_wait_busy`)

- 读取 Flash 状态寄存器
- 检测 Flash 忙状态
- 等待操作完成

**2. Flash 擦除测试** (`flash_sector_erase`)

- 发送扇区擦除命令 `0x20`
- 擦除 4KB 扇区
- 验证擦除操作完成

**3. Flash 编程测试** (`flash_page_program`)

- 发送页编程命令 `0x02`
- 编程 256 字节数据
- 验证编程操作完成

**4. Flash 读取测试** (`flash_read_data`)

- 发送读取命令 `0x03`
- 读取 256 字节数据
- 验证数据完整性

**5. 完整读写测试** (`flash_write_test_data`)

- 生成测试数据（ASCII 字母循环）
- 执行完整的擦除 - 编程 - 读取流程
- 比较写入和读取的数据
- 验证所有 256 字节数据匹配

这个外设的详细描述在 [G233 Board Datasheet][5] 中。

### 实验八 test-flash-read-int

源码路径： `tests/gevico/tcg/riscv64/test-flash-read-interrupt.c`。

该实验用于验证 G233 Board 的中断功能是否正常工作。你需要在 QEMU 中实现这个外设的中断功能。

#### 实验目标

本实验测试 G233 Board 的 SPI 中断驱动功能：
- 验证 SPI 中断处理机制
- 测试中断驱动的数据传输
- 验证中断状态管理
- 测试完整的 Flash 中断读写功能

#### 主要测试内容

**1. SPI 中断处理测试** (`spi0_interrupt_handler`)

- 处理 TXE（发送缓冲区空）中断
- 处理 RXNE（接收缓冲区非空）中断
- 处理错误中断（UDR、OVR）
- 管理中断状态和计数

**2. 中断驱动传输测试** (`g233_spi_transfer_interrupt`)

- 使用中断方式传输数据
- 管理发送和接收缓冲区
- 处理传输完成状态
- 支持超时检测

**3. Flash 中断操作测试**

- 中断驱动的状态读取
- 中断驱动的 JEDEC ID 读取
- 中断驱动的扇区擦除
- 中断驱动的页编程
- 中断驱动的数据读取

**4. 完整中断测试** (`flash_write_test_data`)

- 使用中断方式执行完整 Flash 操作
- 验证中断驱动的数据完整性
- 测试中断计数和状态管理
- 比较中断方式和轮询方式的结果

这个中断对应的外设的详细描述在 [G233 Board Datasheet][5] 中。

### 实验九 test-spi-cs

源码路径： `tests/gevico/tcg/riscv64/test-spi-cs.c`。

该实验用于验证 G233 Board 的双 SPI Flash 片选功能是否正常工作。你需要在 QEMU 中实现 SPI 外设的片选控制功能。

#### 实验目标

本实验测试 G233 Board 上连接的两个不同规格的 SPI Flash 芯片：

- **Flash 0 (CS0)**: W25X16 (2MB)
- **Flash 1 (CS1)**: W25X32 (4MB)

#### 主要测试内容

**1. Flash 识别测试** (`test_flash_identification`)

- 读取两个 Flash 芯片的 JEDEC ID
- 验证 Flash 0 返回 `0xEF3015` (W25X16)
- 验证 Flash 1 返回 `0xEF3016` (W25X32)

**2. 独立 Flash 操作测试** (`test_individual_flash_operations`)

- 分别对两个 Flash 进行擦除、编程、读取操作
- Flash 0 写入 A-Z 字母模式
- Flash 1 写入 a-z 字母模式
- 验证数据完整性

**3. 交叉 Flash 操作测试** (`test_cross_flash_operations`)

在不同地址写入不同数据模式：

- Flash 0 @ 0x1000: 模式 A (0xAA)
- Flash 1 @ 0x1000: 模式 B (0x55)
- Flash 0 @ 0x2000: 模式 C (0x33)
- Flash 1 @ 0x2000: 模式 D (0xCC)

验证交叉操作不影响数据完整性

**4. 交替操作测试** (`test_alternating_operations`)

- 在两个 Flash 之间进行交替读取操作
- 验证片选切换不会影响数据

**5. 容量测试** (`test_flash_capacity`)

测试不同地址范围的数据操作：

- Flash 0: 测试 0x000000 和 0x1F0000 (2MB 范围)
- Flash 1: 测试 0x000000, 0x100000 和 0x3F0000 (4MB 范围)

**6. 并发状态检查** (`test_concurrent_status_check`)

- 同时检查两个 Flash 的状态寄存器
- 验证片选控制不影响状态读取

这个外设的详细描述在 [G233 Board Datasheet][5] 中。

### 实验十 test-spi-overrun

源码路径： `tests/gevico/tcg/riscv64/test-spi-overrun.c`。

该实验用于验证 G233 Board 的 SPI 溢出错误检测功能是否正常工作。你需要在 QEMU 中实现 SPI 外设的错误检测和中断功能。

#### 实验目标

本实验测试 SPI 外设在接收缓冲区未清空时继续接收数据时的溢出检测机制，包括：

- 中断模式下的溢出检测
- 轮询模式下的溢出检测
- 溢出标志的清除机制

#### 主要测试内容

**1. 中断模式溢出检测** (`test_interrupt_overrun_detection`)

启用 SPI 错误中断 (`SPI_CR2_ERRIE`)

- 发送第一个字节但不读取（保持 RXNE 标志）

- 发送第二个字节触发溢出

- 验证中断处理函数正确检测到溢出

**2. 轮询模式溢出检测** (`test_polling_overrun_detection`)

- 禁用中断，使用轮询方式

- 重复上述溢出触发过程

- 通过状态寄存器轮询检测溢出标志

- 验证溢出标志的清除机制

#### 溢出检测原理

SPI 溢出发生在以下情况：

- 接收缓冲区有数据未读取（RXNE = 1）
- 继续发送新数据
- 新数据到达但无法存储到已满的接收缓冲区
- 触发溢出错误标志（OVERRUN = 1）

这个外设的详细描述在 [G233 Board Datasheet][5] 中。

[1]: https://qemu.readthedocs.io/en/v10.0.3/devel/build-environment.html
[2]: https://github.com/riscv-collab/riscv-gnu-toolchain/releases/
[3]: https://classroom.github.com/a/HXuCy8g7
[4]: https://opencamp.cn/qemu/camp/2025/stage/3?tab=rank
[5]: https://qemu.gevico.online/exercise/2026/stage1/cpu/cpu-datasheet/
[6]: https://rust-lang.org/zh-CN/tools/install/
