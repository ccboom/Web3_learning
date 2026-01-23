# W7D3: Deep Dive into Slot Packing

## Introduction

Today we're going to tackle something more challenging: **Slot Packing** technique.

## I. Memory vs Storage Review

### Memory
Memory is like a **blackboard** in a classroom:
- You use it for calculations
- It gets erased when you're done
- **Low cost**, use and erase as needed

### Storage
Storage is like a **permanent safe deposit box**:
- Items stored are kept permanently
- Even 100 years later, nodes will still store your data
- As long as there are nodes running

## II. Ethereum Storage Rules

### Core Rules

1. **Fixed Size**: Each storage slot has a fixed size
   - Exactly **32 bytes**, no more, no less
   - Equals **256 bits**

2. **Expensive Operations**:
   - `sstore` (write) requires network-wide synchronization, costs **22,100 gas**
   - `sload` (read) is also much more expensive than memory operations, costs **2,100 gas**

3. **Per-Slot Pricing**:
   - Whether you store a piece of paper or a pound of gold, the cost is the same
   - As long as you occupy a new slot, Ethereum charges you for one slot

### What is Slot Packing?

**Slot Packing** is the technique of cramming as much data as possible into a single storage slot to save on costs.

**Example**:
- You have two small items of type `uint128`
- Put them in the same slot
- **Benefits**:
  - Cheaper to store
  - Can retrieve both at once
  - Kill two birds with one stone!

## III. Data Packing at the Yul Level

### The Challenge

How do we pack data at the Yul level?

### Solidity's Storage Strategy

Solidity is very smart. It stores data like humans do, filling slots **from right to left**, i.e., **from low bits to high bits**.

This is similar to how we write numbers:
- We write numbers from large to small
- Writing a `1` represents the ones place
- Writing a `2` to the left of `1` makes `21`, representing the tens place
- And so on

### Bit Representation in Computers

In the computer world:
- The **rightmost** position represents the **low bits** (Least Significant Bits, LSB), the smallest value
- To save GAS, Solidity tries to pack things towards the low end, i.e., the **right side**
- Like a box for shuttlecocks - when you put them in vertically, they automatically fall to the bottom

### Example Contract

Suppose we have this contract:

```solidity
contract Packing {
    uint64 public a;
    uint64 public b;
    uint128 public c;
    // 64 + 64 + 128 = 256, exactly fits in slot 0
}
```

In the EVM's view, **Slot 0** looks like this:

```
┌─────────────┬─────────────┬─────────────┐
│  c (128bit) │  b (64bit)  │  a (64bit)  │
│   Left      │   Middle    │   Right     │
└─────────────┴─────────────┴─────────────┘
```

- Variable `a` is placed on the **rightmost**
- Variable `b` is in the **middle**
- Variable `c` is on the **leftmost**

## IV. Three Essential Bit Operations

To truly understand `sload` and `sstore` bit operations, we need to solve a problem: **How do we separate mixed data?**

For example, in Yul we write:
```yul
let data := sload(0)  // Read all data from Slot 0
```

Now `data` contains all three values: `a`, `b`, and `c`.

**Question**: How do we extract only the value of `b`?

**Answer**: Use opcodes to extract `b` from `data`.

### 1. Mask

**Analogy**: Like a transparent glass plate

Imagine this scenario:
- We have a 256-bit paper strip, simulating a slot
- We take a transparent glass plate, like applying a filter to the paper strip

**Rules**:
- At positions with `1`, the glass is **transparent**, you can see the data directly, so data is **preserved**
- At positions with `0`, the glass is **blacked out**, you can't see anything, so it shows only `0`

**Effect**:
- Preserve original values at `1` positions
- Turn data to `0` at `0` positions

### 2. SHL and SHR (Shift Operations)

These are abbreviations:
- `shl` = **Shift Left**, push to the left
- `shr` = **Shift Right**, push to the right

#### SHL Example

```yul
shl(x, y)  // Push y to the left by x positions
```

**Example**:
- Original data is `0011 0001`
- Use instruction `shl(4, 00110001)`
- The left `0011` is pushed away and disappears
- The original right `0001` position becomes empty and is filled with `0`
- Result becomes `0001 0000`

#### SHR Example

`shr` works the same way, but pushes data to the right.

### 3. AND (Bitwise AND)

```yul
and(mask, data)
```

Use the **mask** to cover the `data`, then cut away unwanted data - positions not covered by the mask are set to `0`.

**Example**:
- `mask` is `0000 1111`
- `data` is `1111 0101`
- Use `and(mask, data)`
- In `mask`, `1` means preserve, `0` means overwrite with `0`
- Result: `data` becomes `0000 0101`

## V. Read Operation: Extracting Variable b

With these three opcodes, we can solve the problem!

### Step 1: Right Shift (SHR)

To extract `b`, since it's in the middle, we need to push away `a` on the right first, so `b` is at the rightmost position.

```yul
let data_s = shr(64, data)  // Since a occupies 64 bits, we shift right by 64
```

Now `data_s` becomes: `[0...0 | c | b]`

### Step 2: Masking (AND)

Then cut away the unwanted data on the left using `and`.

Since `b` is 64 bits, we need 64 `1`s to ensure it's not overwritten.

**Methods to Generate Mask**:

