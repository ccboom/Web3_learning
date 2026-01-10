# Solidity Library 深度解析

库是一个非常优雅的概念，它既能够帮我们通过代码复用（Code Reuse）让代码更加整洁，又能帮我们节省部署的成本（虽然有时候会让调用成本增加）。

---

## 第一部分：Library 到底是什么？

### 形象比喻

想象一下，主合约是一个工人，正在工厂（EVM）里生产零件。

**普通合约**：像是隔壁厂的工人，他有自己的工位（Storage），有自己的制造机器。你想让他帮忙做个零件，你需要把你的东西拿过去，然后他做完再送回来或者是暂时存放在他那里。

**Library**：像是一个灵魂，他没有自己的工位，也没有自己的制造机器。
- 但你在你的代码里使用时，他就会附身过来，然后在你的工位使用你的机器来做零件
- 当然，改变的东西都只在你的工位上

### Library 的重要限制

**Library 不能有状态变量**

比如 Library 不能定义 `uint x;`，不能存钱。

**为什么？**

按照我们上面类比的灵魂，你觉得灵魂可能拿得起真实的物品吗？

还有就是 Library 如果允许定义 `uint x;`，那么当合约 A 和合约 B 同时调用这个的时候 x 到底在哪？是存在 Library 的地址上？还是分别存在 A 和 B 的地址上？这会引起极大的混乱。

所以 Solidity 干脆规定，Library 禁止拥有状态变量。它只能处理别人传给它的数据。

---

## 第二部分：两种 Library 形态

在 Solidity 中，Library 虽然写起来都是 `library MyLib {}` 这种形态，但是在编译和部署的时候，根据函数的可见性，会有两种不同的形态生成：

1. **内嵌库（Embedded / Internal）**
2. **链接库（Linked / External）**

我们分别来看看这两种情况，比如我现在在 MathLib 库里面写了一个函数 `function add(uint a, uint b)`。

### 1. Internal 内嵌库 = 智能的复制粘贴

当你在 Library 写一个 `internal` 函数的时候，编译器就会把你当作已经存在的东西。

**编译过程**：

编译器看到你的主合约引用了 `MathLib.add`，而且他是 `internal` 的，那么它会直接把 `add` 函数完整的复制一份到主合约的编码也就是 bytecode 里面。

**结果**：

主合约肯定变大了啊，但是 Library 自己不用单独部署在区块链上。

**运行原理**：

EVM 运行的时候，只是在主合约内部做了一个跳转，到一个函数执行完再回来。相当于不认识字，查了一下字典，然后又重新写作业一样。

**优点**：
- 省事，不用管理库在链上的地址
- 调用非常便宜（内部跳转）

**缺点**：
- 如果很多合约都使用这个库的话，所有的合约都要加入这个库的代码，这样会造成冗余，浪费了空间

### 2. Public/External 链接库 = 委托调用（delegatecall）

**编译过程**：

编译器不会复制代码，而是在主合约里面留一个占位符（Placeholder）。

看起来像这样：`call the code at address ______`。

**部署过程**：

1. 首先你需要把 MathLib 部署到链上，获得一个地址，比如 `0x00125456...`
2. 然后在部署你的主合约之前，把这个地址写入到占位符里面去，这个过程叫做**链接（Linking）**

**运行原理**：

主合约运行到这里的时候，会发起一个 `delegatecall` 到 `0x00123456...`。

然后关键的是：MathLib 里面的代码会借到你主合约的上下文里面，修改的是你的状态，花掉的是你的 GAS。

**优点**：
- 代码可以重复使用，部署一次可以给多个合约一直使用
- 可用性非常高，节省主合约的大小，防止超过 24KB 限制

**缺点**：
- 调用起来的 GAS 费用比较高
- 部署流程复杂，需要先部署库，再部署主合约

---

## 第三部分：using for 语法糖

可能你听的云里雾里的，别担心，先往后看，最后回来看你就可以理解我在说什么，我们先基本了解一下，看到合约代码的时候你就会全明白了。

我们接着来讲一下 `using for`。

### 什么是 using for？

`using for` 就像是机器人升级之后的技能。

**如果我们没有 using for**：

想要计算 `a + b`，就要写 `Math.add(a, b)`，就是调用一个函数来处理两个值相加。

**有了之后**：

好像代码就会自己处理了。我们计算 `a + b` 就变成 `a.add(b)`。

### 本质

`using for` 的本质只是为了让代码写起来更顺手，本质还是没变的。

这个在编程中叫做**语法糖（Syntactic Sugar）**——虽然吃起来更甜，但是本质没变。

