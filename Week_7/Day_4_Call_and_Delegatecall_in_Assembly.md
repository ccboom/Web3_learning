# Day 4: Call Operations - Call and Delegatecall in Assembly

## Overview

Today we'll focus on assembly-level calls, which can be quite confusing.

In Yul, the difference between `call` and `delegatecall` is: **Where is the code executed, and whose resources are used?**

We covered this before, so let's review it again.

## Core Concepts Review

Imagine you're the general manager of a factory:

### 1. call: Regular Call

A customer sends you (Contract A) an order for a part, but you're too busy to handle it.

So you call the factory next door (Contract B) and say: "Here's the money, here's the order, please make this for me."

The materials used are from the neighboring factory, the workspace is also from the neighboring factory, and when it's done, the neighboring factory will tell you it's complete.

**This is equivalent to doing work in B's territory - the execution context has changed.**

### 2. delegatecall: Delegated Call

A customer sends you an order for a part, but you don't know how to make it.

So you invite workers from the neighboring factory (the logic from Contract B) to come to your factory.

They use your own factory's materials and workspace to make the part.

Although the workers are from next door, they're consuming your own materials.

**This is equivalent to doing work in your own factory - the execution context remains in your own factory and hasn't changed.**

## Parameter Breakdown

Now that we've reviewed the basics, let's look at today's content. First, let's examine their parameter lists:

### call Parameters

```solidity
call(g, a, v, in, insize, out, outsize)
```

- `g`: Gas - how much gas to use
- `a`: Address - target address
- `v`: Value - how much ETH to transfer
- `in`: starting position of input data in memory
- `insize`: size of input data
- `out`: where to store output data
- `outsize`: size of output data

### delegatecall Parameters

```solidity
delegatecall(g, a, in, insize, out, outsize)
```

The parameters are almost the same, except it's missing `v`.

### Why is there no Value parameter?

Think about what we just reviewed:

- When using `call`, we need to pay the neighboring factory (gas fee), and then pay for the processing materials. Since the money from the order is with you, you need to transfer money to the neighboring factory (with value).
- When using `delegatecall`, we invite workers from the neighboring factory to our own factory. We only need to pay them wages (gas). The money from the order is still with you, and the neighboring factory's workers can use the money from the order to directly purchase materials, so there's no need to bring value.

## Context Differences

In `call`, `msg.sender` becomes Contract A, but in `delegatecall`, `msg.sender` remains the original sender.

## Storage Slot Collision Risk

This also poses risks for storage - this is actually the storage slot collision we discussed in W5D1.

**The EVM only recognizes slots, not variable names.**

### Example Scenario

If Contract A has two variables:
- `slot 0`: owner
- `slot 1`: balance

Contract B has two variables:
- `slot 0`: balance
- `slot 1`: hash

Now when we ask Contract B (the neighboring workers) to operate on our variables, and we tell them to operate on balance: because Contract B doesn't know your variable names, it only knows its own variables are in `slot 0`.

So it puts the calculated value into `slot 0`, overwriting our `owner` variable! **Permanently losing control of the contract**.

This is **storage collision** - remember it now?

### Rules to Prevent Storage Collision

To prevent Contract B from overwriting Contract A's `owner` variable, we need to establish rules for these two contracts:

If A and B must share the same bookshelf, then when writing the variable order, if we want to add a variable to B, **it must be written at the end of the variable list** to ensure the integrity of A's variables.

## RAW Calls

Next, let's talk about RAW calls.

### What is RAW?

RAW here means: **no decoration, no preprocessing, directly operating on binary data**.

Let's make an analogy:

Think of it like sending a package:

- **When calling Solidity**: You take what you want to send to the courier station, and the courier (compiler) will help you pack, label, and tag everything. You don't need to worry about how things are arranged in the box - you just hand it to the courier and it's done.

- **Raw Call**: You pack it yourself. You need to get a box (allocate memory), put things in (write values), you must measure the size yourself (calculate data size), and then put this sealed box on the truck. This is RAW - you must manually handle every step (byte).

