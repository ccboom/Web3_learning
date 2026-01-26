# Handwriting Merkle Tree (Python)

> Understanding its role in light node verification

---

## Part 1: Basic Concepts of Merkle Tree

Yesterday we learned the SHA-256 algorithm. Today we need to learn Merkle Tree.

### What is Merkle Tree?

Merkle Tree is a structure that compresses a huge amount of data into a single hash, utilizing yesterday's **avalanche effect**—any tiny change will propagate from the bottom all the way up the tree, causing the final root node to change.

### Basic Construction Process

Let's use the classic four blocks to deduce. Suppose we now have data blocks `[A, B, C, D]`:

1. These four blocks are **leaf nodes**. When we upload them, we should not upload the original values, but their hash values.
2. Merkle Tree is a **binary tree**, so now we need to pair them up two by two, merge them into a parent node, and then pair again.

Formula:

$$H_{AB} = Hash(H_A + H_B)$$

Now, we have obtained the hash values of `H_AB` and `H_CD`, and then use the formula again:

$$H_{root} = Hash(H_{AB} + H_{CD})$$

We can calculate the hash value of the root!

### Processing Odd Nodes

Then a new problem arises. If we have five values `[A, B, C, D, E]`, who does E pair with?

Someone surely wants to say pair E with `H_CD`, but this **won't work**.

We are currently using a binary tree, so we should try to use binary pairing as much as possible. If there is always one left over at the end, it will pair with the layer above the root hash, which is very dangerous for encryption.

**Solution**: We clone another E into it, becoming `[A, B, C, D, E, E]`, and let it pair with itself, so we can ensure a neat binary tree.

### Formal Definition

Merkle Tree is also called **Hash Tree**. It is a data structure built using hash values.

We can imagine it as an upside-down tree:

