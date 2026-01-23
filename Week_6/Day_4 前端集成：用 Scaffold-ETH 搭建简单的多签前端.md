# W6D4 学习笔记

今天我们的目标非常有意思

我们要把这几天写的多签合约,变成一个网页应用 (Dapp)

我们今天使用的是 Scaffold-ETH 2,这个工具对于黑客松和初学者来说那简直就是外挂,它帮我们把我们最头痛的前后端链接配置都做好了

你可以把它理解为一个 Web3 开发者的全能工具箱

在开发 Dapp 的时候,我们其实是同时在处理两个不同的世界

1. 区块链世界:这里是后端,有你的智能合约,它们一旦部署,就运行在链上或者本地模拟的链上,逻辑严密,不可篡改
2. 浏览器世界:前端。这里有你的用户界面,也就是用户看到的按钮输入框啥的。

在没工具箱的时候,把这两个世界链接起来是非常痛苦的,你需要手动把合约地址,合约接口复制到前端,一旦合约改了一个标点符号,这一套流程就得重来了。

Scaffold-ETH 2 就像一个自动化的桥梁,它把这两个世界装到了同一个项目里,并且配置好了自动同步

- 📂 `packages/hardhat`:这是后端(区块链世界)。你的 Solidity 代码住在这里。
- 📂 `packages/nextjs`:这是前端(浏览器世界)。你的 React/网页代码住在这里。

当你在 `hardhat` 文件夹里修改并部署合约时,这座桥梁会自动把最新的合约信息同步给 `nextjs` 文件夹,你不需要手动复制粘贴

## 安装 Scaffold-ETH 2

我们现在安装一下 Scaffold-ETH 2

打开你的 vscode 终端

**第一步** 把这个代码仓库下载下来,并且创建一个叫 `multi-sig-ui` 的文件夹

```bash
git clone https://github.com/scaffold-eth/scaffold-eth-2.git multi-sig-ui
```

**第二步** 进入这个文件夹

```bash
cd multi-sig-ui
```

**第三步** 安装依赖包,它会根据 `package.json` 安装所有需要的工具,可能需要几分钟时间

```bash
yarn install
```

## 配置合约

安装好之后我们进入文件夹,看到 `packages/hardhat`

这个就是你的链上工作室,所有的智能合约逻辑都在这里

我们把昨天写的 `MultiSig.sol` 复制到 `packages/hardhat/contracts/` 文件夹里。

然后把文件夹原本自带的 `YourContract.sol` 删掉,为了不会搞混

把文件放进去之后,我们需要告诉 Scaffold-ETH 怎么启动这个合约

这取决于我们昨天的合约需要什么启动参数,打开刚刚的 `MultiSig.sol` 文件,找到 `constructor` 开头的那行代码

```solidity
constructor(address[] memory _owners, uint _numConfirmationsRequired)
```

这行代码告诉我们,我们在创建的时候需要传一个地址的数组 `_owners` 和 `_numConfirmationsRequired` 数字进去

1. `_owners`:管理员是谁,这是一个数组
2. `_numConfirmationsRequired`:几个人签名才能通过,这是门槛

如果没有这两个信息就无法部署合约

我们现在去修改 Scaffold-ETH 2 的"自动部署机器人"脚本,把这两个参数喂给它。

在 VScode 中找到 `packages/hardhat/deploy/00_deploy_your_contract.ts` 这个文件

这里面的代码是部署 `YourContract` 那个自带的合约的,我们把它改为我们自己的

为了方便测试,我们先设置只有自己是多签钱包的唯一管理员,然后门槛也设置为 1

请把这个文件改为下面的代码,覆盖

```typescript
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { DeployFunction } from "hardhat-deploy/types";
import { Contract } from "ethers";

/**
 * Deploys a contract named "YourContract" using the deployer account and
 * constructor arguments set to the deployer address
 *
 * @param hre HardhatRuntimeEnvironment object.
 */
const deployMultiSigWallet: DeployFunction = async function (hre: HardhatRuntimeEnvironment) {

  const { deployer } = await hre.getNamedAccounts();
  const { deploy } = hre.deployments;

  await deploy("MultiSig", {
    from: deployer,
    // 这里对应你的构造函数参数:args: [ _owners, _numConfirmationsRequired ]
    // 我们填入:owners = [deployer] (只有你自己), confirmations = 1
    args: [ ["0xC1d5e7e03033DA5Ad5C539631f1C8c3B79bc1960"], 1 ],
    log: true,
    autoMine: true,
  });

};

export default deployMultiSigWallet;

// Tags are useful if you have multiple deploy files and only want to run one of them.
// e.g. yarn deploy --tags YourContract
deployMultiSigWallet.tags = ["MultiSigWallet"];
```

