# W4D4: Ethers.js 与程序化交互

## 概述

我们昨天学完了 `cast` 命令行工具，今天我们进入程序化交互的世界。这不仅是学会使用一个库，更重要是理解脚本如何模拟一个完整的客户端行为的。我们先把 API 的外壳拨开，看 Ethers.js / Viem 到底帮我们在底层干了什么。

---

## 为什么需要 Ethers.js 或 Viem？

在昨天，使用了 `cast call` 和 `cast send`。其实这都是这两个命令在底层干的事情，和 Ethers.js/Viem 是一模一样的：

1. **构造参数**：把函数名和参数编码为 ABI 格式（在 W3D7 中手动解码过）
2. **网络通信**：通过 HTTP/WebSocket 向 RPC 节点发送 JSON-RPC 请求
3. **签名管理**：管理私钥，对交易哈希进行 ECDSA 签名

Ethers.js 和 Viem 就是把这些步骤封装成了方便的 JavaScript/TypeScript 函数。

### 库的对比

- **Ethers.js (v5/v6)**：老牌劲旅，采用 **面向对象 (OOP)** 设计。一切皆对象（Provider 对象、Wallet 对象、Contract 对象）。
- **Viem**：新起之秀，采用 **函数式编程** 设计。更轻量，Tree-shaking 更好，专为现代前端（如 React hooks）优化。

---

## 核心概念

无论哪个库都逃不开这两个概念：

### 1. Provider / Public Client

- **角色**：它就是你的眼睛和耳朵
- **能力**：它只负责链接区块链节点。它可以读取数据（余额，区块高度）。但它没有私钥，所以绝对无法发送交易。
- **底层对应**：JSON-RPC 的 `eth_getBalance`, `eth_call`, `eth_blockNumber` 等只读方法。

### 2. Signer / Wallet Client

- **角色**：它是你的手
- **能力**：它持有私钥，核心作用只有一个：对数据进行数字签名
- **底层对应**：它在本地进行椭圆曲线加密，生成 (r, s, v)，然后把签名后的数据交给 Provider 去广播

---

## 第一步：建立联系

### 配置 Provider

我们第一件事就是配置那个负责听和看的 Provider。假设我们要链接 Sepolia 测试网，我们使用 Ethers.js（V6），使用一个特定的类来通过 HTTP URL 来创建这个链接：`JsonRpcProvider`

它就如名字所言，是通过 JSON-RPC 协议来提供服务的。

```javascript
import { ethers } from "ethers";

// 这里的 URL 就是你在 Infura 或 Alchemy 申请的地址
const RPC_URL = "https://sepolia.infura.io/v3/你的API_KEY";

// 初始化 provider
const provider = new ethers.JsonRpcProvider(RPC_URL);
```

### 配置 Signer

接下来，我们要给这个脚本装上手，也就是配置 Signer。在 Ethers.js 中，最常用的 Signer 类叫做 `Wallet`。它需要两个参数来初始化：

1. **私钥（Private Key）**：用来证明身份和签名
2. **Provider**：可选项，如果你想让这个钱包能直接发送交易，最好把刚才创建的 provider 传进去，这样它签完名就能通过 Provider 广播出去。

> ⚠️ **警示**：永远不要在代码里明文写真实的私钥，但是在联系的时候，我们可以用假的私钥或者环境变量来模拟。

```javascript
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
```

---

## 第二步：构造和发送交易

有了人（wallet），下一步就是办事（Transaction）。

在以太坊中，发送 ETH 其实就是一个包含 `to`（接收方）和 `value`（金额）的对象。

### 单位转换

我们在发送的时候要注意，EVM 底层只听得懂 Wei（最小单位），要转换。假设我们给 V 神的地址（`0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`）转 0.001 ETH。

在 Ethers.js 中，我们需要一个工具函数把字符串 `0.001` 转换为巨大的 BigInt 类型的 Wei。我们使用 `ethers.parseEther` 来转换：

```javascript
ethers.parseEther("0.001")
```

### 发送交易

想象你在填一张快递单，你不能把收件人和物品分开给快递员，你需要把他们协助一张单子上。格式如下：

