# Implementing a Simple Blockchain: Mining and Hash Calculation

> Week Summary: Write a small chain capable of mining and hash calculation

---

## Part 1: Definition of Block

### What needs to be in a block?

First, let's think about what needs to be in a block?

| Attribute | Description |
|------|------|
| `data` | Transaction data (Mandatory) |
| `timestamp` | Timestamp (Knowing which block came first) |
| `previous_hash` | Hash value of the previous block (Forming a chain) |
| `index` | The position of the current block |
| `hash` | Hash value of this block |
| `nonce` | Random number (Will change during mining) |

### Role of Nonce

**Nonce** is a number. Why do we need a nonce?

When calculating a block, the hash is set with a condition that makes it difficult for people to achieve, so the value of nonce must be constantly changed until the hash of this block becomes a value that meets the conditions. This is the role of the nonce.

### Implementation of Block Class

```python
class Block:
    def __init__(self, index, previous_hash, data):
        self.index = index
        self.timestamp = time.time()
        self.data = data
        self.previous_hash = previous_hash
        self.nonce = 0  # Initial random number, changes during mining
        self.hash = self.calculate_hash()  # Calculate the fingerprint of the current block
```

---

## Part 2: Calculating Block Hash

### How to Calculate Hash?

When calculating the hash of this block, we need to add all other values initialized above for this block to the mix and use the SHA-256 function to calculate.

**Important**: We want to add these data, not add their values, but **concatenate** them, one after another. In Python, we use `str` to concatenate, and finally turn them into bytes for calculation.

### calculate_hash Method

```python
def calculate_hash(self):
    block_string = str(self.index) + str(self.timestamp) + str(self.previous_hash) + str(self.nonce) + str(self.data)
    return hashlib.sha256(block_string.encode()).hexdigest()
```

---

## Part 3: Mining - Proof of Work (PoW)

### What is Proof of Work?

In the BTC network, you can't just calculate any hash to package and publish. The network requires that a **specific hash** must be calculated. Usually, this hash value starts with a certain number of 0s.

### Difficulty Parameter

Now we write a `mine` function with a `difficulty` parameter, representing that the hash starts with `difficulty` number of 0s.

Now we find that other values are fixed, only `nonce` can be changed. This is the role of nonce, very important.

### mine_block Method

```python
def mine_block(self, difficulty):
    if self.hash is None:
        self.hash = self.calculate_hash()

    while self.hash[:difficulty] != difficulty * '0':
        self.nonce += 1
        self.hash = self.calculate_hash()
```

**Working Principle**:

1. Check if the first `difficulty` characters of the hash are all '0'
2. If not, increase nonce and recalculate hash
3. Repeat until a hash meeting the condition is found

---

## Part 4: Implementation of Blockchain

### Connecting Blocks into a Chain

So far, we have completed the definition of a simple block. Now that our building blocks are ready, we need to connect these blocks into a chain.

### Blockchain Storage

We need to create a `Block_chain` class to manage all of this. How do we store these blocks? Is it one after another? In our simple blockchain, we use Python's `list` to implement.

### Genesis Block

Now let's think, we create the first block, usually called the **Genesis Block**. What should our `previous_hash` be? We can just define it as `'0'`.

We also have to set the block generation difficulty, which is the `difficulty` mentioned above.

### Initialization of Block_chain Class

```python
class Block_chain:
    def __init__(self):
        self.blocks = []
        self.difficulty = 4
        self.blocks.append(Block(0, '0', 'hello world'))
```

---

## Part 5: Adding Blocks to the Chain

### Logic of add_block Method

Okay, now we want to make this chain move, that is, implement the `add_block(self, data)` method. To add a block, we need to know the hash value of the previous block and its own index.

**How to find the last element of a list in Python?** Use `list[-1]`.

**How to find its own index?** Use `len(list)`.

### Logic Steps

1. Get the hash value of the previous block
2. Create a new block, write its index, write previous_hash
3. Mine, calculate a nonce to make the hash value of the entire block meet requirements
4. Add to chain, append to the list of the chain

### add_block Method

```python
def add_block(self, data):
    previous_hash = self.blocks[-1].hash
    block = Block(len(self.blocks), previous_hash, data)
    block.mine_block(self.difficulty)
    self.blocks.append(block)
```

---

## Part 6: Prevent Tampering - Chain Verification

### Core of Blockchain

We can now have a complete chain. So the core of blockchain is **Tamper Resistance**. What if someone changes the transfer of 10 yuan to 100,000 yuan in it? We must write a detection function to ensure that the block data in the chain has not been tampered with.

### Two Steps of Verification

1. **Data Self-check**: Recalculate the hash value of the current block to see if it is consistent with the stored `self.hash`. If not consistent, the data has been tampered with.

2. **Link Check**: Check if the hash of the previous block is the `previous_hash` of this block. If they are not equal, it means there is a problem with the chain.

### is_chain_valid Method

```python
def is_chain_valid(self):
    for i in range(1, len(self.blocks)):
        current_block = self.blocks[i]
        previous_block = self.blocks[i - 1]

        # Data Self-check
        if current_block.calculate_hash() != current_block.hash:
            return False

        # Link Check
        if current_block.previous_hash != previous_block.hash:
            return False

    return True
```

---

## Complete Source Code

```python
import hashlib
import time

class Block:
    def __init__(self, index, previous_hash, data):
        self.index = index
        self.timestamp = time.time()
        self.data = data
        self.previous_hash = previous_hash
        self.nonce = 0  # Initial random number, changes during mining
        self.hash = self.calculate_hash()  # Calculate the fingerprint of the current block

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
    
    # Try to tamper with data
    blockchain.blocks[1].data = "Hacked Data"
    if blockchain.is_chain_valid():
        print('Block chain is valid')
    else:
        print('Block chain is invalid')
```

---

## Summary

At this step, a blockchain containing the following features is completed:

✅ **PoW Mining** - Proof of Work Mechanism

✅ **Hash Chain** - Connecting blocks through hash values

✅ **Tamper-proof** - Any data modification will be detected

This is the **Underlying Architecture of BTC and All Cryptocurrencies**!

---

## Test Results

Run the code above, and you will see:

```
Block chain is valid
Block chain is valid
Block chain is invalid
```

The last one is `invalid` because we tampered with the data of the second block, and the chain's verification mechanism detected this tampering!
