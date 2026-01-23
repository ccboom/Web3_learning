# 手写实现 Merkle Tree (Python)

> 理解其在轻节点验证中的作用

---

## 第一部分：Merkle Tree 基础概念

昨天学习了 SHA-256 算法，今天需要学习 Merkle Tree。

### 什么是 Merkle Tree？

Merkle Tree 是一种将巨大数据量压缩为一个 hash 的结构，利用了昨天的**雪崩效应**——任何一个微小的变动都会由底层一直沿着树上传，导致最后的根节点发生变化。

### 基本构建过程

我们用最经典的四块来推演，假设我们现在有数据块 `[A, B, C, D]`：

1. 这四个块是**叶子节点**，我们在把他们上传的时候不应该把原值上传，应该上传他们的 hash 值
2. Merkle Tree 是一个**二叉树**，所以我们现在需要两个两个配对，来合并成父节点之后再配对

公式：

$$H_{AB} = Hash(H_A + H_B)$$

现在好了，我们获得了 `H_AB` 和 `H_CD` 的 hash 值，然后再用公式：

$$H_{root} = Hash(H_{AB} + H_{CD})$$

就可以算出来 root 的 hash 值了！

### 奇数节点的处理

那么新问题出现了，如果我们有五个值 `[A, B, C, D, E]`，那么 E 和谁配对？

有人肯定想说把 E 和 `H_CD` 配对不就行了，但这样**不行**。

我们现在使用的是二叉树，所以配对的时候尽量也要使用二叉配对。到最后一个如果一直剩，那他不就和 root hash 的上一层配对了，这对于加密来说很危险。

**解决方案**：我们再克隆一个 E 进去，变成 `[A, B, C, D, E, E]`，让他和自己配对，这样就可以保证是整齐的二叉树了。

### 正式定义

Merkle Tree 也叫**哈希树**，是一种用哈希值搭建起来的数据结构。

我们可以把他想象成一颗倒过来的树：

