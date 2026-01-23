# 实现一个简单的区块链：挖矿和 Hash 计算

> 本周总结：写一个能实现挖矿和 hash 计算功能的小链

---

## 第一部分：区块 (Block) 的定义

### 一个块里面需要有什么？

首先我们思考一下，一个块里面需要有什么？

| 属性 | 说明 |
|------|------|
| `data` | 交易数据（必须的） |
| `timestamp` | 时间戳（知道哪个块先出来的） |
| `previous_hash` | 前一个块的 hash 值（形成链） |
| `index` | 当前区块的位置 |
| `hash` | 本区块的 hash 值 |
| `nonce` | 随机数（挖矿时会改变） |

### Nonce 的作用

**Nonce** 是一个数字，为什么我们要有 nonce？

我们在计算一个块的时候，hash 会设置一个条件让人们难以达到，所以就要不断改变 nonce 的值，直到这个块的 hash 变成一个符合条件的值。这就是 nonce 的作用。

### Block 类的实现

```python
class Block:
    def __init__(self, index, previous_hash, data):
        self.index = index
        self.timestamp = time.time()
        self.data = data
        self.previous_hash = previous_hash
        self.nonce = 0  # 初始随机数，挖矿时会变
        self.hash = self.calculate_hash()  # 计算当前区块的指纹
```

---

## 第二部分：计算区块的 Hash 值

### 如何计算 Hash？

我们在计算本区块的 hash 的时候，需要把上面初始化出了本区块 hash 的其他值全部加进去搅拌，使用 SHA-256 函数进行计算。

**重要**：我们要把这些数据相加，不是把他们的值相加，而是**拼接**，一个接着一个。在 Python 中我们使用 `str` 来拼接，最后把他们变成字节进行计算。

### calculate_hash 方法

```python
def calculate_hash(self):
    block_string = str(self.index) + str(self.timestamp) + str(self.previous_hash) + str(self.nonce) + str(self.data)
    return hashlib.sha256(block_string.encode()).hexdigest()
```

---

## 第三部分：挖矿 (Mining) - 工作量证明 (PoW)

### 什么是工作量证明？

在 BTC 网络中，并不是随便算出一个 hash 就能打包发布。网络要求必须算出来**特定的 hash** 才行，通常这个 hash 的值是以一定的 0 作为开头的。

### Difficulty 参数

我们现在写一个 `mine` 函数，参数里有一个 `difficulty`，代表 hash 以 `difficulty` 个 0 开头。

现在我们发现其他值都是固定的，只有 `nonce` 可以改变，这就是 nonce 的作用，非常重要。

### mine_block 方法

```python
def mine_block(self, difficulty):
    if self.hash is None:
        self.hash = self.calculate_hash()

    while self.hash[:difficulty] != difficulty * '0':
        self.nonce += 1
        self.hash = self.calculate_hash()
```

**工作原理**：
1. 检查 hash 的前 `difficulty` 个字符是否都是 '0'
2. 如果不是，增加 nonce 并重新计算 hash
3. 重复直到找到符合条件的 hash

---

## 第四部分：区块链 (Blockchain) 的实现

### 积木连成一条链

至此，我们已经完成了一个简单的区块的定义。现在我们积木已经做好了，现在要把这个积木连成一条链。

### 区块链的存储

我们要创造一个 `Block_chain` 类来管理这一切。我们要怎么放这些块？是不是一个挨着一个放？我们在简单区块链中使用 Python 的 `list` 实现。

### 创世区块

那我们现在想，我们创建第一个区块，通常叫**创世区块**。那我们的 `previous_hash` 应该写什么？我们把它定义为 `'0'` 即可。

我们还得设置出块难度，就是上面说的 `difficulty`。

### Block_chain 类的初始化

```python
class Block_chain:
    def __init__(self):
        self.blocks = []
        self.difficulty = 4
        self.blocks.append(Block(0, '0', 'hello world'))
```

---

## 第五部分：添加区块到链

### add_block 方法的逻辑

好了，我们现在要让这条链动起来，也就是实现 `add_block(self, data)` 方法。增加区块我们需要知道上一个块的 hash 值和他自己的 index。

**如何在 Python 中找到 list 的最后一个元素？** 使用 `list[-1]` 即可。

