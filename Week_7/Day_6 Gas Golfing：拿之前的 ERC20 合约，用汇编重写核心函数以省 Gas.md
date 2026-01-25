# Day 6: Gas Golfing - 使用汇编重写 ERC20 核心函数以省 Gas

今天我们使用这一周的学习内容，然后对我们之前写的 ERC20 合约进行一点优化。

## 为什么要使用汇编 (Yul)？

在 EVM 中，虽然 Solidity 便于书写，但是使用编译器生成的字节码并不总是最精简的。
使用 Yul，我们可以直接 **绕过 Solidity 的一些安全检查**（比如溢出检查等），并且 **直接操作堆栈、内存和存储**，从而大幅度地降低 Gas 的消耗。这种被称为 "Gas Golfing" 的技术在追求极致效率的 DeFi 协议中非常常见。

---

## 1. 回顾我们写的 ERC-20 合约

在开始优化之前，先看看标准的 Solidity 实现版本：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MyToken {
    // 1. 变量：代币的名字和代号
    string public name = "My First Token";
    string public symbol = "MFT";
    uint8 public decimals = 18;
    uint256 public totalSupply;

    // 2. 账本：余额与授权
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    // 3. 事件：大喇叭
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    // 4. 铸造函数：凭空印钱 (为了测试方便，我们允许任何人调用)
    function mint(uint256 amount) public {
        totalSupply += amount;
        balanceOf[msg.sender] += amount;
        emit Transfer(address(0), msg.sender, amount);
    }

    // 5. 转账函数：我转给你
    function transfer(address to, uint256 amount) public returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Not enough tokens");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    // 6. 授权函数：我允许你动用我的钱
    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    // 7. 代理转账：你动用我的钱转给别人 (DeFi 核心)
    function transferFrom(address sender, address recipient, uint256 amount) public returns (bool) {
        require(allowance[sender][msg.sender] >= amount, "Allowance exceeded");
        
        allowance[sender][msg.sender] -= amount; // 扣减额度
        balanceOf[sender] -= amount;             // 扣减老板余额
        balanceOf[recipient] += amount;          // 增加接收人余额
        
        emit Transfer(sender, recipient, amount);
        return true;
    }
}
```

---

## 2. 核心挑战：手动寻址 (Manual Storage Layout)

我们在重写 ERC20 核心函数之前，面临的第一个问题也是最关键的挑战：**手动寻址**。
Solidity 帮我们自动处理了变量在存储中的位置（Slot），但是在汇编中，我们必须自己算出来。

让我们先定位数据，根据定义的变量顺序：

| 变量名 | 存储位置 (Slot) | 类型 |
| :--- | :--- | :--- |
| `name` | Slot 0 | string |
| `symbol` | Slot 1 | string |
| `decimals` | Slot 2 | uint8 |
| `totalSupply` | Slot 3 | uint256 |
| `balanceOf` | Slot 4 | mapping |
| `allowance`  | Slot 5 | mapping |

### 如何计算 Mapping 的存储位置？

假设我们现在要修改 `msg.sender` 在 `balanceOf` (Slot 4) 中的余额。
你知道 EVM 是通过什么公式或者算法算出这一行数据的具体存储位置吗？
答案是：**使用 Hash 冲突机制来确定数据存储在哪里**。

计算某个 Key 对应的 Value 存储位置的计算公式如下：

$$StorageLocation = keccak256(Key + SlotIndex)$$

在我们的例子中：
- **Key**：就是我们要查询的地址，比如 `msg.sender`。
- **SlotIndex**：就是 Mapping 变量本身的存储位置，`balanceOf` 在 slot 4。

我们要做的是把这两个东西拼接起来做 Hash 运算。
在 Yul 中我们不能直接写 `+` 来拼接，我们需要利用 **Memory (内存)** 作为暂存区。

### 定位步骤拆解

假设我们要计算 `balanceOf[msg.sender]` 的位置，我们需要分两步 `mstore` 操作。
我们需要分别把 `msg.sender` (Key) 和 `4` (SlotIndex) 写入内存的前 32 字节 (0x00) 和后 32 字节 (0x20)。

也就是：
```yul
mstore(0x00, caller()) // 写入 Key (msg.sender)
mstore(0x20, 4)        // 写入 Slot Index
```

这样我们的内存中就有了两个关键的信息：
- `0x00 - 0x20`: msg.sender (补零后)
- `0x20 - 0x40`: 0x00...04

数据都摆好了，我们需要把这整段数据送进哈希函数来算出最终的存储地址。
在 Yul 中，计算哈希的指令是 `keccak256(offset, size)`。

- `offset`: 从内存哪个位置开始读取数据
- `size`: 需要读取数据的总长度是多少字节

代码如下：
```yul
keccak256(0x00, 64)
```
从 `0x00` 位置开始，读取 `64` 字节的数据，就是 **32字节 Key + 32字节 Slot位置**。

我们通常会把它赋值一个变量，比如这样：
```yul
let location := keccak256(0x00, 64)
```

现在我们手里拿到了 `location`，也就是 `balanceOf[msg.sender]` 在以太坊庞大的存储空间中实际居住的地址。

### 读取与写入

既然我们已经找到位置了，我们就要把数据从这个位置中拿出来。
在 Yul 中，从 Storage 读取数据的指令是 `sload(p)`，其中 p 是存储位置。

读取的代码：
```yul
let currentBalance := sload(location)
```
使用这个代码可以从仓库中把值取出来，并放在堆栈的最顶端。

假如我们现在在重写 `mint()` 函数，我们的目标是让这个余额增加 `amount`。
在 Yul 中，加法使用 `add(x,y)`。

```yul
let newBalance := add(currentBalance, amount)
```

我们只差最后一步：写入存储了。
我们使用 `sstore(p, v)` 来把它写入存储：

```yul
sstore(location, newBalance)
```

---

## 3. 实战：重写 mint() 函数

我们回顾一下 `mint` 函数的逻辑：

```solidity
function mint(uint256 amount) public {
    totalSupply += amount;           // <--- 下一步做这个
    balanceOf[msg.sender] += amount; // 上面讲的流程 ✅
    emit Transfer(address(0), msg.sender, amount);
}
```

### 3.1 处理 totalSupply

接下来我们需要处理 `totalSupply += amount;`。
相比于处理 mapping，处理普通变量就要简单的多，因为它不需要计算 hash，位置是固定的。
我们回忆一下，`totalSupply` 在 **Slot 3**。

所以我们的步骤应该是：先读取 Slot 3 中的值，然后加上 amount，最后再放入 Slot 3 中更新。
一行代码搞定：

```yul
sstore(3, add(sload(3), amount))
```

### 3.2 处理 Event (事件)

最后我们就需要写发射事件了。
在 `mint` 函数中 发射事件的定义是这样的： 
`event Transfer(address indexed from, address indexed to, uint256 value);`

在 Yul 中，根本没有 `emit` 这个关键字。我们要使用 `logN` 系列指令，比如 `log1`, `log2`, `log3`, `log4`。
`log` 后面的数字 `N` 代表这个事件有几个 **Topic**。

那我们这个有几个 Topic 呢？
1.  **事件的哈希签名**：本身算作第一个 Topic。
2.  **Indexed 参数**：每一个标记为 `indexed` 的参数也各算作一个 Topic。

那我们就可以弄清楚了：
- **Topic 1**: `Transfer` 事件签名的哈希 (必须要有)
- **Topic 2**: `from` 地址 (indexed)
- **Topic 3**: `to` 地址 (indexed)
- **Data部分**: `value` (因为没有被 indexed)

所以我们需要使用 `log3`。
指令格式：`log3(offset, size, topic1, topic2, topic3)`。

接下来我们要准备两部分内容：
1.  **内存数据 (Data)**：你需要先将非索引参数写入内存，这里是 `amount`。
2.  **Topics**：准备好三个 Topic 的值。

代码构建：
```yul
// 1. 准备 Data
mstore(0x00, amount)