修改好文件之后保存

我们要让它运行起来了

首先我们要启动一个本地的区块链节点

打开一个新的终端,进入 `multi-sig-ui` 目录,然后输入以下命令

```bash
yarn chain
```

这相当于在你的电脑上启动了一个迷你的以太坊网络,你会看到很多日志

再打开第二个窗口,第一个窗口不要关闭,输入

```bash
yarn deploy
```

如果结果一切顺利,你就会在终端看到类似这样的绿色文字

```
deploying "MultiSig" (tx: 0x842f9d42ebb68074c697da304e3e91248d2ee1989903dc025e3aa73cb0dd6dcc)...: deployed at 0x5FbDB2315678afecb367f032d93F642f64180aa3(这个是你的合约部署的地址) with 1089756 gas
```

如果你看到了这段文字,那么恭喜你你已经征服了后端,你的智能合约已经活跃在本地区块链网络上面了

## 前端交互与调试

下面我们来进入第二阶段:前端交互与调试

在写代码之前,我们要知道一个事情

Scaffold-ETH 2 给我们自带了一个 Debug Contracts(合约调试)的面板。它能自动读取你的合约,然后生成好所有的按钮和输入框,你不用写一行代码就可以直接测试合约的功能。

我们利用这个面板来测试合约

首先我们要打开第三个终端窗口,前两个分别是 `yarn chain` 和 `yarn deploy`,别关掉他们,输入

```bash
yarn start
```

启动后你在终端会看到 `Ready in ... ms`,并告诉你访问地址是 `http://localhost:3000`

我们在浏览器中打开 `http://localhost:3000`

你应该会看到 Scaffold-ETH 的默认首页

在顶部的导航栏你一定看到一个标签叫: Debug Contracts。请点击它。

这时候出现的页面

- 左边:是你的合约名字,包括下面有合约的地址,还有几个只读的函数显示
- 右边:自动列出这个合约所有的读(read)和写(write)的方法

它怎么做到的?这么厉害

这就是 Scaffold-ETH 的自动化机制。你刚刚在运行 `yarn deploy` 的时候,它悄悄生成了一个 JSON 文件,把合约的 ABI(说明书)和地址注入进去前端文件夹里面了。前端页面读取到了这个文件,然后自动生成了对应的按钮。

现在,我们看看右上角有没有自动连接上 wallet,如果没有我们点击 connect 之后选择 Burner Wallet,这个是系统送你的临时钱包,里面有很多假币。

然后再检查 write 里面有没有 `submitTransaction` 方法?如果有就说明程序已经自动读懂你的合约逻辑了。

再在右侧找到 `owners`,输入 `0`,看看显示的地址和连接的地址是不是一样?

如果不一样,不用担心,这是正常的。

因为在终端的世界里面,你运行 `yarn deploy` 的时候,Hardhat 使用了一个默认的测试账号 A 作为部署者给你部署的,在你的脚本里面写了 `owners: [deployer]`,所以这个账号成为了你的管理员

在浏览器的世界中,当你打开网页的时候,Scaffold-ETH 为了方便,给你生成了一个临时的燃烧钱包 (Burner Wallet) 账号 B

所以现在合约只认 A 作为管理员,而不认识 B

这好办,你把你在浏览器的钱包告诉部署脚本就可以了。

复制前端的地址,点击右上角的地址之后,选择 copy address

然后回到 VS Code,打开 `packages/hardhat/deploy/00_deploy_your_contract.ts`。

我们把之前的 `deployer` 换成你的字符串

找到这一行

```typescript
args: [ [deployer], 1 ],
```

修改为

```typescript
// 这里的地址一定要是你浏览器里复制出来的那个!
args: [ ["0x1234...abcd"], 1 ],
```

注意地址要加引号变成字符串

再打开刚刚运行 deploy 的终端,重新输入

```bash
yarn deploy
```

你会看到部署了一个新合约,地址变了

回到浏览器刷新一下界面,再次运行上面的 `owners` 或者查看左边的 `getOwners` 你就会看到你的地址已经变成管理员地址了,搞定

## 使用多签合约转账

我们现在是多签合约真正的主人了,所以我们要开始:使用多签合约转一笔钱

我们通过三步来完成:充值 -> 提案 -> 执行