### 工作原理

新手弄不明白，只有一个参数，那第一个参数去哪里？

比如我们定义了一个函数：

```solidity
function add(uint256 a, uint256 b)
```

那么我现在有两个数 `num1` 和 `num2`。

我们使用 `using for` 就是：

```solidity
num1.add(num2)
```

这个时候，编译器在编译的时候就会悄悄把它变回去：

```solidity
add(num1, num2)
```

这下明白了吧，我们调用的主体 `num1` 自动跳到了第一个参数，括号里的参数自动放在后面接着。

---

## 第四部分：using for 的两种用法

### 1. 计算数值（uint, address, bool）

有些数值传入进来之后，你只需要计算他然后返回即可，不会改变原来的数值。

我们举个例子：

```solidity
// 第一个参数就是普通的 uint
function square(uint self) internal pure returns (uint) {
    return self * self; // 算出平方
}
```

这是一个库函数。

如果现在有：

```solidity
uint A = 2;
A.square();      // 这一行不会改变 A 的值，只会返回一个值
A = A.square();  // 这一行会改变 A 的值，因为 square 的返回值给了 A
```

就好像你把档案复印了一份，拿出去修改了，但是原件还是没变的，你必须把复印件替换原件，这样才能改变数据。

### 2. 计算顺便修改数值（struct, array, mapping）

这些引用类型，如果库函数第一个参数带了 `storage` 关键字，那他是真能修改数据。

我们再举个例子：

```solidity
struct User { uint score; }

// 注意：第一个参数带了 storage！
function levelUp(User storage self) internal {
    self.score += 10; // 直接改这一行
    // 不需要 return
}
```

那我们现在调用它：

```solidity
User storage i = users[msg.sender];
i.levelUp(); // 这次 i.score 真的变了，不用再写 i = i.levelUp()
```

这就好像把钥匙（storage 指针）给了装修队，他们进家里装修，装修的是你自己的家。等你回家一看，家里果然变样了。

明白了吧！

---

## 第五部分：Storage 的魔力

接下来我们要深入探究一下 storage 的魔力了。

这是 Library 中最强大的用法，也是你能不能写出高级合约的关键。

我们通常把第一个参数写为 `self`，模仿 Python 和 Rust。

看看下面的代码吧：

```solidity
struct Wallet {
    uint amount;
}

library WalletLib {
    // 注意这里使用了 storage
    function deposit(Wallet storage self, uint value) internal {
        self.amount += value;
    }
}

contract Bank {
    using WalletLib for Wallet;
    Wallet public mWallet;

    function addMoney() public {
        // 这里 mWallet 是 storage 调用，所以 mWallet.amount 会直接改变！
        mWallet.deposit(200);
    }
}
```

---

## 第六部分：链接库（External Library）

接下来我们讲：**链接库（External Library）**。

刚才我们讲的都是 `internal` 库，现在我们要真正进入深水区，讲一下链接库。

这通常用于大型项目，为了复用逻辑或者说绕过合约限制的大小（Spurious Dragon 硬分叉后限制为 24KB）。

### 工作流程

假设我们需要一个超级大的数学库 `BigMath`，它的所有合约都是 `public` 的。

工作流如下：

1. **部署库**：先部署 `BigMath`，得到一个地址 `0x00....`
2. **编译主合约**：编译 `MyContract`，此时字节码里全是占位符 `__BigMath___________________________`
3. **链接**：把 `0x000....` 填入占位符
4. **部署主合约**

### 反直觉的现象

这里有一个反直觉的现象我们需要讨论一下。

我们知道 Library 不能有状态变量（不能存钱，不能定义 `uint x;`）。

但是当 `MyContract` 通过 `delegatecall` 调用 `BigMath` 函数的时候，`BigMath` 的函数写了一句 `address(this)`。

**请问**：

1. 当代码运行在 `BigMath` 的逻辑里时，`address(this)` 返回的是库 `BigMath` 自己的地址，还是调用者 `MyContract` 的地址？
2. 如果库函数里有一句 `selfdestruct`（自毁），谁会被炸掉？是库，还是主合约？

**答案**：

1. 当然是主合约的，代码是库写的，但是它是附身在主合约上运行的，所以上下文全是主合约的
2. 炸毁的也是主合约，这就是著名的 **Parity 多重签名钱包黑客事件**的原理。黑客调用了一个公共库里的初始化函数，把自己变成了库的主人，然后调用库里的 `selfdestruct`。因为那个库是被主合约通过 `delegatecall` 调用的，结果整个钱包合约连带着被毁了，ETH 全部被锁死了。