### Code-Level Comparison

Let's see the difference at the code level:

#### A. Solidity Call

```solidity
token.transfer(0x123...9, 100)
```

The compiler silently does auto packing in the background:
1. Look up the function ID for `transfer`: `0xa9059cbb`
2. Format `0x123...9` as 32 bytes
3. Format `100` as 32 bytes
4. Pack and send

#### B. Raw Call

No compiler to pack for you - you must manually handle these bytes:

```solidity
assembly {
    // Manually write function ID at the beginning
    mstore(0x00, shl(224, 0xa9059cbb))
    // Manually write first parameter (address)
    mstore(0x04, 0x123...9)
    // Manually write second parameter (amount)
    mstore(0x24, 100)
    // Tell EVM: starting from 0x00, take 68 bytes and send
    // This is Raw Call - you control every byte!
    let result := call(gas(), address, 0, 0x00, 68, 0, 0)
}
```

## Practical Application: Extreme Gas Optimization Tricks

Now that we understand the principles, let's look at practical applications. For extreme gas savings, we have some clever tricks:

```solidity
assembly {
    // 1. Prepare input data
    // Get the data (args) at memory pointer 0x40
    let ptr := mload(0x40)

    // 2. Execute delegatecall
    // Parameters: gas, target address, input data start, input length, output start, output length
    let result := delegatecall(gas(), _impl, ptr, calldatasize(), 0, 0)

    // 3. Handle result (this is a big gotcha!)
    // ...
}
```

### Why are both output start and output length 0?

This is counter-intuitive - why are both the output start and output length 0?

You ask the neighboring workers to make a part, and after they finish, you tell them there's no place to put it (output space is 0). Would they just throw the part in the trash?

Obviously not - they'll put it in a box (**Returndata Buffer**).

### Returndata Buffer Mechanism

In the EVM, there's a very special temporary storage area: `returndata`.

**Regardless of whether you set output space in the call instruction, the EVM will automatically put the return result here first.**

This box is **outside of Memory** - it's independent.

Because as a proxy contract, you have no idea how much space the part will take:
- If it's a 5cm part, you only need a small box
- If it's a 5m part, you need a large crate

If you blindly guess and allocate space before calling, you'll either waste space or not have enough.

### Correct Process

1. **Blind call**: First call with `0, 0` space, let the workers finish and put it in the temporary box
2. **Query size**: After completion, ask the EVM how big the contents of the box are. Instruction: `returndatasize()`
3. **Allocate memory**: Based on the size just obtained, allocate a block of memory that's just enough
4. **Transfer**: Move things from the temporary box to memory. Instruction: `returndatacopy(memory start position, box start position, data length)`

## Ultimate Code Example

Let's write an ultimate code example to demonstrate:

```solidity
assembly {
    // Copy input data (calldata) to memory, prepare for workers
    // Details of calldatacopy are omitted here, assuming it's ready
    let ptr := mload(0x40)
    calldatacopy(ptr, 0, calldatasize())
    
    // 0, 0: do it first, then put it in the temporary box after completion
    let result := delegatecall(gas(), _impl, ptr, calldatasize(), 0, 0)
    
    // Query the data size in the box
    let size := returndatasize()
    
    // Move data from box to memory
    // Copy size-length data starting from 0 in the box to memory at ptr position
    returndatacopy(ptr, 0, size)
    
    // Decide whether to report based on success or failure
    switch result
    case 0 {
        revert(ptr, size)
    }
    default {
        return(ptr, size)
    }
}
```

Done!

## Summary

- The core difference between `call` and `delegatecall` lies in the execution context
- `delegatecall` doesn't need a value parameter because it executes in the caller's context
- Storage slot collision is a risk that requires special attention when using `delegatecall`
- RAW calls provide maximum flexibility and gas optimization opportunities
- The Returndata Buffer mechanism allows dynamic handling of return data, avoiding memory waste
