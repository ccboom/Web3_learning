# 代理合约（Proxy Contract）实战

第五周高强度学习的最后一天，我们来把这周学的内容组装起来，然后写一个代理合约。

---

## 第一部分：为什么需要代理合约？

### 区块链的铁律

我们知道区块链有一个铁律：**一旦部署，就不可修改**。

假设你把一套房子（合约）用钢筋水泥建好了，如果要改户型，那你只能拆了重建。

但是有一个巨大的问题，你这一栋楼里面又不是只有你一家人，那你楼上楼下的邻居（用户的数据和资产）咋办，不管他们了吗？

### 解决方案：身体和大脑分开

所以开发者就想了一个聪明的办法：**把身体和大脑分开**。

我们把合约拆分成两个部分：

#### 1. 外壳 Proxy 合约（身体）

- 它就像一个**前台接待**
- 它有固定的地址，用户永远都是和它交互
- 它负责保存数据
- 但是它很笨，没有任何逻辑业务

#### 2. 大脑 Implementation 逻辑合约（大脑）

- 它就像办公室中的**专家**
- 它只负责逻辑，比如转账、计算等
- 它不会存用户的真实数据

### 工作流程

当用户来办事的时候：

1. 首先到前台
2. 前台会将用户办的事告诉专家
3. 专家会告诉前台怎么做
4. 前台照做就可以了

### 升级机制

如果我们发现，专家的计算方法过时了，我们想要更新他，那我们就应该换掉专家。

- **前台可不能换**：大家的资产都放在前台的账本里，不能乱
- **专家可以换**：找一个新的专家（部署新的逻辑合约），告诉前台以后去找新的专家问，旧的我们就不用他了

---

## 第二部分：存储冲突问题

### 问题的提出

既然前台知道要去问谁，那么肯定要记下他在哪里（address）对吧？

假设 proxy 有一个巨大的文件柜（storage），柜子有无数个抽屉：

1. 前台（proxy）决定把地址存在第 0 个抽屉里
2. 专家合约没有抽屉，它告诉前台：把用户的等级放在第 0 个抽屉里
3. 前台操作的时候，操作的可是自己的抽屉，直接把用户等级覆盖了专家的地址！

当你下次再要找专家的时候，地址变成了 `0x000...01`，这个合约根本不存在，前台直接傻眼了。

这就是所谓的**存储冲突（Storage Collision）**。

### 解决方案：EIP-1967

既然 slot0、slot1 ....这些很靠前的抽屉都很容易被占用，那么我们就得把专家地址藏到一个逻辑合约永远找不到的地方。

我们把专家地址放在一个特殊的 slot 里面：

**EIP-1967 规定**，我们用 `"eip1967.proxy.implementation"` 的 keccak256 值地址来存放。

这个数字大概是 $10^{77}$ ，即使 slot 每秒加 1，宇宙毁灭也加不到这么大的数字。

这样，专家（逻辑合约）里的等级，和前台（proxy 合约）里的 implementation（专家地址）就再也不会打架了。

---

## 第三部分：实现代码

明白了上面的原理，我们就可以开始着手写合约了。

我们需要使用 Assembly（汇编）来读写这个特殊的 slot，由于还没学习汇编，我会把代码拆开讲解。

### 存取专家地址

看看下面的代码：

```solidity
// 这是 EIP-1967 定义的存储位置
bytes32 private constant _IMPLEMENTATION_SLOT = 
    bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);

// 设置顾问地址
function _setImplementation(address newImplementation) private {
    assembly {
        // sstore = Storage store (存入存储)
        // 把 newImplementation 存入 _IMPLEMENTATION_SLOT 这个位置
        sstore(_IMPLEMENTATION_SLOT, newImplementation)
    }
}

// 读取顾问地址
function _getImplementation() private view returns (address impl) {
    assembly {
        // sload = Storage Load (读取存储)
        // 从 _IMPLEMENTATION_SLOT 读取数据放入 impl
        // := 的作用和 solidity 中 = 是一样的
        impl := sload(_IMPLEMENTATION_SLOT)
    }
}
```

### 为什么要减一？

这时候你会发现，不是说好使用 `keccak256("eip1967.proxy.implementation")`，为什么还要**减一**呢？

这是一个有意思的安全细节——**烧掉回来的桥**。

**正常情况下**：