### 第一步:给多签钱包充值

钱包里有钱才能给别人发啊

先在 Debug Contracts 页面,你的合约名字(MultiSig)旁边有个地址,点击一下可以复制

我们点击左下角的 Faucet 按钮

在 Destination Address 中粘贴上你的合约地址

在 Amount 下面输入 `100`

点击 send

回到 Debug Contracts 页面,查看左侧栏,合约的 Balance 会变成 100

现在你的多签钱包已经变得富有了,而不是一个空壳

### 第二步:提交提案

我们现在要提交一个提案了

假设我们从这个账户给外部地址转一笔钱,比如 1 ETH

我们要调用 `submitTransaction` 方法

1. 准备一个接收地址,可以是任何地址,这里我提供一个随机的地址:`0xCf7Ed3AccA5a467e9e704C703E8D87F634fB0Fc9`
2. 填写提案详情

回到 Debug Contract 页面,找到 write 中的 `submitTransaction` 方法,填写三个参数

- `_to (address)`: 填入上面的接收地址。
- `_value (uint256)`: 这里有个大问题,我们写入的是 Wei 而不是 ETH,所以我们写入 1 之后,可以点右边的小乘号 `*`,它会自动变成 ETH 的单位,就是乘以 1000000000000000000
- `_data (bytes)`: 我们只是普通转账,不调用其他合约,所以这里填 `0x` (表示空数据)。

3. 点击 send 按钮提交上链
4. 等待几秒钟,我们可以看到成功提示,这笔交易没有真的转出去而是生成了一个提案

我们可以去左边的区域,找到 `getTransactionCount`,它的显示会变成 1,这代表着多签钱包已经多了一个提案了

### 第三步:签署并执行提案

现在我们接着来签署提案

因为我们设置 1 人即可执行,所以我们签署之后就可以执行提案了

找到 Write 区域的 `confirmTransaction` 方法。

在 `_txIndex (uint256)`: 填入 `0`。(因为这是第 1 个提案,计算机从 0 开始计数)。

点击 send

这时候你在 read 区域找到 `getTransaction`,`_txIndex` 输入 `0`,那么就会显示一些数字

前三个是我们传进去的参数,第四个是是否执行,第五个是签署提案的人数,现在已经变成 1 了

我们再找到 Write 区域的 `executeTransaction` 方法。

在 `_txIndex (uint256)`: 填入 `0`

点击 send

这时候你再查看 `getTransaction 0` 的时候,就会发现 false 变成了 true 交易成功执行了!

我们看看左侧,多签合约的钱是不是减少了,当然你也可以使用区块链浏览器,输入你转账之后的地址查看一下是不是有增加 ETH

## 构建用户界面

完成上面的步骤就跑通了整个流程,合约没问题,部署没问题,权限也没问题

这时候我们其实用的还是 Debug,从程序员的视角来调试合约

用户使用的时候可不能这样,他们只需要几个简单的显示和按钮,很轻松的就可以实现他们想做的而不是说自己非常麻烦的手动输入一大堆

我们现在要把 Debug 变成真正的用户使用 UI

我们要修改的页面在 👉 `packages/nextjs/app/page.tsx`,如果你使用的是旧版本,那么可能是 `pages/index.tsx`

打开这个文件,你可能看到许多现成的代码,把这些都删了,放入以下代码

```tsx
// packages/nextjs/app/page.tsx

"use client";

import type { NextPage } from "next";
import { useAccount } from "wagmi";
// 引入我们需要用到的 hook
import { useScaffoldReadContract, useScaffoldWriteContract } from "~~/hooks/scaffold-eth";

const Home: NextPage = () => {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen py-2">
      <h1 className="text-4xl font-bold mb-8">我的多签钱包 (My MultiSig)</h1>

      {/* 我们一会把组件填在这里 */}
    </div>
  );
};

export default Home;
```

### 显示多签钱包信息

第一步我们先显示多签钱包的管理员和提案数量

在 Scaffold-ETH 2 里,读取数据不需要写复杂的 ethers.js 代码,只需要用到一个叫 `useScaffoldReadContract` 的钩子 (Hook)

请在 `const Home: NextPage = () => {` 下面一行加上这段代码

```tsx
// 1. 读取多签合约的 owners 列表(测试用)
const { data: owners } = useScaffoldReadContract({
  contractName: "MultiSig", // 必须和你部署时的名字一样
  functionName: "getOwners", // 假设你的合约有个 getOwners 函数,如果没有就用 owners(0)
});

// 2. 我们先读"交易数量"来证明读取成功。
const { data: txCount } = useScaffoldReadContract({
  contractName: "MultiSig",
  functionName: "getTransactionCount",
});
```

