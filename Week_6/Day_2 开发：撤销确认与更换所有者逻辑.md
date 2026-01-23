# 多签钱包：撤销机制与人事管理

如果说执行和提交是发动机，那么撤销和更换就是这个车的方向盘。

我们今天要构建多签钱包最核心的两个安全机制：**后悔机制**和**人事管理**。

---

## 第一部分：撤销签名（Revoke Confirmation）

### 场景设定

继续沿用昨天的话题，我们是管理保险柜的管理员。

刚刚有一个管理员 Alice 给 10 号交易签名了，同意执行转账。

但是后来 Alice 再看这笔交易的时候，发现收款地址好像写错了，这时候他想撤销签名。

### 撤销的限制条件

需要两个限制条件才能够撤销：

1. **这个交易本身还没有执行**：如果交易已经执行了，这个时候撤销签名也没有任何意义，而且还会破坏账本的记录，导致看的人一头雾水
2. **这个交易已经被签过字了**：如果检查到这个交易还没被 Alice 签过字，他想要撤销签字，那在逻辑上也行不通。如果可以执行的话，有一个捣乱的人一直撤销别人的签名，会导致交易永远无法执行

### 代码实现

```solidity
function revokeConfirmation(uint _txIndex) public onlyOwner {
    Transaction storage transaction = transactions[_txIndex];
    
    // 验证交易是否存在
    require(transactions.length > _txIndex, "tx not found");
    
    // 验证是否已经签名了
    require(isConfirmed[_txIndex][msg.sender], "tx not confirmed");
    
    // 验证交易是否已经执行
    require(!transaction.executed, "tx already executed");
    
    // 交易签名人数减一
    transaction.numConfirmed -= 1;
    
    // 修改签名状态为未签名
    isConfirmed[_txIndex][msg.sender] = false;
    
    // 发出通知告诉大家
    emit RevokeConfirmation(msg.sender, _txIndex);
}
```

---

## 第二部分：添加管理员（Add Owner）

### 为什么需要提案机制？

我们接下来要进入下一部分，更换管理员了。

如果代码写的失误了，自己把自己踢出去了，或者把钱包弄成了死锁，里面的钱就永远取不出了。

为了防止这种情况，添加新管理员的第一步，就是必须解决**谁有资格添加**的问题。

如果随便一个人都能添加管理员，那我随便加几个自己的地址进来，一样可以通过多签，这是不合理的。

所以**添加新管理员，本身也算是一种提案**。

### 流程

1. 有一个人说，要把 Paul 添加进来做新管理员
2. 其他人开始投票
3. 投票够了，有一个人执行这个交易
4. 合约执行添加管理员功能

### 关键问题：谁是调用者？

这里有一个非常关键的问题：

走到第四步的时候，钱包合约内部去调用 `addOwner` 函数时，调用者到底是谁？

我们分析一下背后的原理：

1. 某个 owner 调用了外层的 `executeTransaction`
2. `executeTransaction` 内部执行了一行底层代码：`target.call(data)`
3. 当 `target` 填的是钱包合约自己的时候，对于被调用的 `addOwner` 来说，自己调用自己，那么 `msg.sender` 也一定是**合约本身**

这就是多签最精妙的地方：**我调用我自己**。

只有通过了多签投票流程，代码才能以合约自己的身份去执行敏感操作。

### onlyWallet 修饰器

现在为了防止有人直接调用 `addOwner` 函数，我们需要给函数加一把锁，写一个 `modifier`：

```solidity
modifier onlyWallet {
    require(msg.sender == address(this), "not authorized");
    _;
}
```

只要加上了这个修饰器，我们就给 `addOwner` 和 `removeOwner` 加上了最强的防盗门，只有经过多签流程，由合约自己发起的调用才能通过。

### 代码实现

```solidity
function addOwner(address _newOwner) public onlyWallet {
    require(_newOwner != address(0), "empty address");
    require(!isOwner[_newOwner], "already exist");
    
    isOwner[_newOwner] = true;
    owners.push(_newOwner);
    
    emit AddOwner(_newOwner);
}
```

---

## 第三部分：移除管理员（Remove Owner）

### 算法挑战：Swap and Pop

我们再接着看最复杂、最容易出错的 `removeOwner`。

这里有一个算法层面的问题，在 Solidity 中，数组不像 Python 或者 JS 那样灵活。如果你直接删除了一个元素 `owners[i]`，那么他就会变成 `0x000...00`。

数组长度不会变短，只会在中间留下一个空洞。

为了保持数组长度和节省 gas，通常使用**Swap and Pop 技巧**。

### 可视化过程

假如我们的列表是这样的：

```
[Paul, Amy, Alice, Bob]
```

现在要移除 Paul。

但是不能整个移动数组往前，那样在链上非常贵。

所以我们的策略是：

1. 把 Bob 挪到 Paul 的位置，也就是把数组最后一个人填入我们要替换的位置
2. 把 Bob 删除

这样不管数组有多长，消耗的 gas 永远是一样的（O(1)）。

**可视化这个过程**：

1. 复制：`[Bob, Amy, Alice, Bob]`
2. 删除：`[Bob, Amy, Alice]`

### 清理 Mapping

这个时候虽然在名单中把 Paul 删除了，但是在快速查验身份的 mapping 中他还在：

