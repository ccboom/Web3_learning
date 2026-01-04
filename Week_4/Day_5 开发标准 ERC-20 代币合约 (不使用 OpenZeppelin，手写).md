# ERC-20 代币标准详解

我们今天就是为了看清代币的本质，它不是你钱包里叮当作响的硬币，只是合约里一行账本数据。

---

## 第一部分：心理模型与数据结构

### 正确的心理模型

ERC-20 合约的核心其实就是一个**巨大的 EXCEL 表格**。当你说我有 100 个币的时候，只是合约里记录着：地址 `0x001....` → 余额 100。

### 设计数据库

除了代币名字（name）、符号（symbol）和小数位（decimals）这些基本信息外，最重要的就是存储余额和授权信息。

#### 余额表（Balance Mapping）

在 solidity 语法中，我们不像在 SQL 数据库里那样建表，我们用一种叫做 **mapping** 的数据结构。

你可以把它想象成一个"自动售货机的储物格"：

- 你给他一个 key（钥匙/索引）
- 它立马给你吐出一个 value（值）

**语法格式**：

```solidity
mapping(Key类型 => Value类型) 可见性 变量名;
```

**我们的需求：余额表**

我们要记录**谁**有多少钱：

- **谁**：以太坊地址，`address`
- **多少钱**：正整数。在 solidity 里，通常用 `uint256`（表示 0 到 $2^{256}-1$ 的巨大整数，不能是负数，也不能是小数）

所以，余额表的代码是这样的：

```solidity
mapping(address => uint256) public balanceOf;
```

`public` 是一个关键字，加上他，任何人都可以直接读取这个变量，不需要再写一个 `getBalance` 函数了。

#### 授权表（Allowance Mapping）

接下来我们说说授权，这是 ERC-20 中最绕的地方。

想象一下，我现在想允许 uniswap（一个去中心化交易所）转走我的 100 个币，这时候有三个信息：

1. 我的地址（owner）
2. uniswap 地址（spender）
3. 授权金额（Amount）

这时候我们就需要一个**表格里的表格**：

```solidity
//       Owner              Spender      Amount
mapping(address => mapping(address => uint256)) public allowance;
```

这是一个大的 mapping，它的 key 是 address 也就是你。

它的 value 又是一个 `mapping(address => uint256)`：

- 这个内部的 key 是 address（被授权的人，比如 uniswap）
- 这个内部的 value 是授权的金额大小 `uint256`

### 理解二维 Mapping

我们自己检测一下到底懂了没懂。

比如我们有以下场景：

- Alice (0xA)
  - 余额: 1000
  - 授权给 Bob (0xB): 50
- Bob (0xB)
  - 余额: 0

这时候，我们要用代码查询 Alice 给 Bob 授权了多少钱，应该是以下哪个？

- A. `allowance[0xB][0xA]`
- B. `allowance[0xA][0xB]`

**答案是 B**：`allowance[0xA][0xB]`

- 第一层 Key `0xA` (Alice) 找到了她的所有授权记录。
- 第二层 Key `0xB` (Bob) 在 Alice 的记录里找到了专门给 Bob 的那个数字。

这就像是查字典：先找首字母 A (Alice)，再找第二个字母 B (Bob)。

---

## 第二部分：事件（Events）

### 为什么需要事件？

你可能在 etherscan 上面看过代币的转账记录，那里列出了 From、To、Amount。

但是区块链本身只更改了状态，它是不会主动通知外界的。如果你不喊一声，etherscan、钱包、交易所根本不知道刚刚发生了转账，用户余额在前端也不会更新。

### 事件定义

solidity 用 `event` 关键字来喊话，如下：

```solidity
// 定义事件：当转账发生时，打印这张"收据"
// indexed 关键字：允许链下工具（如 The Graph）快速根据这个字段搜索日志
event Transfer(address indexed from, address indexed to, uint256 value);

// 定义事件：当授权发生时
event Approval(address indexed owner, address indexed spender, uint256 value);
```

---

## 第三部分：核心函数实现

### 1. Transfer 函数（转账）

这是最常用的函数，通常是 A 转给 B 多少钱。

```solidity
function transfer(address to, uint256 amount) public returns (bool) {
    // 逻辑写在这里
}
```

**为什么只有 to 没有 from？**

因为只有拥有私钥的人才能发起交易。在 solidity 中，有一个全局变量叫 `msg.sender`，它代表了谁在调用这个函数，所以 from 永远默认都是 `msg.sender`。

#### 标准的转账逻辑（四步）

1. **检查（Check）**：转账的人钱够吗？
2. **扣钱（Deduct）**：转账人余额减少。
3. **入账（Credit）**：接收人余额增加。
4. **通知（Emit）**：发送 transfer 事件

#### 完整实现

