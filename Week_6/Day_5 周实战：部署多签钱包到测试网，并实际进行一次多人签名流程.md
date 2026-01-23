# W6D5：多签钱包部署与测试实战

我们在前面 6 天的时间里面，从架构设计到逻辑编写，亲手打造了一个工业级的多签钱包。今天我们要把它部署到测试网进行实际的演练。

## 今天的目标

- **环境准备**：配置三个不同的测试网账户
- **上链部署**：将多签合约部署到 Sepolia，设定 2/3 签名阈值
- **资金注入**：向多签钱包转入资金
- **多人协作**：
  - A 提出提案
  - B 链上确认
  - A 执行提案

## 角色设定

我们需要模拟三个 owner，为了演示清晰，我们需要以下角色：

- **A**：控制权的主要持有者
- **B**：负责审核并签名
- **C**：可能作为备用

## 前提条件

- 你的 `MultiSig.sol` 代码已经准备好
- 你安装了 Foundry
- 你有 Sepolia 的 RPC URL（如果没有，请到 https://www.alchemy.com/rpc/ethereum-sepolia）
- 你的 A 和 B 账户中有少量的 Sepolia ETH

---

## 第一步：配置环境变量和账户

### 1. 创建和导入账户

打开你的 VSCode，连接上终端，然后创建三个账户。

```bash
# 先创建三个存放账户的文件夹
mkdir ownerA
mkdir ownerB
mkdir ownerC

# 再创建三个账户
cast wallet new ownerA
cast wallet new ownerB
cast wallet new ownerC
```

系统会提示你设置密码，记住你设置的密码。

### 2. 获取地址并领水

```bash
cast wallet address --keystore ownerA/xxxxx
cast wallet address --keystore ownerB/xxxxx
cast wallet address --keystore ownerC/xxxxx
```

> **注意**：`xxxxx` 是你生成的文件的随机数，在 `ownerA` 文件夹里，点击 tab 即可自动写入，下同。

可以使用 https://cloud.google.com/application/web3/faucet/ethereum/sepolia 登录谷歌账号领水，或者使用其他的水龙头也行。

### 3. 设置 RPC 变量，方便后续命令

```bash
export RPC_URL="https://eth-sepolia.g.alchemy.com/v2/YOUR_API_KEY"
```

把 `YOUR_API_KEY` 替换成你自己的 key。

---

## 第二步：部署合约到 Sepolia

我们部署一个 2-of-3 的多签钱包，三个 owner，两个人同意才能执行。

### 构造部署函数

我们需要传入两个参数：

```solidity
constructor(address[] memory _owners, uint _required)
```

### 执行命令

我们使用 ownerA 来部署：

```bash
forge create src/MultiSig.sol:MultiSig \
  --rpc-url $RPC_URL \
  --broadcast \
  --account ~/hello_foundry/ownerA/xxxxx \
  --constructor-args "[0xAAAA...,0xBBBB...,0xCCCC...]" 2
```

**参数说明**：
- `--broadcast`：命令为了把交易广播出去
- `xxxxx`：是你生成的文件的随机数，在 `ownerA` 文件夹里，如果错误，则填入你现在的 `ownerA` 的路径
- `"[...]"`：这里的地址用逗号分隔，注意不要有空格，或者用引号包起来
- `2`：表示 `_required`，即阈值

### 部署结果

执行完后你应该看到一些返回值：

```
Deployer: 0xDFdC3.....
Deployed to: 0xfC6.......
Transaction hash: 0xcc18f4......
```

这就是你部署好后的合约地址和 tx hash，我们把这个合约地址叫做 `MS_ADDR`。

---

## 第三步：往多签钱包注入资金

现在多签钱包是空的，为了测试转账功能，我们要转一些钱进去。

### 从 ownerA 向 MS_ADDR 转 0.01 ETH

```bash
cast send MS_ADDR \
  --value 0.01ether \
  --rpc-url $RPC_URL \
  --account ~/hello_foundry/ownerA/xxxx
```

