# Context 与 Delegatecall：Solidity 的任督二脉

我们今天彻底进入 Week5 的内容。

Week5 的内容是 solidity 编程中最抽象的，但也是最强大的概念。一旦你领悟了这个，你就像打通了任督二脉。

今天的主题只有两个词：**context（上下文）** 和 **delegatecall（委托）**。

我们分几步走，慢慢把这两个讲清楚。

---

## 第一部分：普通调用 (Call) 的运作方式

### 场景比喻

比如现在有一个合约叫 A1 合约，有一个合约叫 B1 合约，我们使用 A1 来调用 B1 的函数。

我们假设这样一个例子：

- **你（user）**：是一个老板
- **A1**：是你的购货员
- **B1**：是供应商

现在你告诉 A1 说：去，到 B1 那里给我买个零件。

现在 A1 去了 B1 那里，告诉他说，要一个零件。

好了，B1 现在开始做，做完了拿给 A1 说弄好了，A1 把钱给了 B1，然后把东西拿回来，B1 把钱收到它自己的钱包中，这个事件就结束了。

### 关键问题

对于 B1 来说，谁是顾客？那肯定是 **A1**，A1 过去买的。

那么在记账的时候，写的也是 A1 的名字。

做零件消耗的也是 B1 的东西，用 B1 的材料做出来之后给 A1（**修改的是 B1 的状态**）。

### 代码示例

写个代码来看看，直观感受一下：

```solidity
contract B1 {
    uint256 public num;
    address public sender;
    
    function setNum(uint256 _num) public {
        num = _num;           // 修改B1的状态，我们这里以记录钱数为例
        sender = msg.sender;  // 记录谁来买的
    }
}

contract A1 {
    function callB1(address _B1, uint256 _num) public {
        B1(_B1).setNum(_num); // 拿钱到去B1那里买东西
    }
}
```

### 练习题

比如：用户地址是 `0x00`，合约 A1 地址是 `0x01`，合约 B1 地址是 `0x02`。

现在用户调用了合约 A1，合约 A1 又调用了合约 B1 的 `setNum(100)`，那么：

1. 合约 B1 的 num 变成了 100，请问这个数据存在合约 A1 还是合约 B1？
2. 合约 B1 里的 sender 会变成用户还是 A1？

**答案**：

- 数据存在 **B1**
- sender 会变成 **A1**

**规则**：去谁那里运行，修改的就是谁的数据；谁调用的，变成的就是谁的地址。

---

## 第二部分：委托调用 (Delegatecall)

### 重头戏来了

现在，重头戏来了，我们要使用 **delegatecall** 委托调用。

这是 solidity 最反直觉，但也是最强大的领域。

我们把 delegatecall 想象为：**偷学技术**。

### 场景比喻

还是刚刚那个场景：

你（user）是一个老板，你告诉 A1，我要一个零件，但是我不想让你出去买，你直接用我们场子里的东西现做就行了。

这时候 A1 有一个很厉害的东西，它使用了 delegatecall，这一步相当于 **A1 把 B1 的技术直接偷过来使用**。

### 发生的事情

- **B1 的内部逻辑代码**，被 A1 偷拿了过来使用
- **场地和材料**：使用的是 A1 的
- **对于这个逻辑来说**，让他开始做的是你（user）也就是老板，因为是你指挥 A1 去干活的，确实也是 A1 在干活，不过技术是 B1 的。

### 代码示例

我们再用代码来解释一下目前的状况：

```solidity
// 逻辑合约B1 - 提供代码供A1使用
contract B1 {
    // 这里定义只是为了告诉编译器变量的位置
    uint256 public num;
    address public sender;

    function setNum(uint256 _num) public {
        // 在delegatecall的时候，num指的是调用者A1的第一个变量位置
        // sender指的是用户（user）
        num = _num;
        sender = msg.sender;
    }
}

contract A1 {
    uint256 public num;     // slot0
    address public sender;  // slot1

    function callB1(address _b, uint256 _num) public {
        // A1 委托 B1来执行，相当于B1的代码复制过来执行逻辑
        (bool success,) = _b.delegatecall(
            abi.encodeWithSignature("setNum(uint256)", _num)
        );
    }
}
```

### 练习题

现在用户调用 A1 合约，A1 合约 delegatecall B1 合约的 `setNum(100)`：