然后在 return 中 `{/* 我们一会把组件填在这里 */}` 的下一行加入

```tsx
<div className="border p-4 rounded-xl">
  <p>当前提案数量: {txCount ? txCount.toString() : "0"}</p>
  <p>老板人数: {owners ? owners.length : "加载中..."}</p>
</div>
```

刷新一下浏览器,看看上面是不是更新了显示了

应该显示

```
当前提案数量: 2
老板人数: 1
```

### 创建提案表单

如果正常显示我们就要完成下一步

我们要写一个表单,直接在网页上发起提案

在 React 里,如果想让网页记住用户在输入框里填了什么(比如"转给谁"、"转多少"),我们需要用到 `useState`。

在文件最上方,找到 import 区域,添加以下代码

```tsx
import { useState } from "react";
import { parseEther } from "viem"; // 用于把 "1" ETH 变成 "1000..." Wei
import { useScaffoldReadContract, useScaffoldWriteContract } from "~~/hooks/scaffold-eth";
```

在 `const Home ...` 组件的内部,刚才写了 `useScaffoldReadContract` 下面,增加这几个记忆槽

```tsx
// ... 之前的读取代码 ...
// 记忆用户输入的:接收地址、金额、附加数据
const [toAddress, setToAddress] = useState("");
const [amount, setAmount] = useState("");
const [data, setData] = useState("0x");

// 准备写合约的工具
const { writeContractAsync: submitTx } = useScaffoldWriteContract("MultiSig");
```

我们需要一个 js 函数,当用户点击按钮时,他就会把输入框里的数据打包,发送给合约

接着在上面的变量下面添加下列函数

```tsx
const handleSubmit = async () => {
  try {
    await submitTx({
      functionName: "submitTransaction",
      // 参数顺序必须和合约里一样:_to, _value, _data
      args: [toAddress, parseEther(amount || "0"), data as `0x${string}`],
    });
    console.log("提案提交成功!");
  } catch (e) {
    console.error("提交失败:", e);
  }
};
```

`parseEther(amount)` 非常重要,用户如果输入 0.1,它会帮我们自动转换成合约能看懂的 100000000000000000 Wei

我们在页面上把输入框和按钮显示出来

在 return 的 div 里面,显示提案的 div 下面,加入下列代码

```tsx
{/* 分隔线 */}
<div className="w-full h-1 bg-gray-200 my-8"></div>

{/* 提交新提案的表单 */}
<div className="flex flex-col gap-4 p-6 border-2 border-primary rounded-xl w-96">
  <h2 className="text-2xl font-bold text-center">发起新转账</h2>
  
  {/* 输入地址 */}
  <input
    type="text"
    placeholder="接收方地址 (0x...)"
    className="input input-bordered w-full"
    value={toAddress}
    onChange={(e) => setToAddress(e.target.value)}
  />

  {/* 输入金额 */}
  <input
    type="text"
    placeholder="金额 (ETH)"
    className="input input-bordered w-full"
    value={amount}
    onChange={(e) => setAmount(e.target.value)}
  />

  {/* 提交按钮 */}
  <button 
    className="btn btn-primary"
    onClick={handleSubmit}
  >
    提交提案
  </button>
</div>
```

保存文件,回到浏览器刷新

测试一下,输入随机接收方地址,填入金额比如 0.1 点击提交提案

刷新界面之后,页面上的 "当前提案数量" 是不是自动从 1 变成 2 了?

如果变化了那么恭喜你,已经亲手做出了一个全功能的 web3 表单!

### 签名并执行交易

搞定,接下来到了最重要的环节,签名并且提交交易

我们就在刚刚的表单下面,弄一个审批卡片,用来显示刚刚提交的那笔交易 ID 为 1

我们需要一个读取钩子,这次专门读取第一号交易

回到 `packages/nextjs/app/page.tsx`,在 `useScaffoldReadContract` 代码下面,再加一段:

```tsx
// ... 之前的代码 ...

// 读取第 1 号交易的详情 (注意:Index 是 1,因为刚才你是在已有 1 个交易的基础上又提交了一个)
const { data: txDetails } = useScaffoldReadContract({
  contractName: "MultiSig",
  functionName: "transactions", // 访问 public 的 transactions 数组
  args: [1n], // 参数是 1 (注意后面有个 n,表示 BigInt,这是新版 Wagmi 的要求)
});

// 准备"批准"交易的工具
const { writeContractAsync: confirmTx } = useScaffoldWriteContract("MultiSig");
```

