# Foundry 测试实战：多签钱包

欢迎来到测试环节，未经严格测试的合约代码，不仅是一堆垃圾，更有可能是一个定时炸弹。

特别是多签钱包，它是管理巨额资金的金库，任何一个逻辑漏洞都可能导致灾难。

我们使用的是 Foundry 框架，在 W4 中已经配置好了，如果没配置可以回到 W4 查看一下。

---

## 第一部分：准备多签钱包合约

为了配合我们测试，请在 `src` 文件夹下创建一个 `MultiSig.sol` 的文件，并粘贴以下代码：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract MultiSig {
    // 事件：记录资金变动和交易状态变化
    event Deposit(address indexed sender, uint amount, uint balance);
    event SubmitTransaction(address indexed owner, uint indexed txIndex, address indexed to, uint value, bytes data);
    event ConfirmTransaction(address indexed owner, uint indexed txIndex);
    event RevokeConfirmation(address indexed owner, uint indexed txIndex);
    event ExecuteTransaction(address indexed owner, uint indexed txIndex);

    // 状态变量
    address[] public owners;
    mapping(address => bool) public isOwner;
    uint public numConfirmationsRequired;

    // 交易结构体
    struct Transaction {
        address to;
        uint value;
        bytes data;
        bool executed;
        uint numConfirmations;
    }

    // 存储交易确认状态：txIndex => owner => bool
    mapping(uint => mapping(address => bool)) public isConfirmed;
    Transaction[] public transactions;

    // 修饰符：权限控制
    modifier onlyOwner() {
        require(isOwner[msg.sender], "not owner");
        _;
    }

    modifier txExists(uint _txIndex) {
        require(_txIndex < transactions.length, "tx does not exist");
        _;
    }

    modifier notExecuted(uint _txIndex) {
        require(!transactions[_txIndex].executed, "tx already executed");
        _;
    }

    modifier notConfirmed(uint _txIndex) {
        require(!isConfirmed[_txIndex][msg.sender], "tx already confirmed");
        _;
    }

    // 构造函数：初始化 owners 和确认数
    constructor(address[] memory _owners, uint _numConfirmationsRequired) {
        require(_owners.length > 0, "owners required");
        require(
            _numConfirmationsRequired > 0 &&
                _numConfirmationsRequired <= _owners.length,
            "invalid number of required confirmations"
        );

        for (uint i = 0; i < _owners.length; i++) {
            address owner = _owners[i];

            require(owner != address(0), "invalid owner");
            require(!isOwner[owner], "owner not unique");

            isOwner[owner] = true;
            owners.push(owner);
        }

        numConfirmationsRequired = _numConfirmationsRequired;
    }

    // 允许合约接收 ETH
    receive() external payable {
        emit Deposit(msg.sender, msg.value, address(this).balance);
    }

    // 提交交易
    function submitTransaction(
        address _to,
        uint _value,
        bytes memory _data
    ) public onlyOwner {
        uint txIndex = transactions.length;

        transactions.push(
            Transaction({
                to: _to,
                value: _value,
                data: _data,
                executed: false,
                numConfirmations: 0
            })
        );

        emit SubmitTransaction(msg.sender, txIndex, _to, _value, _data);
    }

    // 确认交易
    function confirmTransaction(
        uint _txIndex
    ) public onlyOwner txExists(_txIndex) notExecuted(_txIndex) notConfirmed(_txIndex) {
        Transaction storage transaction = transactions[_txIndex];
        transaction.numConfirmations += 1;
        isConfirmed[_txIndex][msg.sender] = true;

        emit ConfirmTransaction(msg.sender, _txIndex);
    }

    // 执行交易
    function executeTransaction(
        uint _txIndex
    ) public onlyOwner txExists(_txIndex) notExecuted(_txIndex) {
        Transaction storage transaction = transactions[_txIndex];

        require(
            transaction.numConfirmations >= numConfirmationsRequired,
            "cannot execute tx"
        );

        transaction.executed = true;

        (bool success, ) = transaction.to.call{value: transaction.value}(
            transaction.data
        );
        require(success, "tx failed");

        emit ExecuteTransaction(msg.sender, _txIndex);
    }

    // 撤销确认
    function revokeConfirmation(
        uint _txIndex
    ) public onlyOwner txExists(_txIndex) notExecuted(_txIndex) {
        Transaction storage transaction = transactions[_txIndex];

        require(isConfirmed[_txIndex][msg.sender], "tx not confirmed");

        transaction.numConfirmations -= 1;
        isConfirmed[_txIndex][msg.sender] = false;

        emit RevokeConfirmation(msg.sender, _txIndex);
    }

    // Getter 函数
    function getOwners() public view returns (address[] memory) {
        return owners;
    }

    function getTransactionCount() public view returns (uint) {
        return transactions.length;
    }

    function getTransaction(
        uint _txIndex
    )
        public
        view
        returns (
            address to,
            uint value,
            bytes memory data,
            bool executed,
            uint numConfirmations
        )
    {
        Transaction storage transaction = transactions[_txIndex];

        return (
            transaction.to,
            transaction.value,
            transaction.data,
            transaction.executed,
            transaction.numConfirmations
        );
    }
}
```

---

## 第二部分：搭建测试舞台（setUp）

在开始测试之前，我们需要搭建一个舞台，也就是新建一个文件，我们把它命名为 `MultiSig.t.sol`。

每一个测试合约都需要一个 `setUp()` 函数，你可以把它想象为归零操作，无论运行多少个用例，setUp 会在每个用例之前运行一次，确保每个测试都在干净的环境中进行。

### 核心工作

为了测试多签钱包，我们需要在 setUp 中模拟现实世界的情况。

假设我们要测试一个经典的场景，3 个管理员，需要 2 人确认（2-of-3）的多签钱包场景。

在 setUp 函数中，我们需要做哪两件最核心的工作，才能让后面的测试跑起来？

当然是设置三个管理员进去，然后门槛设置为 2。

### makeAddr 工具

我们使用的 Foundry 有个非常实用的工具，叫做 `makeAddr("name")`。

他会根据你给的名字生成一个地址，然后报错的时候，直接会显示名字而不是 `0x0..` 地址，非常实用。

接下来请看下面的代码内容：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import "../src/MultiSig.sol";

contract MultiSigTest is Test {
    // 1. 定义舞台上的"道具"和"演员"
    MultiSig public wallet;
    address public user1;
    address public user2;
    address public user3;
  
    function setUp() public {
        // TODO 1: 使用 makeAddr 给 user1, user2, user3 赋值
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");
        
        // ... 中间数组部分我们之前说过了 ...
        address[] memory owners = new address[](3);
        owners[0] = user1;
        owners[1] = user2;
        owners[2] = user3;

        // TODO 3: 部署 MultiSig 合约，传入 owners 数组和 2 作为确认数
        wallet = new MultiSig(owners, 2);
    }
}
```

