本文档是专业阶段 Rust 方向的实验手册。

Rust 方向的核心任务：用 Rust 在 QEMU 中实现 I2C 总线、GPIO I2C 控制器和 SPI 控制器，挂载到 G233 SoC 上。评测分两部分——Rust 单元测试验证总线核心逻辑，QTest 验证设备寄存器行为和外设通信。

## 环境搭建

第一步，安装 QEMU 开发依赖和 Rust 工具链。

```bash
# Ubuntu 24.04
sudo sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/ubuntu.sources
sudo apt-get update
sudo apt-get build-dep -y qemu

# 安装 Rust 工具链（版本要求 >= 1.85）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
. "$HOME/.cargo/env"
cargo install bindgen-cli
```

!!! note "提示"

    安装 QEMU 开发环境，请参考导学阶段的 Step0。

    Rust 方向**必须使用 Rust** 实现设备模型，可参考 `rust/hw/char/pl011/` 中的 PL011 Rust 实现。

第二步，点击 [GitHub Classroom 邀请链接][3] 加入实验，系统会自动在组织下为你创建专属仓库并赋予 maintainer 权限。

!!! warning "注意"

    请通过上方链接获取仓库，**不支持手动 fork**。

第三步，clone 仓库到本地：

```bash
git clone git@github.com:gevico/qemu-camp-2026-exper-<你的 github 用户名>.git
```

第四步，添加上游远程仓库，用于同步上游代码变更：

```bash
git remote add upstream git@github.com:gevico/qemu-camp-2026-exper.git
git pull upstream main --rebase
```

第五步，配置并编译：

```bash
make -f Makefile.camp configure
make -f Makefile.camp build
```

## 实验内容

### I2C 总线（Rust 单元测试）

文件 `rust/hw/i2c/src/lib.rs` 中已经定义好 trait 和 struct 的框架，标记了 TODO 的方法需要你来填充：

- `I2CSlave` trait：`address()`、`event()`、`send()`、`recv()` 方法
- `I2CBus` struct：`attach` 挂载设备、`start_transfer` 地址匹配与事件分发、`send`/`recv` 读写转发
- NACK 处理：访问不存在的地址时返回失败

这部分通过 **Rust 单元测试** 验证（3 题）：

| # | 测试名 | 验证内容 |
|---|--------|---------|
| 1 | `test_i2c_bus_create` | 创建总线，挂载设备，验证设备计数 |
| 2 | `test_i2c_bus_read_write` | 挂载 EEPROM 设备，写入后读回验证 |
| 3 | `test_i2c_bus_nack` | 访问不存在的地址，验证返回 NACK |

### GPIO I2C 控制器（QTest）

需要新建一个 Rust SysBus 设备，挂载到 G233 SoC，基地址 `0x10013000`。可参考 `rust/hw/char/pl011/` 中 PL011 的写法。

寄存器映射：

| 偏移 | 名称 | 说明 |
|------|------|-----|
| 0x00 | I2C_CTRL | bit0: EN, bit1: START, bit2: STOP, bit3: RW (0=写，1=读) |
| 0x04 | I2C_STATUS | bit0: BUSY, bit1: ACK, bit2: DONE |
| 0x08 | I2C_ADDR | 7 位从设备地址 |
| 0x0C | I2C_DATA | 数据寄存器 |
| 0x10 | I2C_PRESCALE | 时钟分频器 |

连接的 I2C 设备：AT24C02 EEPROM（地址 `0x50`，容量 256 字节，页大小 8 字节）

这部分通过 **QTest** 验证（4 题）：

| # | 测试名 | 验证内容 |
|---|--------|---------|
| 4 | `test-i2c-gpio-init` | 寄存器复位值，分频器配置 |
| 5 | `test-i2c-gpio-bitbang` | START/STOP/ACK 协议 |
| 6 | `test-i2c-eeprom-rw` | EEPROM 单字节写入和读回 |
| 7 | `test-i2c-eeprom-page` | EEPROM 页写入和边界回绕 |

### Rust SPI 控制器（QTest）

同样新建一个 Rust SysBus 设备，基地址 `0x10019000`。

寄存器映射：

| 偏移 | 名称 | 说明 |
|------|------|-----|
| 0x00 | RSPI_CR1 | bit0: SPE, bit2: MSTR |
| 0x04 | RSPI_SR | bit0: RXNE, bit1: TXE, bit4: OVERRUN |
| 0x08 | RSPI_DR | 数据寄存器 |
| 0x0C | RSPI_CS | 片选 |

连接的 SPI 设备：AT25 EEPROM（256 字节）

这部分通过 **QTest** 验证（3 题）：

| # | 测试名 | 验证内容 |
|---|--------|---------|
| 8 | `test-spi-rust-init` | 寄存器复位值，使能配置 |
| 9 | `test-spi-rust-transfer` | SPI 数据传输，TXE/RXNE 状态 |
| 10 | `test-spi-rust-flash` | AT25 EEPROM 状态寄存器、写入和读回 |

## 提交代码

改完之后提交：

```bash
git add .
git commit -m "feat: subject..."
git push origin main
```

涉及的文件：

- `rust/hw/i2c/src/lib.rs` — I2C 总线核心逻辑（填充 TODO）
- 新建 Rust SysBus 设备文件（GPIO I2C 控制器、SPI 控制器）
- `hw/riscv/g233.c` — 在 SoC 中实例化你的 Rust 设备

!!! note

    核心逻辑必须用 Rust 写，单元测试会检查这一点。

## 测评验收

本地运行全部测题：

```bash
make -f Makefile.camp test-rust
```

共 10 道题（3 道 Rust 单元测试 + 7 道 QTest），每题 10 分，满分 100 分。

每次 push 到 `main` 会触发 CI 评测，得分更新到[排行榜][4]。

[3]: https://classroom.github.com/a/hwWFrmo_
[4]: https://opencamp.cn/qemu/camp/2025/stage/3?tab=rank