如果你定义一个 `mapping(string => uint) map`，然后存入 `map["eip1967.proxy.implementation"]`，系统算出来的存储位置就是 `keccak256("eip1967.proxy.implementation")`。

虽然概率非常小，但万一你的逻辑合约里真有一个 mapping 刚好使用了这个字符串，那还是会撞车的。

**减一的原因**：

- 我们知道 A 的 hash 是 H
- 但是我们不知道一个字符串 B 让他的 hash 正好等于 H - 1

所以手动减一，我们就创造一个不可能由任何 solidity 正常变量生成的存储位置，也就从数学上彻底杜绝了冲突的可能性。

### 黑盒子理解

我们只需要把（`_setImplementation` 和 `_getImplementation`）当作两个黑盒子：

- **盒子 _setImplementation**：给他一个地址，他就能在那个神秘的位置给你保存下来
- **盒子 _getImplementation**：打开这个盒子，就能把那个地址读出来

---

## 第四部分：Fallback 传声筒

### 场景描述

我们的前台有巨大的抽屉我们已经知道了，里面装着专家的地址。

接下来我们要赋予前台最重要的能力，也就是**传声能力**。

想象这样一个场景：

你去前台，告诉工作人员，把我的等级加一（levelup）。

前台的工作人员翻了翻自己的工作手册，发现根本没有 levelup 这个工作类目。

这时候他就会触发**应急反应**，也就是 solidity 中的 `fallback()`。

### Fallback 的四步

在 `fallback()` 里面，我们只需要做一件事，把你的指令原封不动的传给专家解决。

虽然这里必须用 Assembly 汇编来实现原封不动的转发，但是其实只有四步：

1. **抄写指令（calldatacopy）**：把用户发来的指令写到纸上
2. **询问专家（delegatecall）**：去找到专家，然后把指令交给他
3. **产生结果（returndatacopy）**：专家说清楚怎么执行，然后使用专家的方法执行指令，得到结果
4. **交差（return / revert）**：把产生的结果交给用户

### 代码实现

```solidity
fallback() external payable {
    // 从抽屉中找到专家地址
    address _imp = _getImplementation();

    assembly {
        // 这里的代码就是上面的四步的翻译，我们不需要背诵，只要知道它在做什么就可以

        // 1. 复制指令
        calldatacopy(0, 0, calldatasize())

        // 2. 使用 delegatecall 调用逻辑合约
        // out 和 outsize 为 0，我们不知道返回的数据有多长，稍后用 returndatacopy 处理
        let result := delegatecall(gas(), _imp, 0, calldatasize(), 0, 0)

        // 3. 返回数据
        returndatacopy(0, 0, returndatasize())

        // 4. 根据返回的数据决定是报错还是成功执行给用户数据
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

---

## 第五部分：完整代码

明白了吧！

现在我们已经掌握了 proxy 合约的所有秘密：

- 前台（proxy）和专家（logic）分开
- 我们把专家地址藏在一个无人能发现的角落（EIP-1967）
- 通过 delegatecall 来执行专家的逻辑
- 要是想升级，只要把专家地址换新就可以了。前台的账本完全不用动

### 完整代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// ==========================================
// 第一部分：前台 Proxy 合约 (身体/外壳)
// ==========================================
contract Proxy {
    // 1. 定义那个"宇宙尽头"的神秘抽屉位置 (EIP-1967)
    // 这里的 hex 串就是 keccak256("eip1967.proxy.implementation") - 1 的结果
    bytes32 private constant _IMPLEMENTATION_SLOT = 
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    // 构造函数：大楼刚建好时，先请第一个专家进驻
    constructor(address _logic) {
        _setImplementation(_logic);
    }

    // --- 黑盒子：存取专家地址 ---

    function _setImplementation(address newImplementation) private {
        assembly {
            sstore(_IMPLEMENTATION_SLOT, newImplementation)
        }
    }

    function _getImplementation() private view returns (address impl) {
        assembly {
            impl := sload(_IMPLEMENTATION_SLOT)
        }
    }

    // --- 管理员功能 ---
    
    // ⚠️ 注意：为了教学简单，这里没有加 admin 权限控制，也没有把 admin 存到特殊 slot。
    // 在真实生产环境中，必须加权限，否则谁都能把你的专家换掉！
    function upgradeTo(address newImplementation) external {
        _setImplementation(newImplementation);
    }

    // --- 核心能力：传声筒 (Fallback) ---

    // 当用户调用的函数（比如 levelUp）在 Proxy 里找不到时，触发这里
    fallback() external payable {
        _delegate(_getImplementation());
    }

    // 接收纯 ETH 转账
    receive() external payable {
        _delegate(_getImplementation());
    }

    // 具体的汇编转发逻辑 (抄写 -> 问专家 -> 交差)
    function _delegate(address _implementation) internal {
        assembly {
            // 1. 抄写指令：把 calldata 复制到内存
            // calldatacopy(dst, src, length)
            calldatacopy(0, 0, calldatasize())

            // 2. 问专家：使用 delegatecall
            // result := delegatecall(gas, address, inputPos, inputLen, outputPos, outputLen)
            // 这里的 out/outsize 设为 0，因为我们还不知道专家会返回多少数据
            let result := delegatecall(gas(), _implementation, 0, calldatasize(), 0, 0)

            // 3. 拿结果：把专家的返回数据复制到内存
            returndatacopy(0, 0, returndatasize())

            // 4. 交差：根据结果决定是 return (成功) 还是 revert (失败)
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

// ==========================================
// 第二部分：专家 Logic 合约 (大脑/逻辑)
// ==========================================

// 专家 V1：只会简单的升级
contract LogicV1 {
    // 这里的变量会对应 Proxy 的 Slot 0
    // 因为 Proxy 自身没有定义 public 变量，所以 Slot 0 是空的，很安全
    uint256 public level; 
    address public lastUser; // Slot 1

    function levelUp() public {
        level += 1;
        lastUser = msg.sender;
    }
}

// 专家 V2：学会了"双倍经验"升级
contract LogicV2 {
    // ⚠️ 必须保持和 V1 一样的存储布局！
    uint256 public level; 
    address public lastUser;

    // 新增的功能
    function levelUp() public {
        level += 10; // 哇！一次升10级！
        lastUser = msg.sender;
    }

    // 新增一个函数
    function reset() public {
        level = 0;
    }
}
```