### 第一个测试用例

写好 setUp，我们就可以编写我们第一个测试用例了。

我们要检查一下，是不是真的有三个管理员，请使用以下代码，放入上面的合约中：

```solidity
// === 我们的第一个测试 ===
function test_InitialState() public {
    // 检查钱包是否真的记录了3个主人
    uint256 count = wallet.getOwners().length;
    assertEq(count, 3); 
}
```

现在请在终端运行：

```bash
forge test
```

是不是会显示绿色的 `[PASS]`？

说明我们的舞台已经搭建完成，可以一步一步的接着干了。

---

## 第三部分：快乐路径测试

所谓的快乐路径，就是测试的一切都按照预想的形式发生，比如提交交易 → 确认交易 → 执行成功，这是最基础的功能验证。

### 测试 1：提交交易

我们第一步测试的是提交交易。

看一眼 `MultiSig.sol`，就会发现 `submitTransaction` 函数有一个修饰符 `onlyOwner`。

如果直接调用 `submitTransaction`，那调用者其实是 `MultiSigTest` 这个合约本身，它并不是 owners 之一，所以一定会报错。

#### vm.prank() 工具

为了解决这个问题，Foundry 有一个强大的函数 `vm.prank(address)`。

它可以假装下一个函数是由 address 发起的！很厉害的函数，但是，帅不过三秒，它只能对紧接着的下一行生效，再下面就不行了。

请在 `MultiSigTest` 合约中添加一个新的测试函数 `test_SubmitTransaction`：

