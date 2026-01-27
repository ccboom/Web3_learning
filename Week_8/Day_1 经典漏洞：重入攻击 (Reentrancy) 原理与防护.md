# 经典漏洞：重入攻击 (Reentrancy) 原理与防护

这周我们进入 **合约安全周**，分析一些典型的合约安全案例。
我们今天分析的就是 Web3 安全史上最著名的篇章 —— **重入攻击 (Reentrancy)**。
这不仅是一个技术漏洞，更是导致了 ETH 硬分叉、形成了 ETH 和 BTC 两个世界的重大历史事件。

## 1. 核心原理：递归与“回马枪”

我们来通过分布探索理解这个漏洞，首先我们要理解这个“回马枪”是怎么形成的。核心原理就是 **递归**。

在 Solidity 中，当一个合约向另一个合约发送 ETH 的时候，它的执行 **控制权** 会转移给接收方。

### 场景模拟
想象一下这个场景：
1.  **初始状态**：用户 A 在银行合约中有 100 块钱。
2.  **触发攻击**：用户 A（实际上是一个恶意合约）调用 `withdraw()` 要求取钱。
3.  **银行检查**：银行检查余额，确认 A 有 100 块钱。
4.  **转账开始**：银行把 100 块转给了合约 A。
5.  **回马枪**：关键的地方来了！在 A 收到 100 块的那一瞬间，用户 A 的合约会自动触发一个特殊的函数（如 `fallback` 或 `receive`）。
6.  **递归调用**：在这个函数中，用户 A 不结束交易，反而又向银行合约再次发送了一笔取款请求（此时银行还没来得及清除 A 的余额）。
7.  **再次转账**：银行合约一看：“A 还有 100 块”，于是继续给他转账 100...

这种在逻辑执行完之前再次“进去”的操作，就是 **重入攻击**。

---

## 2. The DAO 惨案：改变历史的漏洞

在 2016 年的 The DAO 事件中，攻击者就是利用了类似的代码逻辑。简化后的漏洞代码看起来像这样：

```solidity
// 警告：这是有漏洞的代码！
contract VulnerableBank {
    mapping(address => uint) public balances;

    function withdraw() public {
        uint amount = balances[msg.sender];
        require(amount > 0);

        // 1. 发送资金（此时控制权移交给攻击者）
        // 攻击者利用 msg.sender.call 会触发其合约代码的特性，
        // 反复进入 withdraw，直到把合约里的钱抽干净
        (bool success, ) = msg.sender.call{value: amount}(""); 
        require(success);

        // 2. 更新余额（可惜，如果发生重入，这一行永远跑不到）
        balances[msg.sender] = 0;
    }
}
```

### 为什么更新余额放在发送资金之后会导致严重后果？

在智能合约的世界中，代码是按照顺序执行的。逻辑是这样的：

1.  **查询**：合约看到你有 100 元。
2.  **给钱**：合约先把钱给你。注意，此时合约还没运行下一步的“扣款”动作，这时候你的余额记录仍然是 100 元。
3.  **被中断**：就在你收到钱的一瞬间，你的恶意合约立刻通过 `fallback` 函数再次调用 `withdraw`。
4.  **循环**：银行合约再次检查，发现你“还有” 100 元，于是又给你转了 100。

这就像在自助取款机取款：当钱吐出来的一瞬间，利用机器还没更新账单的 0.1 秒间隙，再次按下取款按钮，又把钱拿了出来。这种攻击的前提就是 —— **取款机在确认账单之前就先交出了控制权**。

即使代码运行得再快，只要它在修改余额之前把控制权交给了你（即执行了发送资金的操作），你就可以控制资金流程了。这时候它就处于了一个 **不一致的状态**，就像一个健忘的柜员，先把钱递出了窗口，还没来得及修改账本，就被你拉住又取了一笔钱。

---

## 3. 为什么 call 是危险的？

在 Solidity 中，有多种发送 ETH 的方式，比如 `transfer`、`send`、`call`。
但 `call` 是最常用的，也是最容易引发重入的。

*   **控制权移交**：当你使用 `msg.sender.call{value: amount}("")` 的时候，以太坊不仅会转账，还会把执行权的“交接棒”传给接收方。
*   **恶意回调**：如果接收方自己是一个合约，它会立即触发自己的 `fallback()` 或者 `receive()` 函数。黑客就在这个函数里写上：`银行.withdraw()`。

---

## 4. 防御手段

### 第一道防线：Checks-Effects-Interactions (CEI) 模式

我们需要改变上述的逻辑，更换为“先记账，再转账”的方式。

1.  **检查 (Checks)**：
    ```solidity
    require(balances[msg.sender] >= amount); // 确认还有没有钱
    ```
2.  **生效 (Effects)**：
    ```solidity
    balances[msg.sender] -= amount; // 先在账本上记下来，即使转账没发生，余额已经减掉了
    ```