![Merkle Tree 结构图](https://github.com/user-attachments/assets/88340418-86af-42f1-b741-fc8f2c4b7e27)

- **树叶（Leaves）**：最下面的一层，是原始数据的 hash 值
- **树枝（Branches）**：中间的一层，是两个树叶的 hash 值的合并 hash
- **树根（Root）**：最顶点，也就是全部加起来的 hash 值，最终的结果

### 核心作用

1. **数据压缩**：能把非常多的数据浓缩为一个 hash 值
2. **防止篡改**：如果改了下面其中的一个 hash 值，最后的 root 值将会变化非常大（雪崩效应）
3. **轻量级验证**：如果我们想知道 A 在这个树里面，我们不需要下载全部的数据，只需要知道他路径上配对的 hash 值即可算出 root hash，再与 root hash 进行对比即可

---

## 第二部分：代码实现

### 1. Hash 函数

首先我们要想想我们最初的数据应该是什么——对的，就是交易的 hash 值。我们先写出以下的代码：

```python
@staticmethod
def hash_data(data):
    # 1. 编码为 bytes
    encoded_data = str(data).encode('utf-8') 
    # 2. 计算 SHA-256 并返回 16进制字符串
    return hashlib.sha256(encoded_data).hexdigest()
```

这个代码可以让我们把一个输入的数据变为一个 hash 值。为了能够处理 int，我们加入了 `str()`，把他变为了 string。

### 2. 初始化函数

接下来就要开始写初始化函数了。我们定义一个类一般都有这个函数，我们要在初始化的时候就把所有交易的 hash 值存储起来，而且我们要在初始化的时候把这个树建立起来，使用 `build()` 函数：

```python
def __init__(self, data_list):
    self.data_list = data_list
    # 初始化最底层（叶子）
    self.leaves = [self.hash_data(i) for i in data_list]
    # 我们用一个列表来保存所有的层级，方便后面做验证
    # index 0 是叶子层，最后一个 index 就是根节点
    self.levels = [self.leaves]
    self.build()
```

### 3. Build 函数

那么 `build` 函数的内容是什么？其实就是往上面盖楼，一层一层的。

首先我们把我们的最低层的 hash 值全部拿出来，计算出两个两个的 hash，然后传入上一层，继续这个操作直到剩下一个 hash，这就是我们的 root：

```python
def build(self):
    current_level = self.leaves
    
    while len(current_level) > 1:
        next_level = []
        
        for i in range(0, len(current_level), 2):
            left_node = current_level[i]
            
            if i + 1 < len(current_level):
                right_node = current_level[i + 1]
            else:
                # 奇数情况，复制最后一个节点
                right_node = left_node
            
            # 父节点哈希 = Hash(左孩子 + 右孩子)
            combined_hash = self.hash_data(left_node + right_node)
            next_level.append(combined_hash)
        
        self.levels.append(next_level)
        current_level = next_level
        
    self.root = current_level[0]
    return self.root
```

### 4. 完整代码

最终我们构建完成了一个 Merkle Tree，总代码如下：

```python
import hashlib

class MerkleTree:
    def __init__(self, data_list):
        self.data_list = data_list
        # 初始化最底层（叶子）
        self.leaves = [self.hash_data(i) for i in data_list]
        self.levels = [self.leaves] 
        self.build() 

    @staticmethod
    def hash_data(data):
        # 1. 编码为 bytes
        encoded_data = str(data).encode('utf-8') 
        # 2. 计算 SHA-256 并返回 16进制字符串
        return hashlib.sha256(encoded_data).hexdigest()

    def build(self):
        current_level = self.leaves
        
        while len(current_level) > 1:
            next_level = []
            
            for i in range(0, len(current_level), 2):
                left_node = current_level[i]
                
                if i + 1 < len(current_level):
                    right_node = current_level[i + 1]
                else:
                    # 奇数情况，复制最后一个节点
                    right_node = left_node
                
                # 父节点哈希 = Hash(左孩子 + 右孩子)
                combined_hash = self.hash_data(left_node + right_node)
                next_level.append(combined_hash)
            
            self.levels.append(next_level)
            current_level = next_level
            
        self.root = current_level[0]
        return self.root

# 测试代码
data = ['A', 'B', 'C', 'D']
mt = MerkleTree(data)
print(f"Merkle Root: {mt.root}")
```

运行之后会得到一个 16 进制的字符串，那就是 **Merkle Root**！

---

## 第三部分：轻节点的验证 (Merkle Proof)

> 如果还不知道轻节点和全节点的区别，可以查看：[轻节点与全节点](https://zhuanlan.zhihu.com/p/7993733330)

### 场景描述

比如我们现在是一个**轻节点**，我们只有最终的 root。

现在有人告诉你说："A 在这个区块中哦"

但是你不知道是不是真的，为了验证这个，你就告诉那人说你必须给我一个**验证路径**，证明这个 A 在这个区块中。

### 验证所需的信息

比如我们现在假设有 `['A', 'B', 'C', 'D']`，那么我们需要一个 Merkle Proof 来验证 A 到底是不是真的在区块中。我们需要哪些 hash 值呢？

1. **A 的 hash 值**：这个从他们给我们的原始数据可以直接计算
2. **A 在这个块中的位置**：我们必须要知道位置的原因是 `hash(A + B)` 与 `hash(B + A)` 完全不同，所以我们要知道正确的位置
3. **路径上配对的 hash 值**：还得知道从叶子节点开始往上的每一个和我们配对的 hash 值，这样才能从路径算出 root hash

### 获取证明路径

好了，我们要求全节点给我们准备这三个东西，那么下面的代码就可以帮助我们获取我们想要的东西：

```python
def get_proof(self, index):
    proof = []
    for level in self.levels[:-1]:  # 不包含根节点层
        # 1. 找兄弟
        if (index % 2) == 0:
            sibling_index = index + 1
        else:
            sibling_index = index - 1

        # 2. 处理落单情况 (指向自己)
        if sibling_index >= len(level):
            sibling_index = index

        # 3. 拿到证据
        sibling_hash = level[sibling_index]
        proof.append(sibling_hash)

        # 4. 向上爬一层
        index = index // 2
    
    return proof
```

### 验证证明

好了，现在有了我们想要的东西了，现在轻节点可以写一个函数来验证他给我们的是不是正确的：

```python
def verify_proof(self, index, proof, target_hash, root_hash):
    temp_hash = target_hash
    for i in range(len(proof)):
        if index % 2 == 0:
            temp_hash = self.hash_data(temp_hash + proof[i])
        else:
            temp_hash = self.hash_data(proof[i] + temp_hash)

        index = index // 2

    print(temp_hash)
    if root_hash == temp_hash:
        return True
    else:
        return False
```

---

## 总结

到此我们写出了全部 Merkle Tree 的核心组件：

| 功能 | 描述 |
|------|------|
| **构建** | 构建一个自下而上的 tree，生成 root |
| **生成证明** | 全节点为其中某个数据提供路径支持 |
| **验证** | 轻节点拿着证明和自己 root 做对比 |

---

## 最后一个问题

我们前面的全部基于一个条件，那就是**轻节点的 root hash 是正确的**。

那么我们如何保证这个 root hash 一定是正确的而不是攻击者伪造的呢？

这就是我们区块链**去中心化**的核心力量：

> 我们的网络里有成千上万个全节点，轻节点会链接多个全节点。如果发现一个 root 和别的全节点 root 不一样，那么轻节点就会发现异常。

---

## 参考资料

- [轻节点与全节点的区别](https://zhuanlan.zhihu.com/p/7993733330)
- SHA-256 算法基础
