# Foundry 单元测试入门

欢迎来到今天的学习，今天的学习非常关键，使用 Forge 编写并运行你的第一个单元测试。

在智能合约开发中，测试的重要性是 Web2 开发的 100 倍。因为合约一旦部署，代码不可改，资金不可追回。Foundry 允许你用 Solidity 编写测试，这意味着你不用在几种语言之间来回切换，非常丝滑。

---

## 第一部分：创建项目

昨天我们安装了 Foundry 工具链，今天的第一步，就是用它来生成一个标准的开发环境。

打开你的 vscode 中的终端，输入以下命令来创建一个新项目，我们就叫他 `hello_foundry` 好了：

```bash
forge init hello_foundry
```

执行成功后，进入这个新文件夹：

```bash
cd hello_foundry
```

### 在 WSL 中打开项目

现在你的 vscode 还是没有文件夹打开啊，不要急，按照以下方法来操作，就可以。

**用 Remote - WSL 扩展（推荐）**：

1. 在 Windows 里打开 VS Code。
2. 安装扩展：Remote - WSL（扩展名通常是 `ms-vscode-remote.remote-wsl`）。
3. 按 `Ctrl + Shift + P` 打开命令面板，输入并选择：`WSL: Connect to WSL`
4. 连接进去后，再按 `Ctrl + K` 然后 `Ctrl + O`（或菜单 File → Open Folder），选择你 WSL 里的路径（例如 `/home/hello_foundry`）打开即可。

---

## 第二部分：理解合约

现在我们看看 `src` 文件夹和 `test` 文件夹里都生成了新的文件，我们现在的新目标是读懂合约。

来看 `src/Counter.sol`：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

contract Counter {
    uint256 public number;

    function setNumber(uint256 newNumber) public {
        number = newNumber;
    }

    function increment() public {
        number++;
    }
}
```

你可以把他想象为工厂里的一台计数器机器。

只有三个简单的部件：

1. **`uint256 public number`**：这是机器的显示屏，它记录当前的数字
2. **`function setNumber(uint256 newNumber)`**：这是设置按钮，你输入几，屏幕就变几
3. **`function increment()`**：这是加一按钮，按一下，屏幕就在原来的基础上加一

### 思考题

OK，那我们假设这个是一个新机器，number 显示为 0。

1. 我们按了一次 `increment()`
2. 接着，我们又设置了 `setNumber(10)`

屏幕最终会是多少？

**答案是 10**。虽然之前执行了 `increment` 让数字变了，但 `setNumber(10)` 是霸道的，它会直接覆盖之前的结果，不管之前是多少。

---

## 第三部分：搭建测试骨架

找到 `test` 文件夹下的 `Counter.t.sol`，把下面的覆盖进去，我们重新书写测试用例：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

// 1. 拿工具箱：引入 Foundry 的标准测试库
import "forge-std/Test.sol";
// 2. 拿图纸：引入我们要测试的目标合约
import "../src/Counter.sol";

// 3. 定义测试合约：继承 Test 工具箱
contract CounterTest is Test {
    
}
```

### 关键问题：为什么要 import？

请看第二部分 `import "../src/Counter.sol"`，如果我们忘了写这一行或者写错路径，会有什么样的后果？

想象一下，`Counter.t.sol` 是你的实验室，而 `Counter.sol` 是你要研究的大熊猫。

如果你不写 `import "../src/Counter.sol";`，就等于你进到了实验室，却没有把大熊猫带进来。

这时候，如果你在代码里写"让大熊猫吃竹子"，电脑就会一脸懵地问你："谁是大熊猫？我不认识啊！"

**后果就是**：编译器会直接报错（Error），告诉你找不到 `Counter` 这个名字，程序根本跑不起来。

现在我们知道引入有多重要了，我们给骨架里面填肉。

---

## 第四部分：setUp 函数

我们要写的第一个核心函数是 `setUp()`。它是测试的准备阶段。

请把下面的代码填入 `CounterTest` 合约的大括号 `{ ... }` 里面：

```solidity
Counter public counter; // 1. 声明一个变量，用来存放我们的"机器"

function setUp() public {
    // 2. 制造一台全新的机器
    counter = new Counter();
    counter.setNumber(0);
}
```

### 关键词：new

`counter = new Counter();` 这行代码的意思是："帮我制造（实例化）一个新的 Counter 合约"。

### 测试隔离 (Test Isolation)

在 `setUp()` 函数里，我们先把 `number` 设置为了 0。假设我们在后面写了一个测试函数 `testIncrement`，Foundry 运行它的时候，会先自动运行一遍 `setUp()`。

如果我又写了第二个测试函数 `testSetNumber`，Foundry 在运行这第二个测试之前，还会再自动运行一遍 `setUp()` 吗？

**答案是会**。每次运行一个新的测试函数之前，`setUp()` 都会重新运行一次。

你可以这样理解：

- `CounterTest` 是一个酒店房间。
- `setUp()` 是保洁阿姨。
- `testIncrement` 是客人 A。
- `testSetNumber` 是客人 B。

为了保证卫生，客人 A 退房后，保洁阿姨必须把房间打扫回最初的状态（执行 `setUp`），然后客人 B 才能住进去。这样，客人 A 留下的垃圾（修改的数据）绝对不会影响到客人 B。

这在测试里叫**"测试隔离"(Test Isolation)**。

---