---

## 第七部分：ABI 约束

最后一个我们要讲的，就是：**The ABI Constraint（ABI 约束）**。

我们来看下面的代码：

```solidity
library DataLib {
    // 这是一个 public 函数，意图让外部链接调用
    function updateScore(mapping(address => uint) storage balances, address user) public {
        balances[user] += 10;
    }
}
```

当你尝试编译这个代码的时候，编译器会直接报错。

### 为什么 mapping 不能公网传输？

**Public 规则约束**：

只要函数贴上了 `public` 或者 `external` 标签，EVM 就假设这个函数能接受来自外面的调用。这意味着传递给它的每一个参数，都必须能打包成 ABI Encoding，也就是一串二进制数据，塞进快递盒子里发出去。

**Mapping 本质**：

`mapping(address => uint)` 不是普通的列表或者数组，它像是一个漆黑的宇宙：
- 你只能通过坐标（Key）来找到星星（Value）
- 关键：你无法遍历它，因为它太大了，有 $2^{256}$ 个槽位，且你不知道哪里有数据，哪里是空的

既然无法遍历 Mapping，那他就无法打包，你没办法把整个宇宙装入盒子中。

所以你想要传递 `public` 里面的 `mapping` 时，他就会报警。

### 那为什么 internal 不受影响？

因为它只在内部调用，他处理东西只需要找到家里的东西就行了，不用外出去寻找。

---

## 总结

### 1. Library 的本质

- **定义**：Library 是一个没有状态的"灵魂"，它附身在主合约上运行，使用主合约的 Storage 和上下文。
- **核心限制**：Library 不能有状态变量（不能定义 `uint x;`），只能处理传入的数据。
- **目的**：代码复用、节省部署成本、绕过合约大小限制（24KB）。

### 2. 两种 Library 形态对比

| 类型 | Internal（内嵌库） | External（链接库） |
| :--- | :--- | :--- |
| **编译方式** | 代码直接复制到主合约 bytecode | 留占位符，需要链接地址 |
| **部署方式** | 不需要单独部署 | 需要先部署库，再链接主合约 |
| **调用方式** | 内部跳转（JUMP） | delegatecall |
| **Gas 成本** | 便宜 | 较贵 |
| **优点** | 简单、快速 | 代码复用、节省空间 |
| **缺点** | 代码冗余 | 部署复杂、调用成本高 |

### 3. using for 语法糖

- **本质**：让代码更优雅，`a.add(b)` 实际上是 `add(a, b)`。
- **工作原理**：调用主体自动成为第一个参数，括号内参数依次排列。
- **两种用法**：
  - **值类型**（uint, address, bool）：不修改原值，需要手动赋值 `A = A.square()`
  - **引用类型**（struct, array, mapping）：如果第一个参数是 `storage`，可以直接修改原值 `i.levelUp()`

### 4. Storage 的魔力

- 当库函数的第一个参数是 `storage` 类型时，它可以直接修改主合约的状态。
- 这是 Library 最强大的特性，也是写高级合约的关键。
- 通常把第一个参数命名为 `self`，模仿 Python 和 Rust 的风格。

### 5. delegatecall 的陷阱

- 通过 `delegatecall` 调用库时，`address(this)` 返回的是主合约地址，不是库地址。
- 如果库中有 `selfdestruct`，会炸毁主合约，不是库本身。
- **历史教训**：Parity 多重签名钱包黑客事件，黑客通过调用库的 `selfdestruct` 摧毁了主合约，导致大量 ETH 被永久锁死。

### 6. ABI 约束

- `public`/`external` 函数的参数必须能被 ABI 编码。
- `mapping` 无法被 ABI 编码（无法遍历、无法打包），所以不能作为 `public` 函数的参数。
- `internal` 函数不受此限制，因为它只在内部调用，直接访问 storage 指针。

### 7. 最佳实践

- **小型项目**：使用 `internal` 库，简单快速。
- **大型项目**：使用 `external` 库，避免合约超过 24KB 限制。
- **状态修改**：善用 `storage` 参数，让库函数直接修改主合约状态。
- **安全第一**：避免在库中使用 `selfdestruct` 等危险操作。

### 一句话总结

Library 是 Solidity 中的"无状态灵魂"，通过 `internal`（复制粘贴）或 `external`（delegatecall）两种方式附身在主合约上运行，配合 `using for` 语法糖和 `storage` 参数，实现优雅的代码复用和状态修改，但需要注意 ABI 约束和 delegatecall 的安全陷阱。
