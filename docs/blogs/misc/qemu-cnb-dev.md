# 基于 CNB 一键启动 QEMU 开发环境

!!! note "主要贡献者"

    - 作者：[@zevorn](https://github.com/zevorn)

## 背景与目标

在本地搭建 QEMU 开发环境时，常见痛点是：依赖链长、工具版本不一致、同一份教程在不同机器上复现困难。

`qemu-lab` 的思路是把“环境准备”前置到云端：你只需要 Fork 仓库并进入 CNB 工作区，就可以直接进入可构建、可调试的开发状态。

- 仓库地址：<https://cnb.cool/gevico.online/qemu-lab>
- 参考 README：<https://cnb.cool/gevico.online/qemu-lab/-/blob/main/README.md>

## 方案核心

根据 README，`qemu-lab` 的设计重点是两件事：

1. 使用 CNB 工作区能力自动完成环境初始化（由 `.cnb.yml` 与 `.ide/` 驱动）。
2. 通过固定源码来源和构建流程，尽量保证构建结果可复现。

这意味着你可以把主要精力放在 QEMU 代码本身，而不是反复处理“环境问题”。

## 一键启动工作流

### 1) Fork 仓库

在 CNB 页面直接 Fork：

```bash
# 浏览器打开
https://cnb.cool/gevico.online/qemu-lab
```

Fork 完成后，你会得到自己的仓库副本。

### 2) 进入云原生开发环境

在你的仓库页面点击“云原生开发”，打开 Code/IDE/Workspace。

首次进入时，平台会按仓库配置自动初始化环境（README 指向 `.cnb.yml` 与 `.ide/`）。

!!! tip

    如果你有团队协作需求，建议统一基于同一份 Fork 模板开展开发，降低“我这里能跑、你那边跑不起来”的概率。

### 3) 获取 QEMU 源码

README 给出的命令如下：

```bash
git clone https://gitlab.com/qemu-project/qemu.git
```

如果你的网络环境对 `gitlab.com` 访问不稳定，可以在团队内部准备镜像源，但需要确保版本策略一致。

### 4) 构建 QEMU

README 示例流程：

```bash
cd qemu
./configure
make -j"$(nproc)"
```

建议在云工作区里优先验证“能编译通过”，再按目标架构增量添加配置参数（例如仅启用需要的 target）。

## 推荐的日常开发节奏

1. 进入 CNB 工作区，等待环境初始化完成。
2. 同步/更新 QEMU 源码。
3. 执行一次完整构建，确认基线可用。
4. 进行小步修改并局部验证。
5. 需要复现问题时，直接分享仓库分支与命令序列。

这种节奏的优势是：环境一致、沟通成本低、复现路径清晰。

## 常见问题与排障建议

### 初始化后命令不可用

先确认工作区初始化是否完成，再检查仓库中的 `.cnb.yml` 与 `.ide/` 是否被误改。

### 构建速度慢

优先检查并行参数是否生效：

```bash
nproc
make -j"$(nproc)"
```

如果工作区规格较小，可先用较低并行度验证功能，再切换更高规格实例做全量构建。