```javascript
wallet.sendTransaction({
  to: "地址",
  value: 转换后的金额
})
```

填入我们需要的数据：

```javascript
const tx = await wallet.sendTransaction({
  to: "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  value: ethers.parseEther("0.001")
});
```

完美！成功的构造并广播了一条交易。

---

## 关键概念：发送 ≠ 确认

这是一个新手最容易踩得坑。上面代码执行完毕之后，变量 `tx` 只是一个 **交易回执（Transaction Response）**。这时候，你的交易只是进入了 Mempool（内存池），还没被打包进入区块。

### 等待确认

我们要是想要脚本实现转账成功检测，那我们必须让代码暂停，直到这笔交易被链上确认。我们使用 `.wait()` 来实现：

```javascript
// 等待交易上链（默认等待 1 个区块确认）
const receipt = await tx.wait();
```

### Transaction Response vs Transaction Receipt

这是一个非常重要的概念区分：

#### Transaction Response（tx）

- 这是你调用 `sendTransaction` 后立刻拿到的。
- 它包含的内容是你填写的信息（To, Value, Data, GasLimit）以及你签名的 Hash。
- **关键点**：拿到它只代表"发送成功"，不代表"执行成功"。交易可能会因为 Gas 不足或逻辑错误在链上失败。

#### Transaction Receipt (receipt)

- 这是 `tx.wait()` 返回的对象。
- 它包含的是链上实际发生的信息：交易在哪一个区块 (`blockNumber`)？用了多少 Gas (`gasUsed`)？最终状态是成功还是失败 (`status: 1` 为成功，`0` 为失败)？
- **关键点**：只有拿到 Receipt，你才能确信这笔交易尘埃落定了。

---

## 第三步：验证结果

现在区块已经打包了，我们的脚本还需要做最后一步：核实。

我们使用 Provider 来获取 V 神的最新余额，使用的方法是 `getBalance`：

```javascript
const balance = await provider.getBalance("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045");
```

到这还没完，你现在直接 `console.log(balance)`，控制台会给你一个巨大的，带 `n` 后缀的数字，比如 `1502000000000000000000n`。

我们在 W3D3 和 W4D3 反复遇到的单位问题，我们需要把它变为人类可读的数字。使用 `ethers.formatEther`，可以把 Wei 变成字符串类型的 ETH：

```javascript
console.log(ethers.formatEther(balance))
```

---

## 完整示例

到现在为止，我们已经把零散的拼图变成了一个完整的程序化转账脚本：

```javascript
import { ethers } from "ethers";

// 1. Setup Provider (眼睛/耳朵)
const provider = new ethers.JsonRpcProvider("https://eth-sepolia.g.alchemy.com/v2/...");

// 2. Setup Signer (手)
const wallet = new ethers.Wallet(process.env.PRIVATE_KEY, provider);

// 3. Send Transaction (动作)
console.log("正在发送交易...");
const tx = await wallet.sendTransaction({
  to: "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  value: ethers.parseEther("0.001")
});

// 4. Wait for Confirmation (等待)
console.log(`交易已发送，哈希: ${tx.hash}`);
await tx.wait();

// 5. Verify (核实)
const balance = await provider.getBalance("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045");
console.log(`Vitalik 的最新余额: ${ethers.formatEther(balance)} ETH`);
```

---

## 深入理解：Nonce 的自动填充

我们在讲构造和发送交易的时候，只填了 `to` 和 `value` 字段。但在以太坊协议中，一笔合法的交易必须包含一个叫 **Nonce** 的字段来防止重放攻击 (Replay Attack)。

既然我们在代码没有指定 nonce，你觉得 Ethers.js 是通过什么方式自动填上的？

正是通过 Provider 自动获取的。我们将 provider 传给了 Wallet，wallet 才能在后台默默地做这件事：它会向链上节点发送一个查询请求（`provider.getTransactionCount`），问："嘿，这个地址目前发起了多少笔交易？" 得到的数字（比如 5）就会自动作为这笔新交易的 Nonce（即 6）。
