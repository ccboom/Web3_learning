# W4D6: 合约部署到 Sepolia 测试网

## 概述

今天来把昨天写的合约部署到测试网 Sepolia 上面，如果后面有新的测试网，都是一样的。我们先不要实操，先来看看部署的底层逻辑。

---

## 部署的底层逻辑：火箭与卫星

当我们说部署合约的时候，往往觉得就是把代码上传到了区块链上。**错！** 实际上，EVM 处理这个事的方式非常独特。

当你发送部署交易时，你发送的那个长长的 16 进制代码（Bytecode），其实是一个 **合二为一的包裹**：

### 1. 助推器：Creation / Init Code

- 这部分代码只运行一次
- 它包含了你的 `constructor` 函数
- 它唯一的任务是：初始化变量，然后把"卫星"送入轨道

### 2. 卫星：Runtime Code

- 这才是真正留在链上，以后供大家使用的代码
- 这是助推器运行结束后的 **返回值**

### 类比

这就好比买了一个 **家居组装包**：
- 组装说明和工具（creation code）用来把家具拼好
- 拼好的家具（Runtime Code）是你最终留在客厅的东西
- 一旦家具拼好，那说明书和工具就没什么用了，不会留在客厅

---

## Etherscan 验证失败的两大"深坑"

验证的本质是对比"链上代码"与"你提供的源码编译出的代码"是否原子级一致。

为什么我的合约在 Etherscan 上验证总是失败呢？明明是一样的啊。要解决这个问题，就要看透部署合约时候的 **隐藏尾巴**。

### 陷阱一：构造函数参数（Hidden Tail）

假设你的合约是 `MyBTC("BTC", "Bitcoin")`：
- `MyBTC` 是代码
- `"BTC"`, `"Bitcoin"` 是数据

**在底层发生了什么？**

当你发送部署交易的时候，EVM 实际上把这些参数拼接到了 Creation Code 的屁股后面。

```
部署前：     [助推器代码] + [卫星代码]
实际发送的：  [助推器代码] + [卫星代码] + [参数: "BTC"， "Bitcoin"]
```

**深层原理**：助推器（Constructor）运行时，会通过特定的操作码（CODECOPY）去读取屁股后面这些参数，把他们写进内存或者存储里。

**验证陷阱**：如果你在 Etherscan 上验证时，填写的参数和当时部署的不一样，哪怕错了一个字母，Etherscan 模拟出来的结果就会和链上的不一致，导致失败。

### 陷阱二：元数据哈希（Metadata Hash）

你可能不知道，Solidity 编译器在编译你的合约代码的时候，会自动在生成的字节码最后面，悄悄加一段 **哈希值**。

这段哈希值里包含了你的 **源码指纹**（IPFS Hash）。

**这意味着什么？**

哪怕你在源码里加了一个空格，根据雪崩效应，最后算出来的 hash 值都天差地别。虽然说逻辑没变，可是末尾的元数据 hash 变了，导致 Bytecode 也变了，导致验证失败。

**关键点**：验证合约时，必须保证本地的代码连空格、换行、注释都要和部署的时候一模一样。

---

## Foundry 脚本实操：用 Solidity 部署 Solidity

Foundry 与传统的 JavaScript 脚本不同，Foundry 有一个非常酷的特性：**用 Solidity 来部署脚本**。

这意味着你根本不用切换语言，写合约用 Solidity，部署合约还用 Solidity，太爽了。

### 项目结构回顾

我们原先创建了一个 Foundry 项目，还记得吗？

```
src/      - 存放你的合约源代码（MyToken.sol 就在这里）
test/     - 存放测试文件（W4D2 学过）
script/   - 存放部署脚本（这就是我们要工作的地方）
```

### 创建部署脚本

我们需要在 `script` 这个文件夹里新建一个文件，比如 `DeployToken.s.sol`（`.s.sol` 是 Foundry 脚本的标准后缀）。

### 脚本核心逻辑

部署脚本的核心逻辑非常简单，一共四步：

1. **准备**：引入必要的库和你的合约
2. **开始广播**：告诉 Foundry 接下来的操作都用我的私钥签名
3. **部署**：像 `new` 一个对象一样创建合约
4. **停止广播**：结束签名

### 代码示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Script} from "forge-std/Script.sol";
import {MyToken} from "../src/MyToken.sol";