```solidity
function test_SubmitTransaction() public {
    // 1. 定义交易参数
    address to = user2;
    uint256 value = 0;
    bytes memory data = "";

    // 2. 验证初始状态：交易数量应该是 0
    assertEq(wallet.getTransactionCount(), 0);

    // TODO: 使用 vm.prank 模拟 user1
    vm.prank(user1);

    // TODO: 调用 wallet.submitTransaction，传入上面的参数
    wallet.submitTransaction(to, value, data);

    // 3. 验证结束状态：交易数量应该是 1
    assertEq(wallet.getTransactionCount(), 1);
}
```

如果你写的也差不多，那么我们就来测试一下。

直接运行：

```bash
forge test --mt test_SubmitTransaction
```

`--mt` 是 match test 的缩写，只允许匹配名字的测试，这样速度更快。

如果看到的是绿色的 pass，那我们就进行下一步。

#### 深度检查

虽然 `getTransactionCount` 能看出来添加了一个，但是我们不知道添加的内容和我们发送的交易是不是一样的，我们也得判断一下。

我们把刚刚的 0 号交易取出来，逐个检查：

1. 目标地址 `txTo` 是否等于 `user2`
2. 执行状态 `txExecuted` 是否为 `false`

请在函数最后继续添加以下代码：

```solidity
// 4. 深度检查：验证数据完整性
// 获取第 0 号交易的详细信息
(
    address txTo, 
    uint256 txValue, 
    bytes memory txData, 
    bool txExecuted, 
    uint256 txConfs
) = wallet.getTransaction(0);

// TODO 1: 验证 txTo 是否等于 user2
assertEq(txTo, user2);

// TODO 2: 验证 txExecuted 是否等于 false
assertEq(txExecuted, false);
```

然后，再次运行：

```bash
forge test --mt test_SubmitTransaction
```

如果看到 pass，恭喜你已经攻克了这一难关。

### 测试 2：签名与执行

现在的状态是：交易池有一个交易，但是还缺少人签名和执行，目前签名人数为 0。

我们是一个 2-of-3 的钱包，意味着必须最少两人签名才能执行交易。

我们需要创建一个新函数 `test_ExecuteTransaction`。

在 Foundry 中，`vm.prank()` 只对下一行有效，所以我们需要调用身份的时候，必须每次都写上。

代码前半段如下，先让 User1 提交一笔交易，然后模拟 User1 和 User2 分别调用 `wallet.confirmTransaction(0)`：

```solidity
function test_ExecuteTransaction() public {
    // 1. 先提交一笔交易 (我们可以直接复用之前的逻辑，或者简化一点)
    vm.prank(user1);
    wallet.submitTransaction(user2, 1 ether, ""); // 这次我们需要发点钱，设为 1 ether

    // TODO: 模拟 user1 确认第 0 号交易
    vm.prank(user1);
    wallet.confirmTransaction(0);
    
    // TODO: 模拟 user2 确认第 0 号交易
    vm.prank(user2);
    wallet.confirmTransaction(0);
    
    // 检查确认数是否变为了 2
    (,,,, uint256 confs) = wallet.getTransaction(0);
    assertEq(confs, 2);
}
```

#### 准备资金

我们已经拿到两个确认了，按道理来说现在可以执行交易了。

但是等会，我们发送的是 1 ETH，我们的多签钱包现在真的有那么多钱吗？我们创建的时候里面也没资金啊。

所以，我们又需要借用 Foundry 里面的工具 `deal(address, uint256)`。

它可以修改任意账户的余额为 uint256 数值。

还有最后一项，执行之后再检查检查，是不是真的到了 user2 手中了：

```solidity
// ... (你刚才写的代码) ...
assertEq(confs, 2);

// === 准备执行 ===

// TODO 1: 使用 deal 给 wallet 合约充值 10 ether
// 语法提示: deal(address(wallet), 10 ether);
deal(address(wallet), 10 ether);

// TODO 2: 模拟 user1 (必须是 owner 才能执行) 调用 executeTransaction(0)
// 别忘了 vm.prank(...)
vm.prank(user1);
wallet.executeTransaction(0);

// === 验证结果 ===
// 验证交易状态是否变为 executed = true
(,,, bool executed,) = wallet.getTransaction(0);
assertEq(executed, true);

// TODO 3: 验证 user2 的余额是否变成了 1 ether (原来是 0)
// 语法提示: assertEq(user2.balance, ...);
assertEq(user2.balance, 1 ether);
```