```solidity
function transfer(address to, uint256 amount) public returns (bool) {
    // 1. 检查余额是否充足
    require(balanceOf[msg.sender] >= amount, "Not enough tokens");
    
    // 2. 扣除发送者的余额
    balanceOf[msg.sender] -= amount;
    
    // 3. 增加接收者的余额
    balanceOf[to] += amount;
    
    // 4. 触发事件
    emit Transfer(msg.sender, to, amount);
    
    // 5. 返回成功
    return true;
}
```

### 2. Mint 函数（铸造）

我们使用了 transfer，钱在不同的人之间流动，那钱一开始是哪里来的？

这就需要一个 `mint` 函数，它的逻辑是：凭空产生一笔钱，给某个人，同时增加总供应量（totalSupply）。

```solidity
function mint(uint256 amount) public {
    // 1. 增加总供应量
    totalSupply += amount;

    // 2. 增加这个人的余额
    balanceOf[msg.sender] += amount;
    
    // 3. 触发事件 (由 0x0 地址发出代表铸造)
    emit Transfer(address(0), msg.sender, amount);
}
```

### 3. Approve 函数（授权）

想象你在 uniswap 上卖掉了 100 个代币，你不能直接转给 uniswap，这样相当于把币送他了。

**流程**：

1. 你（owner）通知合约：批准 uniswap 动用我 100 个代币 → `approve` 函数
2. uniswap 看到批准后，向合约发送指令：把 owner 的代币转给买家 → `transferFrom` 函数

就像你绑定卡在支付宝：你授权支付宝可以在你卡里扣钱。

#### 实现

```solidity
function approve(address spender, uint256 amount) public returns (bool) {
    // 记录 msg.sender 允许 spender 动用 amount
    allowance[msg.sender][spender] = amount;   
    emit Approval(msg.sender, spender, amount);
    return true;
}
```

**关键点**：

- 我是发起人：`msg.sender`
- 我要授权的人是参数：`spender`
- 我要授权的金额是参数：`amount`

**注意**：这里不是"增加" (`+=`)，而是直接"覆盖/设置" (`=`) 为新的金额。

在 ERC-20 协议中，approve 的意思是重新设定额度，而不是追加额度。

**举个例子**：

你授权了淘宝可以扣 50，你现在想把额度改为 100。如果是加，那么现在变成了 50+100=150，这肯定不是你的本意，所以就用替代，直接设置为 100 元。

### 4. TransferFrom 函数（代理转账）

现在 uniswap 要来调用合约，它说：我要把 Alice（sender）的 100 个币给 Bob（recipient）。

这个函数叫 `transferFrom`，它比普通转账多了一个步骤：检查并扣除授权额度。

由三步进行：

1. **查额度**：Uniswap（msg.sender）有没有权利动用 Alice 的钱？
2. **扣额度**：Uniswap 用掉了 100 额度，那么剩余额度要减去 100
3. **转账**：真正把钱从 Alice 挪到 Bob 那里

#### 实现

```solidity
function transferFrom(address sender, address recipient, uint256 amount) public returns (bool) {
    // 1. 检查授权额度是否足够
    require(allowance[sender][msg.sender] >= amount, "Allowance exceeded");
    
    // 2. 扣除授权额度
    allowance[sender][msg.sender] -= amount;
    
    // 3. 扣除老板 (sender) 的余额
    balanceOf[sender] -= amount;
    
    // 4. 增加接收者 (recipient) 的余额
    balanceOf[recipient] += amount;
    
    emit Transfer(sender, recipient, amount);
    return true;
}
```

**关键理解**：

- 谁是老板（owner）：参数里的 `sender`
- 谁是中介（Spender）：调用函数的 `msg.sender`

这三行代码正是所有去中心化金融的基石：

- 当你去 dex 交易时
- 当你去 Aave 借贷时
- 背后跑的全是你刚刚写的这段 `transferFrom` 逻辑

---

## 完整代码

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

## 总结

### 核心概念

1. **ERC-20 的本质**：一个记录地址余额的 EXCEL 表格
2. **两个关键 Mapping**：
   - `balanceOf`：记录每个地址的余额
   - `allowance`：记录授权关系（二维 mapping）
3. **两个关键事件**：
   - `Transfer`：转账事件
   - `Approval`：授权事件

### 四大函数

| 函数 | 作用 | 核心逻辑 |
| :--- | :--- | :--- |
| **transfer** | 直接转账 | 检查 → 扣钱 → 入账 → 通知 |
| **mint** | 铸造代币 | 增加总供应 → 增加余额 → 通知 |
| **approve** | 授权额度 | 设置 allowance → 通知 |
| **transferFrom** | 代理转账 | 检查授权 → 扣授权 → 转账 → 通知 |

### 一句话总结

ERC-20 就是通过 mapping 存储余额和授权，通过 event 通知外界，通过四个函数实现代币的转账、铸造和授权，是整个 DeFi 生态的基础。