contract DeployToken is Script {
    function run() external {
        // 1. 获取部署者的私钥 (从环境变量中读取)
        // vm.envUint 是 Foundry 读取 .env 文件的指令
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        // 2. 开始广播交易 (使用该私钥签名)
        vm.startBroadcast(deployerPrivateKey);
        
        // 3. 部署 MyToken 合约
        MyToken token = new MyToken();
        
        // 4. 停止广播
        vm.stopBroadcast();
    }
}
```

### vm.startBroadcast() 的作用

在脚本中，我们需要调用 `vm.startBroadcast()`。它的作用是：**告诉 EVM，接下来所有链上操作，都要使用部署者的私钥生成交易并发出去**。

---

## 命令行执行：上链 Sepolia

脚本写好了，我们离开代码编辑器，在命令行（Terminal）来执行它。

在 Foundry 中，我们要用 `forge script` 命令来运行这个脚本。为了让它真正部署到 Sepolia 测试网，我们需要告诉它两件事：

1. **RPC URL**：去哪里找节点？
2. **Broadcast**：确认要广播这笔交易

### 命令格式

假设你的 `.env` 文件里已经配置好了 `SEPOLIA_RPC_URL`：

```bash
forge script script/DeployToken.s.sol --rpc-url $SEPOLIA_RPC_URL --broadcast
```

### 命令拆解

#### 1. `forge script script/DeployToken.s.sol`

- **意思**：嘿 forge，运行这个脚本文件
- **作用**：它会编译你的合约，并在本地模拟运行一遍 `run()` 函数

#### 2. `--rpc-url $SEPOLIA_RPC_URL`

- **意思**：连上这个地方！
- **作用**：告诉 Forge 别只在本地玩，通过我们 env 中配置的那个链接，和真的 Sepolia 测试网去对话

#### 3. `--broadcast`

- **意思**：广而告之，真的发出去
- **作用**：如果没有这个标记，Forge 只会模拟跑一次；加上它，Forge 才会用你的私钥签名，把交易广播到链上

---

## 总结

### 1. 部署的底层逻辑：火箭与卫星

EVM 部署合约并非简单上传代码，而是发送一段"合二为一"的字节码（Bytecode）：

- **助推器 (Creation Code)**：包含 constructor。只运行一次，负责初始化变量，任务完成后即消失。
- **卫星 (Runtime Code)**：是助推器运行后的返回值。这才是最终留在链上、供长期调用的逻辑代码（类似于拼装完家具后，留下的家具，扔掉说明书）。

### 2. Etherscan 验证失败的两大"深坑"

验证的本质是对比"链上代码"与"你提供的源码编译出的代码"是否原子级一致。

- **陷阱一：构造函数参数（Hidden Tail）**
  - 原理：参数是拼接在 Creation Code 屁股后面的"外挂行李"。
  - 导致失败：验证时填写的参数必须与部署时完全一致，错一个字母都会导致 EVM 模拟结果不同。

- **陷阱二：元数据哈希（Metadata Hash）**
  - 原理：编译器会在字节码末尾加一段包含源码指纹（IPFS Hash）的哈希值。
  - 导致失败：雪崩效应。源码中哪怕改了一个空格、换行或注释，指纹都会变，导致最终 Bytecode 不匹配。

### 3. Foundry 脚本实操：用 Solidity 部署 Solidity

Foundry 的核心优势在于脚本语言也是 Solidity（无需切换 JS/TS）。

- **位置**：脚本文件放在 `script/` 目录下，后缀为 `.s.sol`。
- **代码核心四步**：
  1. 引入：`forge-std/Script.sol` 和目标合约。
  2. 广播开始：`vm.startBroadcast(privateKey)`，告诉 EVM 后续操作需签名。
  3. 部署：`new MyToken()`。
  4. 广播结束：`vm.stopBroadcast()`。

### 4. 命令行执行：上链 Sepolia

使用 `forge script` 命令将脚本转化为实际交易。

**命令格式**：
```bash
forge script script/DeployToken.s.sol --rpc-url $SEPOLIA_RPC_URL --broadcast
```

**参数拆解**：
- `forge script ...`：在本地模拟运行脚本。
- `--rpc-url`：指定连接的节点（通往真实世界的桥梁）。
- `--broadcast`：关键动作。没有它只是模拟；加上它，才会用私钥签名并真正把交易发到链上。