3.  **交互 (Interactions)**：
    ```solidity
    (bool success, ) = msg.sender.call{value: amount}(""); // 最后才把钱送出去
    require(success); // 检查是否转账成功
    ```
这种先更新状态、后交互的模式在安全领域称为 **Checks-Effects-Interactions** 模式。

**问题**：万一先记账了，转账没转出去怎么办？
**回答**：不用担心。以太坊的交易具有 **原子性**。这意味着每一个交易中，所有操作要么都成功，要么都失败。我们在最后添加 `require(success)` 来检查是否转账成功，如果失败了，整个操作（包括之前的扣款）会全部回滚。

### 第二道防线：重入锁 (Reentrancy Guard)

有时候代码逻辑太复杂，很难严格遵守“检查-效果-交互”模式，分不清楚前后顺序怎么办？
这时候就会拿出我们的第二件法宝：**重入锁**。

就像你在进入洗手间之后，第一件事就是把门反锁。只要你没出来，外面的人无论怎么敲门都进不来。

代码实现大概如下：

```solidity
bool internal locked;

modifier noReentrant() {
    require(!locked, "No reentrancy"); // 检查门锁了没
    locked = true;                     // 进门，锁上
    _;                                 // 执行业务逻辑（比如取钱）
    locked = false;                    // 完事，开锁
}
```

假设有一个黑客在 `_;` (执行业务) 的时候想再次调用这个函数，他能成功吗？
显然是不行。他可以顺利进来，但是因为有锁 `require(!locked)`，把他拦住了，所以合约会由于多次进入而 `revert`。

---

## 5. 进阶讨论：多函数重入 (Cross-function Reentrancy)

先看看：**多函数重入**。
我们之前学的重入锁，通常加在某个函数上。但如果你的合约里有多个函数共享一个变量，而你只锁定了其中一个，攻击者就会从另一个没锁的函数中“溜进来”。

### 漏洞示例

```solidity
contract VulnerableBank {
    mapping (address => uint) public balances;
    bool internal locked;

    modifier noReentrant() {
        require(!locked, "No reentrancy");
        locked = true;
        _;
        locked = false;
    }

    // 1. 取款函数：加了锁，看起来很安全？
    // 注意：这里违背了 CEI 模式，是先发钱后改余额
    function withdraw() public noReentrant {
        uint amount = balances[msg.sender];
        
        // A. 发送资金 (此时控制权转交给攻击者)
        (bool success, ) = msg.sender.call{value: amount}(""); 
        require(success);

        // B. 更新余额 (只有等 A 结束才会执行)
        balances[msg.sender] = 0; 
    }

    // 2. 转账函数：糟糕，忘记加锁了！
    function transfer(address to, uint amount) public {
        if (balances[msg.sender] >= amount) {
            balances[to] += amount;
            balances[msg.sender] -= amount;
        }
    }
}
```

**攻击路径**：
假设账户中有 100 ETH。
1.  调用 `withdraw()`。
2.  程序运行到了 **A 行**，把 100 ETH 发给了你。
3.  你的合约触发 `fallback` 函数。
4.  在 `fallback` 中，你**不再**调用 `withdraw`（因为有锁），而是调用 `transfer`。
5.  `transfer` 没锁！它可以正常执行。此时你的余额还没被清零（因为 `withdraw` 里的 B 行还没执行）。
6.  你通过 `transfer` 把这 100 ETH 再次转到你的另一个账户。
7.  `transfer` 结束，回到 `withdraw`，最后才把余额清零。
8.  **结果**：你拿到了 100 ETH 现金，同时你的另一个账户里多了 100 ETH 存款。银行亏损两倍。

**教训**：我们设置重入锁的时候，必须 **全局生效**，或者覆盖所有读写共享状态的函数。如果只锁前门，不锁后窗，依然会被盗。

---

## 6. 工业级防线：OpenZeppelin 的省钱技巧

我们自己写的锁原理虽然简单，但是比较容易写错。
在真实的开发中，大家几乎都会直接继承 OpenZeppelin 的 `ReentrancyGuard`。

这里有一个有趣的细节。通常我们认为锁使用 `bool` 来作为开关最直接：
*   `false (0)` = 没锁
*   `true (1)` = 锁了

但是 OpenZeppelin 是这样实现的：

```solidity
// OpenZeppelin 的 ReentrancyGuard.sol
abstract contract ReentrancyGuard {
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;

    uint256 private _status;

    constructor() {
        _status = _NOT_ENTERED;
    }

    modifier nonReentrant() {
        require(_status != _ENTERED, "ReentrancyGuard: reentrant call");
        _status = _ENTERED;  // 上锁：把 1 变成 2
        _;
        _status = _NOT_ENTERED; // 解锁：把 2 变回 1
    }
}
```

它专门避开了 `0`，在 `1` 和 `2` 之间切换，为什么？