和刚才的 `handleSubmit` 类似,我们需要一个函数来处理点击"批准"按钮的动作。

```tsx
const handleConfirm = async () => {
  try {
    await confirmTx({
      functionName: "confirmTransaction",
      args: [1n], // 批准第 1 号交易
    });
    console.log("批准成功!");
  } catch (e) {
    console.error("批准失败:", e);
  }
};
```

然后再添加一个执行的逻辑,接着上面的代码写入

```tsx
// ... 之前的代码 ...

// 1. 新增:读取"最少确认数" (需要知道到底几个人签字才够)
const { data: requiredConfirms } = useScaffoldReadContract({
  contractName: "MultiSig", // 注意:你的合约名字叫 MultiSig,不是 MultiSigWallet,请核对!
  functionName: "numConfirmationsRequired",
});

// 2. 新增:准备"执行"交易的工具
const { writeContractAsync: executeTx } = useScaffoldWriteContract("MultiSig");
```

在 `handleConfirm` 下面,增加一个 `handleExecute`:

```tsx
const handleExecute = async () => {
  try {
    await executeTx({
      functionName: "executeTransaction",
      args: [1n], // 执行第 1 号交易 (Index 1)
    });
    console.log("执行成功!");
  } catch (e) {
    console.error("执行失败:", e);
  }
};
```

我们在前端页面根据不同的状态显示不同的按钮

- 如果已执行 -> 显示"✅ 已完成"。
- 如果签名不够 -> 显示"✍️ 确认"按钮。
- 如果签名够了但还没执行 -> 显示 "🚀 执行"按钮(或者两个都显示)。

请在 div 下面继续添加

```tsx
{/* 交易详情卡片 */}
<div className="flex flex-col gap-4 p-6 border-2 border-green-500 rounded-xl w-96 mt-8">
  <h2 className="text-2xl font-bold text-center text-green-700">交易控制台 #1</h2>
  
  {txDetails && requiredConfirms ? (
    <div className="text-sm space-y-2">
      <p><strong>目标:</strong> <span className="break-all">{txDetails[0]}</span></p>
      <p><strong>金额:</strong> {txDetails[1].toString()} Wei</p> 
      
      {/* 进度条显示:当前签名数 / 需要签名数 */}
      <p className="text-lg font-bold">
        签名进度: <span className="text-blue-600">{txDetails[4].toString()}</span> / {requiredConfirms.toString()}
      </p>

      <p><strong>状态:</strong> {txDetails[3] ? "✅ 已执行完毕" : "⏳ 等待处理"}</p>

      {/* 按钮逻辑区域 */}
      <div className="flex gap-2 mt-4">
        
        {/* 1. 如果还没执行,总是可以点击"确认" (这里简化了,实际应该检查是否已经签过) */}
        {!txDetails[3] && (
          <button 
            className="btn btn-warning flex-1"
            onClick={handleConfirm}
          >
            ✍️ 确认/签名
          </button>
        )}

        {/* 2. 只有当"当前签名 >= 要求签名" 且 "未执行" 时,才出现执行按钮 */}
        {!txDetails[3] && txDetails[4] >= requiredConfirms && (
          <button 
            className="btn btn-success flex-1"
            onClick={handleExecute}
          >
            🚀 执行交易
          </button>
        )}
      </div>

      {/* 如果已执行,显示个祝贺信息 */}
      {txDetails[3] && (
        <div className="alert alert-success mt-4">
          <span>交易已成功上链转账!</span>
        </div>
      )}
    </div>
  ) : (
    <p>正在加载数据...</p>
  )}
</div>
```

现在按顺序执行

1. 保存代码,刷新页面
2. 点击签名按钮,确认之后页面上的签名进度应该会变成 1 / 1,满足条件之后,旁边就会出现一个绿色的执行交易的按钮
3. 点击执行交易按钮等待一会,提案状态会变成已经执行完毕,最重要的是,你的多签钱包内的余额真的减少了!

如下图

![{0AB7C54A-DA61-47AE-8BDC-4A0BA7B42FEC}](https://github.com/user-attachments/assets/123616ac-8c2d-4b9a-9d85-a4fc37921fbc)

恭喜你,你现在已经不仅仅是写代码,而是在创建完整的 Dapp 了!
