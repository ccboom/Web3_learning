# W6D1：多签钱包（Multi-Sig Wallet）

多签钱包是区块链世界中最重要的基础设施之一，它管理的资金量甚至超过了许多银行。

---

## 一、多签钱包的基本概念

### 1.1 现实生活中的类比

想象有一个巨大的保险箱：

- **普通钱包（EOA）**：只有一个钥匙孔，谁拿到钥匙谁就能把钱拿走，太不保险了，万一钥匙丢了都没地找去
- **多签钱包（Multi-Sig）**：这是一个特制的保险箱，有好几个钥匙孔，需要很多人同时协作才能打开

### 1.2 门槛设计的考量

我们现在雇了三个人（Alice、Bob、Mark），为了既安全又方便，我们得几个人同时打开比较好？

**如果我们选择三个人同时开**：
- 假如有一个人去度假了，那么保险柜就打不开了
- 如果有一个把钥匙丢了，那么保险柜就永远都打不开了

**所以三个人同时打开不合理，最合理的是两个人**：
1. **防止作恶**：如果一个人就能打开保险柜，那么有可能会被某一人把钱都拿走
2. **防止意外**：万一有一个人钥匙丢了，那么两个人还能打开保险柜，这就是容错

---

## 二、多签钱包的核心数据结构

### 2.1 管理员名单

我们需要一个列表，把这些人都记下来，我们使用一个数组来保存：

```solidity
// 这是一个名单，记着谁拿着钥匙
address[] public owners;
```

### 2.2 门槛（Threshold）

我们要规定几个人同意才能打开保险柜：

```solidity
// 这是一个数字，比如 2
uint256 public numConfirmationsRequired;
```

在代码中我们通常使用 `numConfirmationsRequired` 表示需要多少个人确认。

OK，那我们目前为止只有两行数据：
1. `owners[alice, bob, mark]`
2. 门槛：2

### 2.3 优化：使用哈希表快速验证

想象有一天，我们存的钱越来越多，公司人也越来越多了，三个人管理不太行了，我们改为 50 个人管理（Owners），还得雇个保安来保护保险柜。

这时候有人过来说：我是管理员之一，我要打开保险柜。

那么 `address[] public owners` 只有这个名单，智能合约（保安）只能这么做：
- 拿着名单，看第一行是这个人吗，不是的话看第二行是吗，第三行是吗，一直看完或者找到这个人为止

**这种循环在区块链上是非常昂贵的**，名单越长 Gas 费越高，如果名单太长的话 Gas 可能高到让交易失败！

**解决方案：使用哈希表**

我们给保安一个电脑，只要他输入名字（地址）就可以立马显示是不是管理员，不需要手动去翻阅名单了。

```solidity
// 输入一个地址，告诉你他是不是管理员
// 比如输入 Alice 的地址 => 返回 true
mapping(address => bool) public isOwner;
```

这个 mapping 的作用就是快速证明是不是管理员，方便快捷。

---

## 三、提案（Transaction）结构

### 3.1 提案的必要信息

当 Alice 想从保险柜里拿点钱，去 Uniswap 买币的时候，她需要发起一个**提案（Proposal/Transaction）**。

这张提案必须写在纸上，让大家都看看，纸上需要写清楚信息：

1. **金额**：我们肯定要确定转账金额
2. **转账的地址**：我们也需要知道往哪转
3. **指令**：我们需要知道 EVM 要处理什么指令，也就是这个提案到底要干嘛

### 3.2 提案结构体

```solidity
struct Transaction {
    address to;              // 转给谁
    uint256 value;           // 多少钱
    bytes data;              // 什么具体操作

    // 还有两个辅助信息，方便管理
    bool executed;           // 这件事情办完了没？防止重复操作
    uint256 numConfirmations; // 现在有几个人通过了这个提案？方便计数
}
```

### 3.3 提案存储

我们需要一个本子，上面记录了提案的顺序：

```solidity
Transaction[] public transactions;
```

`transactions[0]` 就是第 1 号提案，`transactions[1]` 就是第 2 号提案……以此类推。