### EVM 的 Gas 优化技巧

这是一个 EVM 底层的省钱技巧（Gasrefund）：
1.  **0 -> 非 0**：把一个存储位置从“空”变成“有值”，消耗的 Gas 非常高（20,000 gas 左右），就像开垦荒地。
2.  **非 0 -> 非 0**：修改一个本来就有值的存储位置，消耗的 Gas 会少很多（2,900 gas 左右）。

如果是 `bool`，解锁状态通常是 `false (0)`。每次上锁把它变成 `1` 都会消耗高昂的“开垦费”。
然而在 `1` 和 `2` 之间切换，每次都是热更新，能省下不少 Gas。

---

## 7. 完整示例：攻击合约与安全合约

### 攻击合约示例

下面是一个完整的重入攻击示例：

```solidity
// 攻击者合约
contract Attacker {
    VulnerableBank public bank;
    
    constructor(address _bankAddress) {
        bank = VulnerableBank(_bankAddress);
    }
    
    // 发起攻击
    function attack() external payable {
        // 先存入一些 ETH
        bank.deposit{value: msg.value}();
        // 然后开始取款攻击
        bank.withdraw();
    }
    
    // fallback 函数：在收到 ETH 时自动触发
    fallback() external payable {
        // 如果银行还有钱，继续取款
        if (address(bank).balance >= 1 ether) {
            bank.withdraw();
        }
    }
    
    // 提取攻击所得
    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

### 安全的银行合约实现

下面是一个使用 CEI 模式和 OpenZeppelin 重入锁的安全实现：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SecureBank is ReentrancyGuard {
    mapping(address => uint) public balances;
    
    event Deposit(address indexed user, uint amount);
    event Withdraw(address indexed user, uint amount);
    
    // 存款函数
    function deposit() external payable {
        require(msg.value > 0, "Must deposit some ETH");
        balances[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }
    
    // 安全的取款函数：使用 CEI 模式 + 重入锁
    function withdraw() external nonReentrant {
        uint amount = balances[msg.sender];
        
        // 1. Checks: 检查余额
        require(amount > 0, "Insufficient balance");
        
        // 2. Effects: 先更新状态
        balances[msg.sender] = 0;
        
        // 3. Interactions: 最后才转账
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        emit Withdraw(msg.sender, amount);
    }
    
    // 查询合约余额
    function getContractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

---

## 8. 重入攻击防护最佳实践

### ✅ 推荐做法

1. **始终遵循 CEI 模式**
   - Checks（检查）：验证条件
   - Effects（生效）：更新状态变量
   - Interactions（交互）：调用外部合约或转账

2. **使用 OpenZeppelin 的 ReentrancyGuard**
   ```solidity
   import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
   
   contract MyContract is ReentrancyGuard {
       function sensitiveFunction() external nonReentrant {
           // 你的代码
       }
   }
   ```

3. **优先使用 `transfer()` 或 `send()`（在适用场景下）**
   - 这两个函数只转发 2300 gas，不足以执行复杂的重入攻击
   - 但要注意：在某些情况下可能会失败（如接收方是合约且 fallback 函数消耗 gas 较多）

4. **使用 Pull Payment 模式**
   - 不要主动推送资金，而是让用户自己来取
   ```solidity
   mapping(address => uint) public pendingWithdrawals;
   
   function withdraw() external {
       uint amount = pendingWithdrawals[msg.sender];
       pendingWithdrawals[msg.sender] = 0;
       payable(msg.sender).transfer(amount);
   }
   ```

### ❌ 避免的做法

1. **不要在状态更新前进行外部调用**
   ```solidity
   // ❌ 错误示例
   msg.sender.call{value: amount}("");
   balances[msg.sender] = 0;
   ```

2. **不要只锁定部分函数**
   - 如果多个函数共享状态，要么全部加锁，要么确保都遵循 CEI 模式

3. **不要忽视跨合约重入**
   - 攻击者可能通过调用你依赖的其他合约来实现重入

---

## 9. 总结

重入攻击是智能合约安全中最经典、最危险的漏洞之一。理解它的核心要点：

1. **根本原因**：在状态更新前交出控制权
2. **攻击手段**：利用 `fallback`/`receive` 函数递归调用
3. **防御策略**：
   - CEI 模式（首选）
   - 重入锁（辅助）
   - Pull Payment 模式（特定场景）

记住：**先记账，再转账**。这是防止重入攻击最简单也最有效的原则。

---

## 10. 延伸阅读

- [The DAO Hack 详细分析](https://www.gemini.com/cryptopedia/the-dao-hack-makerdao)
- [OpenZeppelin ReentrancyGuard 源码](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol)
- [Solidity 官方文档：安全考虑](https://docs.soliditylang.org/en/latest/security-considerations.html)
- [Consensys 智能合约最佳实践](https://consensys.github.io/smart-contract-best-practices/)