**如何找到他自己的 index？** 使用 `len(list)` 即可。

### 逻辑步骤

1. 获取前一个块的 hash 值
2. 创建新的区块，写入他的 index，写入 previous_hash
3. 挖矿，算出一个 nonce 让整个区块的 hash 值符合要求
4. 上链，添加到链的 list 里面

### add_block 方法

```python
def add_block(self, data):
    previous_hash = self.blocks[-1].hash
    block = Block(len(self.blocks), previous_hash, data)
    block.mine_block(self.difficulty)
    self.blocks.append(block)
```

---

## 第六部分：防止篡改 - 链的验证

### 区块链的核心

我们现在已经可以有一个完整的链了。那么区块链的核心就是**防止篡改**。如果有人把里面的转账 10 块变成了转账 10 万我们怎么办？我们得写一个检测的功能，以保证链中区块数据没有被篡改。

### 验证的两个步骤

1. **数据自检**：重新计算当前区块的 hash 值，看他和存储的 `self.hash` 是否一致。如果不一致，那么数据被篡改了

2. **链路检查**：检查上一个块的 hash 是不是这个块的 `previous_hash`。如果不相等，说明链出问题了

### is_chain_valid 方法

```python
def is_chain_valid(self):
    for i in range(1, len(self.blocks)):
        current_block = self.blocks[i]
        previous_block = self.blocks[i - 1]

        # 数据自检
        if current_block.calculate_hash() != current_block.hash:
            return False

        # 链路检查
        if current_block.previous_hash != previous_block.hash:
            return False

    return True
```

---

## 完整源码

```python
import hashlib
import time

class Block:
    def __init__(self, index, previous_hash, data):
        self.index = index
        self.timestamp = time.time()
        self.data = data
        self.previous_hash = previous_hash
        self.nonce = 0  # 初始随机数，挖矿时会变
        self.hash = self.calculate_hash()  # 计算当前区块的指纹

    def calculate_hash(self):
        block_string = str(self.index) + str(self.timestamp) + str(self.previous_hash) + str(self.nonce) + str(self.data)
        return hashlib.sha256(block_string.encode()).hexdigest()

    def mine_block(self, difficulty):
        if self.hash is None:
            self.hash = self.calculate_hash()

        while self.hash[:difficulty] != difficulty * '0':
            self.nonce += 1
            self.hash = self.calculate_hash()


class Block_chain:
    def __init__(self):
        self.blocks = []
        self.difficulty = 4
        self.blocks.append(Block(0, '0', 'hello world'))

    def add_block(self, data):
        previous_hash = self.blocks[-1].hash
        block = Block(len(self.blocks), previous_hash, data)
        block.mine_block(self.difficulty)
        self.blocks.append(block)

    def is_chain_valid(self):
        for i in range(1, len(self.blocks)):
            current_block = self.blocks[i]
            previous_block = self.blocks[i - 1]

            if current_block.calculate_hash() != current_block.hash:
                return False

            if current_block.previous_hash != previous_block.hash:
                return False

        return True


if __name__ == '__main__':
    blockchain = Block_chain()
    blockchain.add_block('hello world')
    if blockchain.is_chain_valid():
        print('Block chain is valid')
    else:
        print('Block chain is invalid')
    
    blockchain.add_block('hello world2')
    if blockchain.is_chain_valid():
        print('Block chain is valid')
    else:
        print('Block chain is invalid')
    
    # 尝试篡改数据
    blockchain.blocks[1].data = "Hacked Data"
    if blockchain.is_chain_valid():
        print('Block chain is valid')
    else:
        print('Block chain is invalid')
```

---

## 总结

到这一步，一个包含以下特性的区块链就完成了：

✅ **PoW 挖矿** - 工作量证明机制

✅ **Hash 链条** - 通过 hash 值连接区块

✅ **不可篡改** - 任何数据修改都会被检测出来

这就是 **BTC 和所有加密货币的底层架构**！

---

## 测试结果

运行上面的代码，你会看到：

```
Block chain is valid
Block chain is valid
Block chain is invalid
```

最后一个是 `invalid`，因为我们篡改了第二个区块的数据，链的验证机制检测到了这个篡改！