```solidity
mapping(address => bool) public isOwner;
```

如果不修改这个 mapping，那他虽然不在名单上了，但是还能在某些时候蒙混过关。

所以我们要修改这个 mapping，以保证 Paul 彻底没有权限。

### 防止死锁

还有一个问题，如果我们门槛是 4，现在删除了 Paul，人数变成 3，那么就彻底锁死了！

所以还得判断当前的门槛在删除完后还能不能正常运作。

### 代码实现

以上，总合所有条件，我们写出完整的 `removeOwner` 代码：

```solidity
function removeOwner(address _owner) public onlyWallet {
    // 检查有这个 owner 吗
    require(isOwner[_owner], "not owner");
    
    // 检查删除完之后还能够人数签名吗
    require(owners.length - 1 >= numConfirmationsRequired, "chain death: threshold too high");
    
    // 移除 owner 权限
    isOwner[_owner] = false;
    
    // 数组操作：Swap and Pop
    // 假设我们要删的人在 index 位置 (实际代码通常需要一个 helper 函数或者前端传进来 index)
    // 这里为了演示简便，假设找到了 index
    // owners[index] = owners[owners.length - 1];
    // owners.pop();
    
    emit OwnerRemoval(_owner);
}
```

### 边界情况

这里有一个问题，假设只有你一个人了，然后门槛也是 1，你现在想把你自己也删除了，弃用这个钱包，可以做到吗？

**显然是不可以**，`require(owners.length - 1 >= numConfirmationsRequired, "chain death: threshold too high");` 这个没法通过。

如果不阻止你，钱包就会变成死钱包。

---

## 第四部分：修改门槛（Change Requirement）

### 场景

现在管理员是 `[Paul, Amy, Alice, Bob]`，门槛是 4。

你要裁掉一个人，这时候为了保证安全，`removeOwner` 函数阻止了你。

为了顺利进行，我们需要降低门槛。

### 限制条件

这个函数看起来很简单，但是为了安全，这个数字不能随便填写：

- **最大值**：门槛的最大值就是管理员的数量，超出去则永远不能进行交易
- **最小值**：只要大于 0，哪怕是 1 也可以

### 代码实现

这下我们写出完整的代码：

```solidity
function changeRequirement(uint _required) public onlyWallet {
    require(_required > 0 && _required <= owners.length, "invalid requirement");
    
    numConfirmationsRequired = _required;
    
    emit RequirementChange(_required);
}
```

完成！

---

## 第五部分：EVM 的被动性

### 最后一个问题

如果有一个交易，3 个人已经签字了，我们删除了一个管理员降低了门槛，变成 3 个人就可以执行，这时候他会自动执行吗？

***不会的***，EVM 是被动的，哪怕满足了所有条件，不通过 EOA 去调用它一下，它会永远在那里静止。

就像多米诺骨牌，虽然摆好了，但是必须有人推倒第一块。

---

## 总结

### 1. 撤销签名（Revoke Confirmation）

**功能**：允许管理员在交易执行前撤回自己的签名。

**限制条件**：
- 交易必须存在
- 交易尚未执行
- 调用者必须已经签过名

**核心逻辑**：
- 减少签名计数
- 更新签名状态映射
- 发出撤销事件

### 2. 添加管理员（Add Owner）

**核心机制**：通过多签提案流程添加新管理员。

**关键设计**：
- 使用 `onlyWallet` 修饰器，确保只能通过多签流程调用
- `msg.sender == address(this)` 保证是合约自己调用自己
- 防止添加零地址或重复地址

**精妙之处**：合约通过 `delegatecall` 调用自己，实现"我调用我自己"的安全机制。

### 3. 移除管理员（Remove Owner）

**算法优化**：使用 Swap and Pop 技巧，O(1) 时间复杂度。

**安全检查**：
- 验证要删除的地址是管理员
- 确保删除后剩余人数 ≥ 门槛（防止死锁）
- 同时清理数组和 mapping

**边界情况**：不允许删除最后一个管理员，防止钱包变成死钱包。

### 4. 修改门槛（Change Requirement）

**功能**：动态调整多签门槛。

**限制条件**：
- 门槛必须 > 0
- 门槛必须 ≤ 管理员总数

**使用场景**：在删除管理员前先降低门槛，避免死锁。

### 5. EVM 的被动性

**重要概念**：EVM 不会主动执行任何操作。

即使所有条件都满足（如签名数达到门槛），也需要有人主动调用 `executeTransaction` 才能执行。

### 核心安全机制总结

| 机制 | 作用 | 关键点 |
| :--- | :--- | :--- |
| **onlyOwner** | 限制只有管理员能操作 | 基础权限控制 |
| **onlyWallet** | 限制只能通过多签流程 | 防止直接调用敏感函数 |
| **撤销签名** | 提供后悔机制 | 交易执行前可撤回 |
| **Swap and Pop** | 高效删除数组元素 | O(1) 时间复杂度 |
| **门槛检查** | 防止死锁 | 确保始终有足够人数签名 |

### 一句话总结

多签钱包通过 `onlyWallet` 修饰器实现"合约调用自己"的安全机制，配合撤销签名、动态管理员和门槛调整功能，构建了一个既灵活又安全的去中心化资产管理系统，但需要注意 EVM 的被动性和防止死锁的边界条件。
