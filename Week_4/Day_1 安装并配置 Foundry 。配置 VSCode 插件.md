# Week 4：Foundry 工具链入门

欢迎来到 Week 4：工具链 (Toolchain)。

前三周我们像"修表匠"一样，用镊子和放大镜研究了区块链的每一个零件（哈希、签名、EVM 字节码）。从今天开始，我们要升级装备，换上重型工业机械。我们要学习目前以太坊开发领域最硬核、最流行的开发框架——**Foundry**。

---

## Day 1：安装并配置 Foundry

### 1. 为什么要学 Foundry？

在过去，开发者主要用 Hardhat（基于 JavaScript）。但这两年，Foundry（基于 Rust）几乎统治了高阶开发圈。

- **速度**：它是用 Rust 写的，编译和测试速度比 Hardhat 快几十倍。
- **原生**：你可以直接用 Solidity 写测试脚本。这非常重要！你不需要在 JavaScript 和 Solidity 之间切来切去，这能极大提升你的思维连贯性。
- **内置工具**：还记得我们昨天辛苦手算的解码吗？Foundry 里有个工具叫 `cast`，一行命令就能搞定。

---

## 2. Foundry 三剑客

Foundry 不是一个单一的软件，它是一个工具箱，里面有三把"神兵利器"：

### Forge (锻造)

- **角色**：工厂流水线。
- **用途**：用来写代码、编译合约、运行测试。这是你以后每天用得最多的命令。

### Cast (投射)

- **角色**：瑞士军刀 / 遥控器。
- **用途**：用来和区块链交互。比如查询余额、发送交易、解码数据（W3D7 的手动解码，用 cast 只要 1 秒）。

### Anvil (铁砧)

- **角色**：私人沙盒。
- **用途**：在你的电脑本地启动一个假的区块链节点。你可以在上面随便发交易，不需要花一分钱 Gas，速度极快，重启后数据归零。

---

## 3. 实战操作：安装 Foundry

请根据你的电脑系统，执行下面的操作。

### ⚠️ 准备工作

- 你需要一个终端（Terminal）。
- **Windows 用户注意**：Foundry 不直接支持传统的 CMD 或 PowerShell。你需要安装 WSL (Windows Subsystem for Linux)。
- **Mac / Linux 用户**：直接打开终端即可。

### 第一步：下载安装脚本

在终端里复制粘贴这行命令，然后按回车：

```bash
curl -L https://foundry.paradigm.xyz | bash
```

这行命令会从 Foundry 官网下载一个安装引导程序。

### 第二步：更新环境变量

命令跑完后，它通常会提示你运行一行类似 `source ...` 的命令，或者让你重启终端。为了保险起见，请关闭当前的终端窗口，重新打开一个新的。

### 第三步：安装核心组件

在新的终端里，输入这行命令：

```bash
foundryup
```

这行命令会自动下载 Forge、Cast、Anvil 的最新二进制文件。根据网速，可能需要几十秒。

### 第四步：验证安装

输入下面的命令，检查是否安装成功：

```bash
forge --version
```

如果看到类似 `forge 0.2.0 (....)` 的输出，恭喜你，安装成功！

---

## 4. 配置 VSCode (代码编辑器)

光有命令行还不够，我们需要一个舒服的写代码环境。

### 基础配置

1. 打开 Visual Studio Code。
2. 点击左侧的"扩展 (Extensions)"图标（或者按 `Ctrl+Shift+X` / `Cmd+Shift+X`）。
3. 搜索关键词：**Solidity**。
4. 推荐安装 Nomic Foundation 开发的 Solidity 插件（通常在最上面，logo 是以太坊标志）。

它可以帮你高亮代码颜色、检查语法错误。

---

## 5. Windows 用户特殊配置：WSL 集成

VS Code 会像一座桥梁，把 Windows 和 WSL 无缝连接起来。你几乎感觉不到你是在连接 Linux，体验和本地一模一样。

请按照下面的步骤操作，只需 1 分钟就能配好：

### 第一步：安装"桥梁"插件

1. 打开 Windows 上的 VS Code。
2. 点击左侧的**扩展 (Extensions)** 图标。
3. 搜索关键词：**WSL**。
4. 找到微软官方出品的 WSL 插件（蓝色企鹅图标），点击 **Install**。

### 第二步：如何连接？

1. 打开 VS Code。
2. 点击左下角的绿色小方块图标（或者是 `><` 形状的）。
3. 在弹出的菜单里选择 **"Connect to WSL"**。
4. 它会打开一个新的 VS Code 窗口，连接到你的 WSL 环境。

### 第三步：验证环境

在连接了 WSL 的 VS Code 窗口里：

1. 按下 `Ctrl + ~` (波浪号) 打开内置终端。
2. 这个终端显示的应该是类似 `username@PC-Name:~$` 的 Linux 提示符，而不是 Windows 的 `C:\Users...`。
3. 在这里输入：

```bash
forge --version
```

如果能看到版本号，说明你的 VS Code 已经成功调用了 WSL 里的 Foundry！

---

## 6. 给新手的两个重要提示

### 文件放哪里？

请务必把你的代码文件放在 WSL 的文件系统里（比如 `~/projects/`），而不要放在 Windows 的 C 盘/D 盘，然后通过 `/mnt/c/` 去访问。

**原因**：跨系统文件读写非常慢。放在 WSL 内部，编译速度会快 10 倍以上。

### 插件需要"重新"安装

当你第一次连入 WSL 时，VS Code 会发现这是一个新环境。

你之前安装的 Solidity 插件可能会变灰，旁边有个按钮叫 **"Install in WSL: Ubuntu"**。

点击它。这意味着把插件安装到 Linux 端，这样它才能正确分析 Linux 里的代码。

---

## 总结

### 核心要点

1. **Foundry 是什么**：一个基于 Rust 的以太坊开发框架，包含 Forge（编译/测试）、Cast（交互）、Anvil（本地节点）三个工具。

2. **为什么选择 Foundry**：
   - 速度快（Rust 编写）
   - 原生 Solidity 测试
   - 内置强大的工具链

3. **安装步骤**：
   - 下载安装脚本：`curl -L https://foundry.paradigm.xyz | bash`
   - 运行 `foundryup` 安装核心组件
   - 验证：`forge --version`

4. **编辑器配置**：
   - 安装 Solidity 插件
   - Windows 用户需要配置 WSL 集成
   - 确保文件放在 WSL 文件系统内

5. **关键提示**：
   - 代码文件放在 WSL 内部（`~/projects/`）以获得最佳性能
   - 连接 WSL 后需要重新安装插件