### 3.4 签名记录

好了，我们在纸上写了提案，贴到墙上了，比如：
- ID：0
- 去向：Paul
- 金额：100 ETH
- 指令：无

现在 Alice 和 Bob 觉得都没问题，想签字批准。

我们需要一个牌子，上面会记录哪位管理员批准了哪个提案。

这是一个经典的**双重查询**：
1. 先查是**哪笔交易**（transaction ID）
2. 再查是**谁**（Owner Address）
3. 结果：签了还是没签（True/False）

在 Solidity 中，这个是 mapping 套 mapping：

```solidity
mapping(uint256 => mapping(address => bool)) public isConfirmed;
```

- Key 1 是交易编号，为 `uint256`
- Key 2 是管理员地址，也就是 `address`

---

## 四、多签钱包合约代码

### 4.1 基本骨架

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract MultiSigWallet {
    address[] public owners;                 // 管理员名单
    mapping(address => bool) public isOwner; // 快速核验身份
    uint256 public numConfirmationsRequired; // 通过提案门槛

    // 提案结构体
    struct Transaction {
        address to;
        uint256 value;
        bytes data;
        bool executed;
        uint256 numConfirmations;
    }

    // 存储交易的本子
    Transaction[] public transactions;
    
    // 记录到底谁通过提案的本子
    // 交易ID => 管理员地址 => 是否同意
    mapping(uint256 => mapping(address => bool)) public isConfirmed;
}
```

### 4.2 构造函数（Constructor）

现在一切都准备好了，我们得弄一个开业仪式，把第一批管理员和规矩都定下来。

这就是 `constructor` 函数，构造函数只会在部署合约的时候运行一次。

#### 需要防范的问题

1. 如果在 `_owners` 中，不小心加入了 `0x00...00` 地址怎么办？
2. 如果 `owners` 中只有三人，但是 `_numConfirmationsRequired` 设置为 4 了怎么办？
3. 如果 `_numConfirmationsRequired` 设置为 0 了怎么办？

- 如果加入 `0x00...00`，比如我们有 3 个 owners，但是门槛又设置为 3，那么钱就永远取不出来了
- 如果有 3 个人设置为 4，那也和上面情况是一样的
- 如果设置为 0，那么就随便有个人就能取钱，多签钱包的意义就失去了

#### 构造函数代码

```solidity
constructor(address[] memory _owners, uint256 _numConfirmationsRequired) {
    // 检查1：管理员不能为空
    require(_owners.length > 0, "Owners required");

    // 检查2：门槛必须合理，不能大于管理员数量
    require(
        _numConfirmationsRequired <= _owners.length && _numConfirmationsRequired > 0,
        "Invalid number of required confirmations"
    );

    for (uint i = 0; i < _owners.length; i++) {
        address owner = _owners[i];
        // 管理员地址不能为0
        require(owner != address(0), "Invalid owner");
        // 防止管理员重复
        require(!isOwner[owner], "Owner not unique");
        
        isOwner[owner] = true;  // 设置管理员mapping，为了快速检查
        owners.push(owner);     // 把管理员写入名单
    }
    
    numConfirmationsRequired = _numConfirmationsRequired;
}
```

---

## 五、多签钱包操作流程

多签钱包操作流程分为三步：

1. **Submit**：有人提出提案
2. **Confirm**：管理员确认提案
3. **Execute**：执行提案

### 5.1 第一步：Submit（提交提案）

比如我们说 Alice 想给 Paul 转 100 ETH，我们把这个函数叫 `submitTransaction`。

#### 权限控制

你觉得这个 `submitTransaction` 是所有人都能调用，还是只有管理员可以调用？

**答案：只有管理员才可以调用**

虽然在以太坊上写提案需要发起人自己支付 Gas，如果你让所有人都能提交提案的话，黑客也是需要自己付 Gas 的。但是他会恶心你，比如他故意发送 100000 个垃圾提案，这会导致你的 `transactions` 数组变得无比巨大，这时候你打开前端的时候网页直接卡死，真正你想要的提案却找不到了。这就是 **DoS（拒绝服务）攻击**。

所以我们必须要设置一下只有管理员才能提案。

#### 修饰符：只有管理员

```solidity
modifier onlyOwner() {
    require(isOwner[msg.sender], "Not owner"); // 只有验证了管理员的指纹才能通过
    _; // 继续运行函数体
}
```

#### 提交提案函数

现在 Alice 要发起提案了，逻辑很简单：发起提案 → 塞进数组 → 告诉大家

```solidity
function submitTransaction(
    address _to,
    uint256 _value,
    bytes memory _data
) public onlyOwner {
    // 获取提案的ID
    uint256 txIndex = transactions.length;

    // 创建新交易并加入数组
    transactions.push(
        Transaction({
            to: _to,
            value: _value,
            data: _data,
            executed: false,      // 刚创建，肯定没执行
            numConfirmations: 0   // 刚创建，赞成票为 0
        })
    );

    // 告知大家，有个新提案
    emit SubmitTransaction(msg.sender, txIndex, _to, _value, _data);
}
```

### 5.2 第二步：Confirm（确认提案）

现在提案已经在数组里面了，第一笔提案的 ID 是 0。

Bob 登录后看到了这个提案，然后点了**通过**按钮。

#### 防止重复签名攻击

如果 Bob 昨天已经通过一次了，今天又看到之后又通过了一次，我们没设置检查，会发生什么？

我们假设 Bob 想要把钱都拿走：
1. Bob 发起提案，给他自己的地址转账 1 个 ETH
2. Bob 调用 `confirmTransaction(0)`，签了一次，`numConfirmations += 1`，当前获得了一个签名
3. Bob 又调用了一次，`numConfirmations += 1`，当前获得了两个签名！
4. 因为签字数量够了，所以 Bob 执行了这次提案，然后 Bob 故技重施，拿走了所有的钱！

**所以我们在增加票数之前，先检查一下 `isConfirmed` 这个本子，如果查到已经签名了，直接报错。**

#### 确认提案函数

```solidity
function confirmTransaction(uint256 _txIndex) public onlyOwner {
    // 检查是否有这个交易，没有就报错
    require(_txIndex < transactions.length, "tx does not exist");
    // 检查交易是否已经执行，已经执行就报错
    require(!transactions[_txIndex].executed, "tx already executed");
    // 检查是不是已经签过字，已经签过的话就报错
    require(!isConfirmed[_txIndex][msg.sender], "tx already confirmed");

    // 所有检查通过，开始记录
    // 签字的数量 +1
    transactions[_txIndex].numConfirmations += 1;
    // 把管理员设置为已经签字的状态
    isConfirmed[_txIndex][msg.sender] = true;
    // 通知大家有管理员签字了
    emit ConfirmTransaction(msg.sender, _txIndex);
}
```

### 5.3 第三步：Execute（执行提案）

Alice 拿着签字够了的提案去执行，合约需要做两件事：
1. 把状态改为已经执行（`executed = true`）
2. 把钱真的转走

#### 执行提案函数

```solidity
function executeTransaction(uint256 _txIndex) public onlyOwner {
    Transaction storage transaction = transactions[_txIndex];

    // 检查是否签名数量够，还有检查是否已经执行过了
    require(
        transaction.numConfirmations >= numConfirmationsRequired,
        "cannot execute"
    );
    require(!transaction.executed, "tx already executed");

    // 设置状态为已经执行
    transaction.executed = true;
    // 执行交互内容
    (bool success, ) = transaction.to.call{value: transaction.value}(
        transaction.data
    );
    require(success, "tx failed");
    // 告诉大家提案已经执行了
    emit ExecuteTransaction(msg.sender, _txIndex);
}
```

---

## 六、Gas 优化：Gnosis Safe 的链下签名方案

### 6.1 传统多签的 Gas 问题

完成！所有的逻辑都已经完成了，但是有一个大问题，**在链上花的钱太多了**。

我们假设是一个 10 人的保险柜，5 个管理员签字可以运行：

1. Alice 提交交易：1 次 gas
2. Bob 批准：1 次 gas
3. Charlie 批准：1 次 gas
4. Dave 批准：1 次 gas
5. Eve 批准：1 次 gas
6. Frank 执行：1 次 gas

**合计 6 次链上交易！** 如果以太坊比较拥堵的话，费用会直线上升。

### 6.2 Gnosis Safe：链下签名

Gnosis Safe 的意思是说，签名这种事没必要上链啊，链下解决了再传上去不就行了。

**传统方式**：Bob 说我同意，发送一笔修改 `isConfirmed` 变量的交易

**Gnosis 方式**：
- Bob 自己在链下用私钥对交易内容签名
- 其他管理员也在链下签名
- 最后大家把签名发给 Alice
- Alice 如果集齐 5 个签名，那么就打包为一个大包裹，发送一笔交易给合约
- **这就让 6 笔交易变成 1 笔！**

### 6.3 架构改变：极简主义

既然我们改用了链下签名，那么我们的合约架构就得大改。

我们刚刚使用 `mapping(uint => mapping(address => bool)) isConfirmed` 来记录谁签字了。

但是在 Gnosis 模式下，Alice 给合约传进来一串签名数据，合约要做现场数学运算。

#### ecrecover：签名验证机器

有一个神奇的机器：
- 把一份合同（Hash）放进左边
- 把一个签名放进右边
- 显示屏上就会显示：是 XXX 的签名！

在 EVM 中，我们学过，使用 `ecrecover` 可以验证签名，也就是 **Elliptic Curve Recover**（椭圆曲线恢复）。

```solidity
// 机器显示的人名 = ecrecover(合同哈希, 签名的V, 签名的R, 签名的S)
address signer = ecrecover(msgHash, v, r, s);
```

#### Gnosis 的极简设计

有了这个机器，Gnosis 做了一个大胆的决定：**把我们刚刚写的变量，统统删掉！**

这就是为什么 Gnosis Safe 能成为行业标杆的原因：**极简主义**

**不需要 `isConfirmed` 了**：
- 不需要记录谁签了，你们在链下签好，传上来签名，执行的时候用 `ecrecover` 验证一遍，人数够了直接放行

**不需要 `transactions` 了**：
- 交易内容直接包含在签名信息传进来就行了，只要签名验证通过，说明管理员确实同意这次交易，合约不需要像写日记一样保存下来

### 6.4 防重放攻击：Nonce

但是这时候就有个问题，`transactions` 数组被删除了，Alice 拿着签名转账了，过一会 Alice 又来了，还是这个签名，还能转账，这是不对的。

为了解决这个问题，Gnosis 引入了它最核心的业务状态变量：**nonce**

```solidity
uint256 public nonce;
```

它的逻辑就像银行流水号：
1. 所有签名中，必须包含当前的 `nonce` 值，假如是 0
2. 合约执行时，检查签名的 `nonce` 是不是 0
3. 执行成功后，合约会把 `nonce` 改为 1
4. 你再拿着 `nonce` 为 0 的签名过来，合约直接会拒绝你，因为已经执行过了

---

## 总结

### 传统多签钱包

**核心组件**：
- `owners[]`：管理员名单
- `isOwner`：快速验证身份的哈希表
- `numConfirmationsRequired`：通过门槛
- `transactions[]`：提案存储数组
- `isConfirmed`：双重映射记录签名状态

**操作流程**：
1. Submit：管理员提交提案
2. Confirm：管理员逐个确认（每次都上链）
3. Execute：达到门槛后执行

**缺点**：Gas 费用高，每个管理员确认都需要一次链上交易

### Gnosis Safe 优化方案

**核心创新**：链下签名 + 链上验证

**架构简化**：
- 删除 `isConfirmed` 映射
- 删除 `transactions[]` 数组
- 只保留 `nonce` 作为防重放机制

**优势**：
- 将多笔交易合并为一笔
- 大幅降低 Gas 费用
- 使用 `ecrecover` 进行签名验证
- 极简主义设计，成为行业标杆

**防重放**：通过 `nonce` 机制确保每个签名只能使用一次