![Merkle Tree Structure Diagram](https://github.com/user-attachments/assets/88340418-86af-42f1-b741-fc8f2c4b7e27)

- **Leaves**: The bottom layer, which are the hash values of the original data.
- **Branches**: The middle layer, which are the merged hashes of two leaf hash values.
- **Root**: The very top, which is the hash value of everything added together, the final result.

### Core Functions

1. **Data Compression**: Can condense a very large amount of data into a single hash value.
2. **Prevent Tampering**: If one of the hash values below is changed, the final root value will change drastically (avalanche effect).
3. **Lightweight Verification**: If we want to know if A is in this tree, we don't need to download all the data. We only need to know the pairing hash values on its path to calculate the root hash, and then compare it with the root hash.

---

## Part 2: Code Implementation

### 1. Hash Function

First, we need to think about what our initial data should be—right, it is the hash values of transactions. Let's first write the following code:

```python
@staticmethod
def hash_data(data):
    # 1. Encode to bytes
    encoded_data = str(data).encode('utf-8') 
    # 2. Calculate SHA-256 and return hexadecimal string
    return hashlib.sha256(encoded_data).hexdigest()
```

This code allows us to turn an input data into a hash value. To be able to handle integers, we added `str()` to turn it into a string.

### 2. Initialization Function

Next, we start writing the initialization function. We usually have this function when defining a class. We need to store the hash values of all transactions during initialization, and we also need to build this tree during initialization using the `build()` function:

```python
def __init__(self, data_list):
    self.data_list = data_list
    # Initialize the bottom layer (leaves)
    self.leaves = [self.hash_data(i) for i in data_list]
    # We use a list to save all levels to facilitate verification later
    # index 0 is the leaf layer, the last index is the root node
    self.levels = [self.leaves]
    self.build()
```

### 3. Build Function

So what is the content of the `build` function? It's essentially building up, layer by layer.

First, we take out all the hash values of our bottom layer, calculate the hashes two by two, pass them to the upper layer, and continue this operation until there is one hash left, which is our root:

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
                # Odd case, replicate the last node
                right_node = left_node
            
            # Parent Node Hash = Hash(Left Child + Right Child)
            combined_hash = self.hash_data(left_node + right_node)
            next_level.append(combined_hash)
        
        self.levels.append(next_level)
        current_level = next_level
        
    self.root = current_level[0]
    return self.root
```

### 4. Complete Code

Finally, we have built a Merkle Tree. The total code is as follows:

```python
import hashlib

class MerkleTree:
    def __init__(self, data_list):
        self.data_list = data_list
        # Initialize the bottom layer (leaves)
        self.leaves = [self.hash_data(i) for i in data_list]
        self.levels = [self.leaves] 
        self.build() 

    @staticmethod
    def hash_data(data):
        # 1. Encode to bytes
        encoded_data = str(data).encode('utf-8') 
        # 2. Calculate SHA-256 and return hexadecimal string
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
                    # Odd case, replicate the last node
                    right_node = left_node
                
                # Parent Node Hash = Hash(Left Child + Right Child)
                combined_hash = self.hash_data(left_node + right_node)
                next_level.append(combined_hash)
            
            self.levels.append(next_level)
            current_level = next_level
            
        self.root = current_level[0]
        return self.root

# Test Code
data = ['A', 'B', 'C', 'D']
mt = MerkleTree(data)
print(f"Merkle Root: {mt.root}")
```

Running it will get a hexadecimal string, which is the **Merkle Root**!

---

## Part 3: Light Node Verification (Merkle Proof)

> If you don't know the difference between light nodes and full nodes, you can check: [Light Nodes vs Full Nodes](https://zhuanlan.zhihu.com/p/7993733330)

### Scene Description

For example, we are now a **light node**, and we only have the final root.

Now someone tells you: "A is in this block."

But you don't know if it's true. To verify this, you tell that person that you must give me a **verification path** to prove that this A is in this block.

### Information Required for Verification

For example, let's assume we have `['A', 'B', 'C', 'D']`, then we need a Merkle Proof to verify whether A is really in the block. What hash values do we need?

1. **Hash value of A**: This can be calculated directly from the original data they gave us.
2. **Position of A in this block**: The reason we must know the position is that `hash(A + B)` is completely different from `hash(B + A)`, so we need to know the correct position.
3. **Pairing hash values on the path**: We also need to know every pairing hash value with us from the leaf node up, so that we can calculate the root hash from the path.

### Get Proof Path

Okay, we ask the full node to prepare these three things for us, then the following code can help us get what we want:

```python
def get_proof(self, index):
    proof = []
    for level in self.levels[:-1]:  # Does not include root node layer
        # 1. Find Sibling
        if (index % 2) == 0:
            sibling_index = index + 1
        else:
            sibling_index = index - 1

        # 2. Handle odd case (point to itself)
        if sibling_index >= len(level):
            sibling_index = index

        # 3. Get Evidence
        sibling_hash = level[sibling_index]
        proof.append(sibling_hash)

        # 4. Climb up one layer
        index = index // 2
    
    return proof
```

### Verify Proof

Okay, now that we have what we want, the light node can write a function to verify if what he gave us is correct:

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

## Summary

We have written all the core components of the Merkle Tree:

| Function | Description |
|------|------|
| **Build** | Build a bottom-up tree to generate root |
| **Generate Proof** | Full node provides path support for a specific data item |
| **Verify** | Light node compares the proof with its own root |

---

## Last Question

Everything we did before is based on one condition, that is **the light node's root hash is correct**.

So how do we guarantee that this root hash must be correct and not forged by an attacker?

This is the core power of our blockchain's **decentralization**:

> There are thousands of full nodes in our network, and light nodes will connect to multiple full nodes. If it finds that one root is different from other full nodes' roots, the light node will detect the anomaly.

---

## Reference Materials

- [Difference between Light Nodes and Full Nodes](https://zhuanlan.zhihu.com/p/7993733330)
- SHA-256 Algorithm Basics