修改完成之后再次运行：

```bash
forge test --mt test_ExecuteTransaction
```

如果是绿色的 PASS 那么恭喜你又更进了一步！

---

## 第四部分：悲伤路径测试

接下来我们的测试不乐观了，我们走悲伤路径。

在黑客眼里，他们只会走悲伤路径，他们会尝试各种不合规的操作看看能不能达到目的，我们的任务就是确保合约在面对这些操作时，能够果断拒绝。

### vm.expectRevert() 工具

我们需要使用的核心工具是 `vm.expectRevert()`。

它的作用是：**预计下一行会报错**。

- 如果下一行真报错了，判断正确，显示绿色 Pass
- 如果下一行没报错，判断错误，显示红色 failed

### 测试 1：确认数不足

我们模拟一个场景，需要两个人签名才能执行交易，但是现在只有一个人签名了，它还是决定执行交易，那么这时候合约应该报错：`cannot execute tx`。

请补全下面的函数：

```solidity
function test_RevertWhen_NotEnoughConfirmations() public {
    // 1. 准备工作：User1 提交并确认了一笔交易
    vm.prank(user1);
    wallet.submitTransaction(user2, 1 ether, "");
    
    vm.prank(user1);
    wallet.confirmTransaction(0);

    // 现在确认数只有 1 (需要 2)

    // TODO: 告诉 Foundry 下一步操作应该报错，错误信息是 "cannot execute tx"
    // 语法提示: vm.expectRevert(bytes("错误信息"));
    vm.expectRevert(bytes("cannot execute tx"));

    // TODO: 模拟 User1 强行调用 executeTransaction(0)
    vm.prank(user1);
    wallet.executeTransaction(0);
}
```

这时候我们运行一下测试命令：

```bash
forge test --mt test_RevertWhen_NotEnoughConfirmations
```

如果合约报错了，那么测试就会通过，因为预言应验了。

如果合约执行成功了，反而会显示失败，这就是异常测试的精髓。

### 测试 2：非管理员提交

既然我们在做悲伤测试，我们不能跳过一个最经典的场景：

比如一个路人（hacker）想要调用提交交易，合约应该直接拒绝，并抛出 `not owner` 错误。

请补全这个新函数：

1. 创建一个不认识的人 hacker
2. 用 hacker 来提交交易
3. 使用 `vm.expectRevert` 来判断它会失败，错误信息是 `not owner`

```solidity
function test_RevertWhen_NonOwnerSubmits() public {
    // 1. 创建一个黑客地址
    address hacker = makeAddr("hacker");

    // 2. 预言会报错 "not owner"
    vm.expectRevert(bytes("not owner"));

    // 3. 模拟黑客身份 (vm.prank) 并提交交易
    vm.prank(hacker);
    wallet.submitTransaction(hacker, 0, "");
}
```

这一步非常关键，只有管理员才能操作资金，这个测试确保了任何旁人想要动歪心思都不行，会被合约拒绝。

请运行一下这个测试用例，看看能不能通过：

```bash
forge test --mt test_RevertWhen_NonOwnerSubmits
```

如果通过了，我们要来到我们今天的最终章了。

---

## 第五部分：模糊测试（Fuzz Testing）

### 让混乱降临

到目前为止，我们都是使用硬编码：

- 转账金额为 1 ETH
- 接收人是 user2

但在真实世界里，怎么可能每次都这么正确。

用户可能会输入各种各样的数据，万一用户输入转账 0.000000000001 ETH 或者一个巨大的数值，那合约又得炸。

单元测试只能证明**你认为会发生的事**是对的，而模糊测试会生成**几百个随机输入**，去寻找你意想不到的情况。

### Foundry 的模糊测试

在 Foundry 中，你不需要特意配置，只要修改函数参数即可。

在普通 solidity 测试中，函数的参数是由调用者发过去的。

但在 Foundry 的模糊测试中，你只要把参数写在括号内，Foundry 就会自动变成那个调用者，帮我们疯狂填入各种随机数据。

我们想测试的是给任意地址转任意金额。

查看以下代码：