---

## 总结

### 1. 代理合约的核心概念

**问题**：区块链合约一旦部署就不可修改，但业务逻辑需要升级。

**解决方案**：将合约分为两部分：
- **Proxy（前台/身体）**：固定地址，存储数据，无业务逻辑
- **Implementation（专家/大脑）**：可替换，包含业务逻辑，不存储数据

### 2. 存储冲突问题

**问题**：Proxy 和 Implementation 共享同一个 Storage，容易发生存储槽位冲突。

**解决方案**：EIP-1967 标准
- 使用 `keccak256("eip1967.proxy.implementation") - 1` 作为存储位置
- 这个位置约为 $10^{77}$ ，逻辑合约永远无法到达
- 减一操作确保没有任何 Solidity 变量能自然生成这个位置

### 3. 核心技术：delegatecall

**工作原理**：
1. 用户调用 Proxy 合约
2. Proxy 的 `fallback()` 被触发
3. 通过 `delegatecall` 调用 Implementation
4. Implementation 的代码在 Proxy 的上下文中执行
5. 修改的是 Proxy 的 Storage，使用的是 Proxy 的 `msg.sender`

### 4. Fallback 的四步流程

1. **calldatacopy**：复制用户的调用数据
2. **delegatecall**：委托调用 Implementation 合约
3. **returndatacopy**：复制返回数据
4. **return/revert**：返回结果或回滚

### 5. 升级机制

**升级步骤**：
1. 部署新的 Implementation 合约（V2）
2. 调用 Proxy 的 `upgradeTo(newAddress)`
3. Proxy 更新 Implementation 地址
4. 用户数据保持不变，逻辑已升级

**注意事项**：
- 新版本必须保持与旧版本相同的存储布局
- 不能改变已有变量的顺序和类型
- 只能在末尾添加新变量

### 6. 安全考虑

**权限控制**：
- 生产环境必须添加 admin 权限控制
- 只有管理员才能调用 `upgradeTo()`
- Admin 地址也应存储在特殊 slot（EIP-1967 定义了 admin slot）

**存储布局**：
- 升级时必须保持存储布局兼容
- 违反会导致数据错乱或丢失

### 一句话总结

代理合约通过将数据存储（Proxy）和业务逻辑（Implementation）分离，利用 EIP-1967 标准避免存储冲突，通过 delegatecall 实现逻辑调用，从而在保持合约地址和用户数据不变的情况下实现业务逻辑的可升级性。