1. 这个 100 最终存在了哪里？
2. 运行逻辑的时候，`msg.sender` 是谁？

**答案**：

- 存在了 **A1** 里面
- 运行的时候 `msg.sender` 是 **user**

### Solidity 高级编程的核心心法

- **数据**：谁调用逻辑在谁那里（A1）
- **用户**：谁调用合约是谁（user）
- **逻辑**：从被调用的逻辑（B1）那里借来的

### 可升级合约的原理

这就是所有可升级合约的原理（Upgradable Contracts）：

- **A1 是躯体**：只保存数据，不保存逻辑
- **B1 是灵魂**：只保存逻辑，不保存数据
- **想升级的时候**：把 C1 地址替换了 B1 的地址，即可改变逻辑，数据完全保留，只是灵魂换了。

---

## 第三部分：存储槽冲突 (Storage Collision)

### 黑客最喜欢的漏洞

我们还得额外再讲一个东西，黑客非常喜欢的东西，那就是 **存储槽冲突（Storage Collision）**。

虽然 delegatecall 非常厉害，但是也有一些副作用。

它根本不认识变量名，你叫 L1 也行，叫 BBBBBB 也可以，它统统不管，它只认**槽位（slot）**。

### 比喻

想象合约是一个超市存包柜子，按照序号上面写了 slot0、slot1...

- B1 的逻辑是把 slot 0 里写入 num，因为它是 B1 的第一个变量
- A1 在代理执行的时候，不管 slot 0 是哪个变量，它都会直接存储在 slot 0 里面

### 危险示例

看下面的例子：

```solidity
contract B1 {
    uint256 public num;  // slot0

    function setNum(uint256 _num) public {
        num = _num;  // 写入slot0
    }
}

contract A1 {
    address public owner;  // slot0
    uint256 public num;    // slot1

    function callB1(address _b, uint256 _num) public {
        _b.delegatecall(
            abi.encodeWithSignature("setNum(uint256)", _num)
        );
    }
}
```

### 攻击场景

我们想象一下，合约 A1 的 owner 原本是管理员的地址 `0x0123....`

黑客调用了 `callB1`，然后传入参数为自己算出来的一个地址上去。

结果 A1 使用了 delegatecall 执行了 B1 的逻辑。

这时候 **owner 就变成了黑客传上去的地址！管理员被覆盖了！**

这就是所谓的**存储冲突攻击 (Storage Collision)**。

在区块链历史上导致了数千万美元的经典漏洞，在早期 proxy 中很多事故。

---

## 第四部分：Context（上下文）

### 什么是 Context？

这时候好像忘了一个东西，不是讲 context 和 delegatecall 吗？

现在讲 context。

Context 是计算机科学里面最虚的但又是最重要的词。

在 solidity 和以太坊世界中，**Context 就是现场的所有环境**。

### 三要素

当你有一行代码被执行的时候，有三要素：

1. **谁执行的？**
2. **在哪执行的？**
3. **花了多少钱？**

这三个要素打包到一起，就叫做 **context**。

### 办公室场景比喻

想象一个公司大楼办公室的场景：

#### 1. 工牌（msg.sender）- Who

工作人员脖子上挂的工牌。

合约必须知道谁在调用它，不然没办法做权限控制，比如 `onlyOwner` 这种限制。

#### 2. 独立办公室（address(this) 和 storage）- Where

现在工作人员在哪个独立办公室里面。

如果说让工作人员把墙上的照片拿下来，那么他拿的就是这个办公室的照片。

在代码中，就是 `address(this)` 和他的存储变量（storage）。

#### 3. 钱包里的钱（msg.value）- Money

工作人员进办公室的时候手里拿着多少钱？

这个钱是准备给这个办公室的主人的。

### Call vs Delegatecall 的 Context 变换

我们回过头去看看 call 和 delegatecall 的区别，其实就是 context 的变换。

#### Call：

user 派 A 人员去 B 工作室办事

1. **初始状态**：
   - who：user
   - where：A 的房间
   - context：初始环境

2. **现在派 A 出去办事**：
   - who：A
   - where：B 的房间
   - context：**发生了转换**

在普通调用中，context 会随着链条不断变换，B 根本不知道 user 的存在，他只知道 A 过来办事了。

#### Delegatecall：

user 派 A 去做事，但是 A 借用了 B 的逻辑