```solidity
function testFuzz_SubmitTransaction(address _to, uint256 _value) public {
    // 1. 还是要假装是 owner (比如 user1)
    vm.prank(user1);

    // TODO: 调用 submitTransaction，但是使用传入的随机参数 _to 和 _value
    // 数据 data 依然可以留空 ""
    wallet.submitTransaction(_to, _value, "");

    // 2. 验证保存的数据是否和随机生成的参数一致
    (address txTo, uint256 txValue, , , ) = wallet.getTransaction(0);

    // TODO: 写两个断言 (assertEq)
    // 检查 txTo 是否等于 _to
    // 检查 txValue 是否等于 _value
    assertEq(txTo, _to);
    assertEq(txValue, _value);
}
```

把下面代码放入其中，然后运行测试命令：

```bash
forge test --mt testFuzz_SubmitTransaction
```

### 理解输出

运行完之后，仔细观察终端输出的数字。

在 `[PASS]` 之外，你应该还能看到类似 `(runs: 256, μ: ...)`

- **runs**：代表着 Foundry 是一个不知疲倦的机器人，在一瞬间帮你跑完了 256 组完全不同的参数
- **μ (Mu)**：这是希腊字母，代表"平均值"。也就是这 256 次测试中，平均每次消耗了多少 Gas。
- **~ (Tilde)**：这个波浪号代表"中位数"。

---

## 总结

### 1. 测试的重要性

- 未经测试的合约是定时炸弹，特别是多签钱包这种管理巨额资金的合约。
- Foundry 提供了强大的测试框架，让我们能够全面测试合约的各种场景。

### 2. setUp() 函数

- 每个测试用例运行前都会执行 `setUp()`，确保测试环境干净。
- 使用 `makeAddr("name")` 创建测试地址，报错时显示名字而不是地址。

### 3. 快乐路径测试

测试正常流程是否按预期工作：

- **test_InitialState**：验证初始化状态
- **test_SubmitTransaction**：验证提交交易功能
- **test_ExecuteTransaction**：验证完整的提交→确认→执行流程

### 4. 关键测试工具

| 工具 | 作用 | 示例 |
| :--- | :--- | :--- |
| `vm.prank(address)` | 模拟某个地址发起下一次调用 | `vm.prank(user1);` |
| `deal(address, uint)` | 给某个地址充值 ETH | `deal(address(wallet), 10 ether);` |
| `assertEq(a, b)` | 断言两个值相等 | `assertEq(count, 3);` |
| `vm.expectRevert(bytes)` | 预期下一行会报错 | `vm.expectRevert(bytes("not owner"));` |

### 5. 悲伤路径测试

测试异常情况是否被正确拒绝：

- **test_RevertWhen_NotEnoughConfirmations**：确认数不足时应该拒绝执行
- **test_RevertWhen_NonOwnerSubmits**：非管理员应该无法提交交易

### 6. 模糊测试（Fuzz Testing）

- 通过在函数参数中声明变量，Foundry 会自动生成随机输入进行测试。
- 默认运行 256 次，可以发现意想不到的边界情况。
- 函数命名以 `testFuzz_` 开头。

### 7. 测试命令

```bash
# 运行所有测试
forge test

# 运行特定测试（match test）
forge test --mt test_SubmitTransaction

# 查看详细输出
forge test -vv

# 查看 gas 报告
forge test --gas-report
```

### 一句话总结

通过 Foundry 的 `setUp()`、`vm.prank()`、`vm.expectRevert()` 和模糊测试等工具，我们可以全面测试多签钱包的快乐路径（正常流程）和悲伤路径（异常情况），确保合约在各种场景下都能正确运行或正确拒绝，从而避免资金损失。


### 完整代码
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import "../src/MultiSig.sol";