Method 1: Direct hexadecimal
```yul
0xFFFFFFFFFFFFFFFF  // 16 F's, each F represents 4 ones
```

Method 2: Bit shift trick (recommended)
```yul
(1 << 64) - 1
```

**Why does this work?**

This is a classic technique to quickly generate a sequence of `1`s.

Example:
- `1000` is actually `1 << 3`, `1` is in the fourth position
- Subtract `1`, get `0111`
- Successfully turned 3 bits into `1`s

Therefore:
- `1 << 64` is `1` followed by 64 `0`s
- Subtract `1` and you get one `0` followed by 64 `1`s

Simple, right!

### Complete Code

```yul
let b := and((1 << 64) - 1, shr(64, data))
// c's position is 0, so it's overwritten
// Only the last 64 bits remain, which is b's value
```

Merged into one formula:
```yul
let b := and((1 << 64) - 1, shr(64, data))
```

Done! ✅

## VI. Write Operation: Modifying Variable b

### Why is Writing More Complex?

Reading is complicated but doesn't destroy data. **Storage is the real gas killer**.

Now suppose we continue with the contract above and want to change `b`'s value to `new_value` without changing `a` and `c`.

### EVM's SSTORE Behavior

EVM's `sstore(key, value)` is very brutal - if used directly, it will **overwrite all 256 bits of the slot**.

If you write:
```yul
sstore(0, new_value)
```

Then the entire `slot 0` will only have `new_value`, and both `a` and `c` will disappear!

### Correct Write Process

To modify properly, we need a process:

```
Read first → Modify → Write back
```

**Analogy**: Imagine a classroom blackboard with some data on it. When writing, we need to erase `b`'s value first, then write the new value. Otherwise, writing directly on top will create overlapping, illegible data.

### Detailed Steps

#### Step 1: Read the Entire Value

```yul
let data := sload(0)
```

#### Step 2: Modify (Erase + Write)

We must have an **inverted mask** to preserve data, like this:

```
┌─────────────┬─────────────┬─────────────┐
│  11...1     │  00...0     │  11...1     │
│  c (keep)   │  b (erase)  │  a (keep)   │
└─────────────┴─────────────┴─────────────┘
```

**Generate Inverted Mask**:

We just generated a mask `(1 << 64) - 1` that keeps the rightmost 64 bits intact.

Now we need to:
1. Move these 64 bits to `b`'s position
2. Then invert it, keeping other positions intact while overwriting `b` in the middle

**NOT Instruction**:
- `not(x)` turns `0` into `1` and `1` into `0` in `x`, i.e., inversion

**Code**:
```yul
let mask := (1 << 64) - 1              // 00...00 | 00...00 | 11...11
let mask_not := not(shl(64, mask))     // 11...11 | 00...00 | 11...11
```

Two operations here:
1. Shift left (`shl`)
2. Invert (`not`)

Now `mask_not` is `[11...1 | 00...0 (b) | 11...1]`, where `0` occupies `b`'s 64-bit position.

**Erase b's value**:
```yul
let data_s := and(mask_not, data)
```

Now `data_s` becomes:

```
┌─────────────┬─────────────┬─────────────┐
│  c (128bit) │  00...00    │  a (64bit)  │
│   Left      │   Middle    │   Right     │
└─────────────┴─────────────┴─────────────┘
```

Using the mask, we preserved `a` and `c`'s values, and `b`'s value was overwritten.

#### Step 3: Insert New Value

Next, we need to insert the new value to complete the assembly.

`new_value` is initially at the rightmost position, so we need to use `shl` to shift it left to align with `b`'s position:

```yul
let shifted_new_value := shl(64, new_value)
```

Now it becomes: `[00...00 | new_value | 00...00]`

We now have two paper strips:
```
Strip 1:  c |     0     | a   // Middle transparent, sides have data
Strip 2:  0 | new_value | 0   // Sides transparent, middle has data
```

Now we need to overlay these two to generate the complete new strip.

**OR Instruction**:

In bit operations, to **merge** two non-conflicting data pieces, we typically use the `or` instruction.

It's like overlaying two paper strips - transparent areas are covered by opaque areas from the other strip.

```yul
// 3. Merge: overlay the two strips, merge opaque areas
let result := or(data_s, shifted_new_value)

// 4. Write: put the value back into the storage slot
sstore(0, result)
```

**Mission accomplished!** 🎉

## VII. Complete Code Examples

### Reading Variable b

```yul
let data := sload(0)
let b := and((1 << 64) - 1, shr(64, data))
```

### Modifying Variable b

```yul
// 1. Read
let data := sload(0)

// 2. Erase b's value
let mask := (1 << 64) - 1
let mask_not := not(shl(64, mask))
let cleared_data := and(mask_not, data)

// 3. Prepare new value
let shifted_new_value := shl(64, new_value)

// 4. Merge
let result := or(cleared_data, shifted_new_value)

// 5. Write
sstore(0, result)
```

## Summary

Through this article, we learned:

1. The difference between **Memory vs Storage**
2. The principles and advantages of **Slot Packing**
3. **Three essential bit operations**: Mask, SHL/SHR, AND
4. **Read operations**: How to extract individual variables from packed slots
5. **Write operations**: How to modify individual variables without destroying other data

Master these techniques, and you can implement efficient gas optimization at the Yul level! 💪
