# Today we study the Gas Golfing competition topics

Extreme Gas optimization is not just about saving money; it is also an excellent way to deeply understand the underlying mechanisms of the EVM. We will examine the code from the microscopic perspective of bytecode and opcodes, just like experts.

To make this practical session both deep and accessible, we start with simple examples and go deeper.

## Level 1: The Battle of Loops and Memory

First, let's write a baseline code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GasGolf {
    // Task: Reduce the Gas consumption of this function to the limit
    function sum(uint256[] memory nums) external pure returns (uint256 total) {
        for (uint256 i = 0; i < nums.length; i++) {
            total += nums[i];
        }
    }
}
```

There is nothing wrong with this logic, but it is extremely extravagant in terms of Gas consumption.

### 1. Memory vs Calldata

We can save a huge amount of gas by changing just one word. Do you know which one it is?

Here we need to review the two ways EVM handles data:

1. **Copy a copy: memory**
    When you use memory, the EVM copies the entire array passed in by the user into memory. Every number copied requires paying Gas; the larger the array, the more expensive the copying.

2. **Read directly: calldata**
    If you tell the EVM to read the original data directly, it doesn't have to spend money moving it; it just goes to read whichever one is needed. This saves expensive copying fees, especially for large read-only arrays.

Now back to the code: `function sum(uint256[] memory nums)`. You know where to change it, right?

Changing `memory` to `calldata` is the most immediate money-saving trick, eliminating the copying step.

Now our first line of code has evolved:

```solidity
function sum(uint256[] calldata nums) external pure returns (uint256 total)
```

### 2. Unchecked and Loop Optimization

Let's look at the loop code:

```solidity
for (uint256 i = 0; i < nums.length; i++)
```

In versions after 0.8.0, the compiler is very cautious. It automatically adds a **safety lock** to every operation to check if the number has become too large or if it will overflow. Each check costs a lot of gas.

Think about it: In this loop, increasing from `i = 0` until the array length, will it exceed the upper limit of `uint256`?

**Absolutely impossible**. Such a large array would cost an astronomical amount of GAS.
So checking it every loop is purely a waste of time, just like checking your ID card every time you go home.

In solidity 0.8+, we can use `unchecked` to tell the compiler: Don't worry, I promise this part won't exceed the limit. This can save a lot of Gas.

> **What does unchecked do?**
>
> * **Before Solidity 0.8.0**: If the odometer reaches 9999999 and drives another 1 km, it will directly become 0000000. This is called **overflow**. Although the number is wrong, the car can still drive, which was the source of many vulnerabilities in the past.
> * **Solidity 0.8.0+**: For safety, a safety lock is set. If you still want to drive after reaching 999999, it will stall the engine (revert) to prevent the array from becoming an unexpected result.
>
> **The cost?** Every time we do addition, it has to check if it has exceeded the limit, which consumes extra Gas.
>
> **The role of unchecked**: As a developer, when you can be 100% sure that this value will not explode, you can use `unchecked` to tell the compiler to remove that safety lock. I can handle the consequences myself, thus saving the checking money.

Now the loop looks like this:

```solidity
for (uint256 i = 0; i < nums.length; i++) {
    total += nums[i];
}
```

If we want to wrap `i++` with `unchecked`, we need to change the structure of the for loop. We need to move `i++` inside the loop body and then wrap it.

There is also a small detail. In Gas optimization, we usually use `++i` instead of `i++` because `i++` needs to save the old value before adding one in the underlying logic, while `++i` adds one directly, which can save a tiny bit of gas.

The code is as follows:

```solidity
for (uint256 i = 0; i < nums.length; ) {
    total += nums[i];
    unchecked { ++i; }
}
```

### 3. Caching Array Length

Now let's look at `i < nums.length` above the loop.

This code means that for every loop, the EVM has to read how long the `nums` array is. It's like asking the referee "Have I finished this lap?" every time you drive a small section in a car race. The efficiency is very low.

Although `calldata` is cheaper than reading memory, it is still much more expensive than reading the stack.
Let's try to store the length in a variable before the loop starts, so we don't have to read it every time.

So before the loop, we write:

```solidity
uint256 len = nums.length;
```

This way we don't need to find the length every time.

### 4. Loop Unrolling

Next, we are going to unroll the loop. This is a very brute-force method.

For example, when we are moving bricks:

* **Ordinary loop**: Move one brick at a time. 1000 bricks require 1000 moves.
* **Loop unrolling**: Move 10 bricks at a time. 1000 bricks only require 100 moves.

Although the total number of bricks hasn't changed, the number of trips you take has changed, significantly reduced.

So in our loop, if we process two elements at once, what changes will happen to the code?
The `total += ...` in the loop needs to be written twice, and `i` increases by 2 each time.

But there is a fatal trap here. We have to be careful about out-of-bounds issues, for example, array length `len = 5`:

* Round 1 (i = 0): Process nums[0] and nums[1]. ✅ No problem.
* Round 2 (i = 2): Process nums[2] and nums[3]. ✅ No problem.
* Round 3 (i = 4): Process nums[4] and nums[5]... ❌ **Crashed!**

Because `nums[5]` doesn't exist at all, the program will revert directly.
To prevent this directly from happening, we need to ensure `i + 1 < len`.

So the code is as follows:

```solidity
function sum(uint256[] calldata nums) external pure returns (uint256 total) {
    uint256 len = nums.length; 
    
    uint256 i = 0;
    // Loop unrolling: process 2 at a time
    for (; i + 1 < len; ) {
        total += nums[i];
        total += nums[i+1];
        unchecked { i += 2; }
    }
    
    // Handle the remaining one (if length is odd)
    if (i < len) {
        total += nums[i];
    }
}
```

After several steps of optimization, the gas consumption of this function has been greatly reduced! This is the aesthetic of expert-level code.

## Level 2: The Art of Storage Compression

Let's review the previous content, binary storage.
How much stuff can `uint32`, `uint256` store? How many bytes is it?

For example, let's take `uint32`:
The 32 in `uint32` refers to 32 bits.
A bit is the smallest unit in a computer, just 0 or 1.

In the computer world, 1 Byte = 8 bits. This is an iron law.
So calculate it:

$$32 \text{ Bits} \div 8 = 4 \text{ Bytes}$$

So `uint32` occupies 4 bytes, `uint16` occupies 2 bytes, and `uint256` occupies 32 bytes.

So how many numbers can it store?
Computers only recognize 0 and 1.

* If there is only 1 bit, you can store 2 numbers: 0, 1 ($2^1$)
* If there are 2 bits, you can store 4 numbers: 00, 01, 10, 11 ($2^2$)
* ...
* If there are 32 bits, how many numbers can you store?
    2 to the power of 32: $$2^{32} = 4,294,967,296$$

Because we count from 0, the maximum number `uint32` can represent is 4,294,967,295 (about 4.29 billion).

Next, let's understand it with an example. We want to design a contract:

* `isAdmin` (Is administrator)
* `isActive` (Is account active)
* `canMint` (Can mint)
* `canBurn` (Can burn)
* `lastUpdated` (Timestamp of last update)

You first write a set of variables to store these 5 pieces of data:

```solidity
bool isAdmin;      // 1 Byte
bool isActive;     // 1 Byte
bool canMint;      // 1 Byte
bool canBurn;      // 1 Byte
uint32 lastUpdated; // 4 Bytes
```

This is perfect. Can we change `uint32` to `uint256`?
Obviously not. If it becomes `uint256`, it occupies 32 bytes. 32 + 1 + 1 + 1 + 1 exceeds the range of one slot, and will use two slots, which is much more expensive.

To see clearly, please look at the table below:

| Byte Position (Bytes) | Variable Content |
| :--- | :--- |
| 0 | isAdmin |
| 1 | isActive |
| 2 | canMint |
| 3 | canBurn |
| 4 - 7 | lastUpdated |
| 8 - 31 | (Free, can stuff more things!) |

**Money-saving effect**: You only need to pay for SSTORE once to update these five variables at the same time. In Ethereum, it's equivalent to a 50% discount!