contract MultiSigTest is Test {
    MultiSig public wallet;
    address public user1;
    address public user2;
    address public user3;

    function setUp() public {
        // 1. 初始化用户
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        user3 = makeAddr("user3");

        // 2. 准备参数
        address[] memory owners = new address[](3);
        owners[0] = user1;
        owners[1] = user2;
        owners[2] = user3;

        // 3. 部署合约 (2/3 多签)
        wallet = new MultiSig(owners, 2);
    }

    // === 我们的第一个测试 ===
    function test_InitialState() public {
        // 检查钱包是否真的记录了3个主人
        uint256 count = wallet.getOwners().length;
        assertEq(count, 3); 
    }

    // === 我们的第二个测试 ===
    function test_SubmitTransaction() public {
        // 1. 定义交易参数
        address to = user2;
        uint256 value = 0;
        bytes memory data = "";

        // 2. 验证初始状态：交易数量应该是 0
        assertEq(wallet.getTransactionCount(), 0);

        // TODO: 使用 vm.prank 模拟 user1
        vm.prank(user1);

        // TODO: 调用 wallet.submitTransaction，传入上面的参数
        wallet.submitTransaction(to,value,data);

        // 3. 验证结束状态：交易数量应该是 1
        assertEq(wallet.getTransactionCount(), 1);

        // 4. 深度检查：验证数据完整性
        // 获取第 0 号交易的详细信息
        (
            address txTo, 
            uint256 txValue, 
            bytes memory txData, 
            bool txExecuted, 
            uint256 txConfs
        ) = wallet.getTransaction(0);

        // TODO 1: 验证 txTo 是否等于 user2
        assertEq(txTo, user2);

        // TODO 2: 验证 txExecuted 是否等于 false
        assertEq(txExecuted, false);
    }


    function test_ExecuteTransaction() public {
        // 1. 先提交一笔交易 (我们可以直接复用之前的逻辑，或者简化一点)
        vm.prank(user1);
        wallet.submitTransaction(user2, 1 ether, ""); // 这次我们需要发点钱，设为 1 ether

        // TODO: 模拟 user1 确认第 0 号交易
        vm.prank(user1);
        wallet.confirmTransaction(0);
        // TODO: 模拟 user2 确认第 0 号交易
        vm.prank(user2);
        wallet.confirmTransaction(0);
        
        // 检查确认数是否变为了 2
        (,,,, uint256 confs) = wallet.getTransaction(0);
        assertEq(confs, 2);

        // === 准备执行 ===

        // TODO 1: 使用 deal 给 wallet 合约充值 10 ether
        // 语法提示: deal(address(wallet), 10 ether);
        deal(address(wallet), 10 ether);
        

        // TODO 2: 模拟 user1 (必须是 owner 才能执行) 调用 executeTransaction(0)
        // 别忘了 vm.prank(...)
        vm.prank(user1);
        wallet.executeTransaction(0);

        // === 验证结果 ===
        // 验证交易状态是否变为 executed = true
        (,,, bool executed,) = wallet.getTransaction(0);
        assertEq(executed, true);

        // TODO 3: 验证 user2 的余额是否变成了 1 ether (原来是 0)
        // 语法提示: assertEq(user2.balance, ...);
        assertEq(user2.balance, 1 ether);
    }

    function test_RevertWhen_NotEnoughConfirmations() public {
        // 1. 准备工作：User1 提交并确认了一笔交易
        vm.prank(user1);
        wallet.submitTransaction(user2, 1 ether, "");
        
        vm.prank(user1);
        wallet.confirmTransaction(0);

        // 现在确认数只有 1 (需要 2)

        // TODO: 告诉 Foundry 下一步操作应该报错，错误信息是 "cannot execute tx"
        // 语法提示: vm.expectRevert(bytes("错误信息"));
        vm.expectRevert(bytes("cannot execute tx"));

        // TODO: 模拟 User1 强行调用 executeTransaction(0)
        vm.prank(user1);
        wallet.executeTransaction(0);
    }

    function test_RevertWhen_NonOwnerSubmits() public {
        // 1. 创建一个黑客地址
        address hacker = makeAddr("hacker");

        // 2. 预言会报错 "not owner"
        vm.expectRevert(bytes("not owner"));

        // 3. 模拟黑客身份 (vm.prank) 并提交交易
        vm.prank(hacker);
        wallet.submitTransaction(hacker, 0, "");
    }

    function testFuzz_SubmitTransaction(address _to, uint256 _value) public {
        // 1. 还是要假装是 owner (比如 user1)
        vm.prank(user1);

        // TODO: 调用 submitTransaction，但是使用传入的随机参数 _to 和 _value
        // 数据 data 依然可以留空 ""
        wallet.submitTransaction(_to, _value, "");

        // 2. 验证保存的数据是否和随机生成的参数一致
        (address txTo, uint256 txValue, , , ) = wallet.getTransaction(0);

        // TODO: 写两个断言 (assertEq)
        // 检查 txTo 是否等于 _to
        // 检查 txValue 是否等于 _value
        assertEq(txTo, _to);

        assertEq(txValue, _value);
        
    }
}
```