## 第五部分：编写第一个测试用例

现在的状态是：

- 保洁阿姨 (setUp) 已经把机器造好了，初始 `number` 是 0。
- 我们现在要请第一位客人进来测试"加一"功能。

请把下面的代码复制到 `CounterTest` 合约里（跟在 `setUp` 函数后面）：

```solidity
function testIncrement() public {
    // 1. 执行操作：按下"加一"按钮
    counter.increment();

    // 2. 验证结果：断言 (Assert)
    // assertEq 的意思是：我敢打赌，左边的东西等于右边的东西。
    assertEq(counter.number(), 1);
}
```

### 断言 (Assert)

这里出现了一个新词：`assertEq`。它是 assert Equal（断言相等）的缩写。

假如我在代码里写了：`assertEq(counter.number(), 5);`

意思是"我断言现在的数字等于 5"。但实际上，刚才执行完 `increment()` 后，数字应该是 1。

Foundry 会报错（Fail），并告诉你："期望是 5，但实际是 1"。

测试框架是非常"铁面无私"的。`assertEq` 的作用就是：如果结果不是我想要的，必须立刻报警！这正是我们写测试的目的——防止错误的逻辑混入我们的代码。

### 运行测试

请在你的终端（Terminal）里输入这行简单的命令：

```bash
forge test
```

运行之后，你会看到几行输出像下面的：

```
[PASS] testIncrement() (gas: 28760)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 488.06µs (70.98µs CPU time)
```

这说明你的代码逻辑完全正确：

- Foundry 启动了测试环境。
- 运行了 `setUp()`，把计数器清零。
- 运行了 `testIncrement()`，执行了加 1 操作。
- 最后，它验证了结果确实是 1，所以给了你一个 PASS。

---

## 第六部分：编写第二个测试用例

既然"加一"按钮没问题，现在我们要测试机器的第二个功能：`setNumber`（直接设置数字）。

请回到你的 `Counter.t.sol` 文件，在 `testIncrement` 函数的后面（但是在整个合约最后的 `}` 之前），加入下面这段新代码：

```solidity
function testSetNumber() public {
    // 1. 我们尝试把数字直接设为 77
    counter.setNumber(77);
    
    // 2. 验证：屏幕上的数字应该是 77
    assertEq(counter.number(), 77);
}
```

保存文件后，再次在终端运行：

```bash
forge test
```

这一次，你应该会看到两行 `[PASS]`。

### Gas 消耗对比

现在能告诉我 `testSetNumber` 消耗了多少 Gas 吗？

```
[PASS] testIncrement() (gas: 28760)
[PASS] testSetNumber() (gas: 29180)
```

让我们来做一个有趣的对比：

- `testIncrement`：消耗了 28,760 Gas
- `testSetNumber`：消耗了 29,180 Gas

### 为什么 setNumber(77) 会比 increment() 稍微贵那么一点？

区块链（EVM）的世界里，只要是修改存储（Storage），写下数字 1 和写下数字 77 消耗的能量（Gas）其实是完全一样的。

**导致 Gas 变高的真正凶手，其实是运费——也就是我们要发送给合约的数据包大小。**

请再次看看 `src/Counter.sol` 里这两个函数的定义，注意看括号 `()` 里面的内容：

```solidity
function increment()

function setNumber(uint256 newNumber)
```

我们可以这样形象地理解：

- **调用 `increment()`**：就像寄了一封空信封。你只告诉合约"去执行 increment 吧"，信封里没有别的东西。
- **调用 `setNumber(77)`**：就像寄了一封塞了钱的信封。你不仅要告诉它"去执行 setNumber"，还要在信封里额外塞进一张写着 77 的纸条。

在区块链的世界里，每一个字节的数据传输都需要付费。那个额外的数字 77（参数）占用了空间，所以你需要为这部分"重量"多付一点点 Gas 运费。

---

## 第七部分：体验"失败"

现在你已经看腻了绿色的 `[PASS]`，但在实际开发中，我们 90% 的时间其实是在和红色的报错打交道。

为了让你以后遇到报错时不慌张，我们现在故意搞破坏，写一个错误的测试，看看 Forge 会怎么提示我们。

请修改 `testIncrement` 函数，把它的断言改成错误的。比如：刚才 `increment` 之后明明应该是 1，我们非要说它是 100。

```solidity
function testIncrement() public {
    counter.increment();
    // 故意改成 100，看看会发生什么？
    assertEq(counter.number(), 100);
}
```

修改好后，请再次运行：

```bash
forge test
```

结果应该是：

```
[FAIL: assertion failed: 1 != 100] testIncrement()
```

### 解读报错信息

这里面藏着三个关键线索：

1. **FAIL**：测试失败了，这点很直观。

2. **assertion failed**：这是最关键的。它告诉你是因为 `assertEq` 断言不成立才挂掉的。

3. **1 != 100**：这是案发现场。
   - **左边的 1**：这是真实发生的事情（Actual）。你的 `counter.number()` 实际上是 1。
   - **右边的 100**：这是你的幻想（Expected）。你在代码里写了期望它是 100。
   - **!=**：意思是"它俩不相等啊大哥！"。

### 专家经验

在写测试时，养成一个好习惯：把"实际值"放在左边，把"期望值"放在右边。即 `assertEq(实际计算结果, 预期结果)`。这样看报错时，你永远知道左边那个数字是合约真实跑出来的结果。