- who：user
- where：A 的房间
- A 把 B 的逻辑拿过来了，偷师学会了技巧，然后 A 照着做

此时：

- context：**依旧是原始环境**

delegate 之所以叫委托，就是让代码在当下的 context 中运行，仿佛 B 根本没有，只是借用了 B 的技术。

这下明白了吧！

---

## 总结

### 1. 核心概念对比：Call vs Delegatecall

这是理解本次课程的关键，我们通过"谁在干活"和"改了谁的账本"来区分。

#### A. 普通调用 (Call) —— "外包采购"

- **场景**：老板 (User) 让员工 (A1) 去供应商 (B1) 那里买零件。
- **行为**：A1 带着钱去 B1 的地盘，B1 在自己的工厂生产，消耗 B1 的原料，记在 B1 的账本上。
- **Context 变化**：
  - 谁调用的 (`msg.sender`)：变成了 A1（对于 B1 来说，A1 是客户）。
  - 在哪运行 (`address(this)`)：在 B1 的地址。
  - 修改的数据 (Storage)：修改 B1 的状态（num 存在 B1 里）。
- **一句话总结**：去别人家里干活，改别人的数据。

#### B. 委托调用 (Delegatecall) —— "偷学技术 / 灵魂附体"

- **场景**：老板 (User) 让员工 (A1) 做零件，但 A1 不会做，于是 A1 把 B1 的"制作说明书（代码逻辑）"拿过来，在 A1 自己的工厂里，用 A1 的原料做。
- **行为**：逻辑代码是 B1 的，但执行环境全是 A1 的。
- **Context 保持**：
  - 谁调用的 (`msg.sender`)：依然是 User（老板）。A1 只是个躯壳，B1 感觉不到 A1 的存在，以为是 User 直接在操作。
  - 在哪运行 (`address(this)`)：在 A1 的地址。
  - 修改的数据 (Storage)：修改 A1 的状态（num 存在 A1 里）。
- **一句话总结**：借别人的脑子（逻辑），在自己家里干活，改自己的数据。

### 2. Solidity 终极心法：可升级合约原理

基于 delegatecall 的特性，我们推导出了区块链最著名的设计模式——**可升级合约 (Upgradable Contracts)**。

- **躯体 (Proxy 合约 - A1)**：负责保存数据（状态变量）。它永远不变，地址也不变。
- **灵魂 (Logic 合约 - B1)**：负责提供逻辑（函数代码）。
- **升级方法**：当需要升级功能时，只需要把 B1 换成 C1。A1 指向的逻辑地址变了，但 A1 里的数据（钱、用户资料）完美保留。

### 3. 高危雷区：存储槽冲突 (Storage Collision)

这是 delegatecall 最大的副作用，也是黑客的最爱。

- **原理**：EVM（以太坊虚拟机）不认识变量名（如 owner, num），它只认识**位置 (Slot)**。Slot 0, Slot 1, Slot 2...
- **灾难场景**：
  - A1 合约：Slot 0 是 owner (管理员地址)。
  - B1 合约：Slot 0 是 num (普通数字)。
  - **攻击**：当 A1 delegatecall B1 的 setNum 时，B1 的代码会无脑修改 Slot 0。结果 A1 的 owner 被覆盖成了 num 的值，管理员权限丢失！
- **防御**：在使用代理模式时，必须严格保证代理合约(A1) 和逻辑合约(B1) 的存储布局 (Storage Layout) 完全一致。

### 4. 什么是 Context (上下文)？

Context 是代码执行时的"现场环境"，由三大要素组成：

- **Who (工牌)**：`msg.sender`。谁在这一刻按下了按钮？
- **Where (办公室)**：`address(this)` 和 storage。现在在哪修改数据？
- **Money (资金)**：`msg.value`。这次操作带了多少钱？

Call 与 Delegatecall 的本质区别就在于对 Context 的处理：

- **Call**：切换 Context（换了房间，换了工牌）。
- **Delegatecall**：保留 Context（房间不变，工牌不变，只换了干活的方法）。

### 课后顺口溜（助记）

- **Call** 是出门办事，sender 变了，改别人家数据。
- **Delegatecall** 是在家练功，sender 没变，偷别人逻辑改自家数据。
- 槽位不对要人命，升级合约靠灵魂。