> **注意**：把 `MS_ADDR` 写入你部署的合约地址。

### 检查多签钱包的余额

```bash
cast balance MS_ADDR --rpc-url $RPC_URL
```

如果显示 `10000000000000000`（即 0.01 ETH），说明转入成功！

---

## 第四步：多人签名环节

终于来到激动人心的多人签名环节了！我们假设 ownerA 想从多签钱包提取 0.005 ETH 作为自己的本月工资。

### 1. 发起提案

我们需要调用 `submitTransaction(address _to, uint _value, bytes memory _data)`。

**参数说明**：
- `_to`：接收钱的地址，owner 的地址（`0xRecipient...`）
- `_value`：0.005 ETH（即 5000000000000000 wei）
- `_data`：因为是纯转账，没有附带数据，所以是 `0x`

**使用命令**：

```bash
cast send MS_ADDR "submitTransaction(address,uint256,bytes)" \
  ownerAaddress 0.005ether 0x \
  --rpc-url $RPC_URL \
  --account ~/hello_foundry/ownerA/xxxxx
```

> **注意**：把 `MS_ADDR` 换成合约地址，把 `ownerAaddress` 换成 ownerA 地址。

### 查看当前提案数

接着调用合约查看当前提案数，确认提案是否产生：

```bash
cast call MS_ADDR "getTransactionCount()" --rpc-url $RPC_URL
```

返回值应该是 `1`，我们生成的提案索引（`txIndex`）是 `0`。

---

### 2. 确认提案

由 ownerB 来确认，因为 ownerA 一个人确认是不够的，现在 ownerB 也要确认之后才能通过提案。

我们要调用 `confirmTransaction(uint _txIndex)`，本次 `txIndex` 为 `0`。

```bash
cast send MS_ADDR "confirmTransaction(uint256)" \
  0 \
  --rpc-url $RPC_URL \
  --account ~/hello_foundry/ownerB/xxxxx
```

> **注意**：这里用了 `--account ~/hello_foundry/ownerB`，代表切换了身份。

耐心等待一会，会返回一些值，代表成功。

我们再使用 ownerA 进行上述操作，等待成功之后继续下面的操作。

---

### 3. 确认签名数量

调用以下命令：

```bash
cast call MS_ADDR "getTransaction(uint256)" 0 --rpc-url $RPC_URL
```

会返回一个长长的值，你估计看不懂。我们调用以下命令：

```bash
# 1. 先把返回值赋值给变量（或者你直接手动复制那一长串）
RETURN_DATA=0x0000... # (你刚才那一长串)

# 2. 使用 decode 命令
cast abi-decode "getTransaction(uint256)(address,uint256,bytes,bool,uint256)" $RETURN_DATA
```

这下显示的值就能看明白了吧！看看最后一个值，是不是显示 `2`？

- 如果是，就说明两个人都签名了
- 如果不是，你就得检查谁没有签名了

---

### 4. 执行提案

接下来到了执行的环节，签名人数够了之后，我们就需要执行了。由任何一个人执行都可以，这里我们选择 A 执行。

调用 `executeTransaction(uint _txIndex)`：

```bash
cast send MS_ADDR "executeTransaction(uint256)" \
  0 \
  --rpc-url $RPC_URL \
  --account ~/hello_foundry/ownerA/XXXXX
```

---

## 第五步：验证结果

最后我们检查一下，看看到底成功了没有？

### 余额的变化

- 多签钱包少了 0.005 ETH
- owner 多了 0.005 ETH

### 使用 cast 验证

```bash
# 检查多签钱包余额
cast balance MS_ADDR --rpc-url $RPC_URL
```

看到只剩下 **5000000000000000**。

```bash
# 检查 ownerA 余额
cast balance ownerA --rpc-url $RPC_URL
```

看到多了 5000000000000000。

**成功！** 🎉









  