// 2. 准备 Topics 并发射
// topic1: Transfer 事件签名
// topic2: from (0x00...00)
// topic3: to (msg.sender)
let transferSig := 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
log3(0x00, 32, transferSig, 0, caller())
```

> **为什么来源是 0？** 
> 因为我们在做 `mint()` 函数，这就是一个凭空印钱的函数，代币不是从某人手里转出来的，而是从虚空中产生的。
> 在区块链上，我们用 **零地址** 来代表虚空或者系统。

### 3.3 完整代码展示

把它们拼接在一起，看看 `mint` 函数用汇编重写后的完整样子：

```solidity
function mint(uint256 amount) public {
    assembly {
        // ---------------------------------------------------
        // 1. 更新 balanceOf[msg.sender] += amount
        // ---------------------------------------------------
        mstore(0x00, caller())   // 内存[0x00] = msg.sender
        mstore(0x20, 4)          // 内存[0x20] = 4 (balanceOf Slot ID)
        
        let slotBalance := keccak256(0x00, 64) // 计算 Mapping 哈希槽位
        
        // 读取 -> 加法 -> 写入 (Load -> Add -> Store)
        let currentBal := sload(slotBalance)
        sstore(slotBalance, add(currentBal, amount))

        // ---------------------------------------------------
        // 2. 更新 totalSupply += amount
        // ---------------------------------------------------
        // totalSupply 固定在 Slot 3
        let currentSupply := sload(3)
        sstore(3, add(currentSupply, amount))

        // ---------------------------------------------------
        // 3. 发射 Transfer 事件
        // ---------------------------------------------------
        // 需要先计算事件签名的哈希 (Transfer(address,address,uint256))
        // keccak256("Transfer(address,address,uint256)")
        // = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
        let transferSig := 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
        
        mstore(0x00, amount) // 将 amount (Data部分) 写入内存准备发射
        
        // log3(offset, size, topic1, topic2, topic3)
        log3(0x00, 32, transferSig, 0, caller())
    }
}
```

---

## 4. 总结与权衡 (Trade-offs)

用汇编重写之后，不仅省去了 Solidity 默认的溢出检查，还跳过了很多中间变量的读写。
但是，看着我们刚写好的汇编代码，你觉得相比于原来的 Solidity 代码，我们牺牲最大的代价是什么？

虽然我们在运行时省下了用户的 Gas，但是为了实现这些手动逻辑，代码的字节码体积往往会膨胀，导致部署成本的增加。这就是一个典型的 **“以空间换时间”** (部署时的空间费用 vs 运行时的 Gas 费用) 的例子。

### 对比分析

| 维度 | 标准 Solidity | Yul 汇编优化 |
| :--- | :--- | :--- |
| **可读性** | 高，逻辑一目了然 | 低，像在看天书 |
| **安全性** | 编译器帮你做检查 (溢出、零除等) | 全靠手写，漏写一个 if 就可能导致资金被盗 |
| **审计难度** | 审计员只要看逻辑漏洞 | 审计员需要逐行检查堆栈操作，容易看走眼 |
| **维护性** | 容易修改和升级 | 改一行代码可能要重新计算所有堆栈位置 |

**结论**：通常我们只会对 **调用频率极高** 且 **逻辑相对简单** 的核心函数（如 `transfer`, `approve`）使用这种极端的优化。

---

## 5. 实战演练 (Homework)

剩下的几个函数就交给你来进行实战演练了！试着把 `burn` (销毁) 或者 `transfer` 函数也用汇编重写一遍。

**提示 Tips:**
1. `transfer` 需要检查余额是否足够 (`sub` 指令如果结果为负数在无符号整数里会下溢变成巨大的数字，Yul 不会报错，所以你需要手动检查 `gt` 或 `lt`)。
2. 别忘了更新两个人的余额（发送者减少，接收者增加）。
