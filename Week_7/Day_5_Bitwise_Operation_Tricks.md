# Day 5: Bitwise Operation Tricks - Optimizing Math Calculations with shl, shr, and, or

## Introduction

Finally, we've reached Day 5! After mastering assembly memory and storage, today we're diving into the microscopic world of **bits**.

In high-level languages, we're accustomed to using `*` for multiplication and `/` for division. However, at the EVM (Ethereum Virtual Machine) level, mathematical operations are extremely expensive and consume a lot of gas. As optimization masters, we must learn to use **bitwise operations** to replace certain mathematical operations, thereby saving Gas.

## 1. Shift Operations: shl and shr

Let's start with the two most commonly used operations: `shl` and `shr`.

### 1.1 Decimal Analogy

Imagine working with base-10 numbers:

- If the number `1` shifts left by one position, we add a `0` on the right, making it `10` - the number is multiplied by 10
- If the number `1000` shifts right by one position, the entire number moves right, becoming `100` - the rightmost `0` is eliminated, and the number is divided by 10

The principle is the same in binary, except the multiplier changes.

### 1.2 Left Shift Operation: shl (Shift Left)

Let's use the simplest example: `3`

- Decimal: `3`
- Binary: `0011` (showing only the last 4 bits for convenience)

In Yul, the syntax for `shl` is `shl(x, y)`, as we've discussed.

**Example 1: `shl(1, 3)`**

Shift `3` left by one position:

```
Original: 0011 (3)
Shifted:  0110 (6)
```

After shifting left, it becomes `0110`, which in decimal is `2 + 4 = 6`.

Notice anything? It became `3 × 2`!

**Example 2: `shl(2, 3)`**

```
Original: 0011 (3)
Shifted:  1100 (12)
```

It becomes `1100`, which is `4 + 8 = 12` - that's `3 × 4`.

**Summary:**

```
shl(n, y) is equivalent to y × 2^n
```

Now you understand that when calculating `× 2`, `× 4`, etc., `shl` is much more Gas-efficient than the `mul` instruction.

### 1.3 Right Shift Operation: shr (Shift Right)

Since left shift multiplies by 2, let's look at right shift.

Let's use `12` as an example: `1100`

**Example 1: `shr(1, 12)`**

```
Original: 1100 (12)
Shifted:  0110 (6)
```

It becomes `0110`, which is `2 + 4 = 6`

**Example 2: `shr(2, 12)`**

```
Original: 1100 (12)
Shifted:  0011 (3)
```

It becomes `0011`, which is `2 + 1 = 3`

**Example 3: `shr(3, 12)`**

```
Original: 1100 (12)
Shifted:  0001 (1)
```

It becomes `0001`, which is `1`

**Summary:**

Each right shift divides by 2.

```
shr(n, y) is equivalent to y / 2^n (rounded down)
```

## 2. AND Operation: Efficient Modulo and Masking

Next, let's discuss the powerful `and` operation.

In Solidity optimization, `and` is often used to replace `mod` operations or for data masking.

### 2.1 AND Operation Rules

As we've discussed before, `and` returns `1` only when both corresponding bits are `1`; otherwise, it returns `0`.

You can think of it as a filter - only at positions allowed in your mask (`1`) will the original data show through.

### 2.2 Using AND to Replace Modulo Operations

Suppose we want to calculate `13 % 4`. The conventional approach would use the `mod` instruction, but we're experts, so we use `and`.

```
13 in binary: 1101
3 in binary:  0011

Performing the operation:
  1 1 0 1
& 0 0 1 1
---------
  0 0 0 1
```

The result is `1` - perfect!

**Summary:**

```
x % 2^n is equivalent to x & (2^n - 1)
```

> **Note**: The mask should be `2^n - 1`. In the example above, `13 % 4` should use `13 & 3` (because `4 = 2^2`, so the mask is `2^2 - 1 = 3`)

This is why in Ethereum's underlying implementation and many high-performance programming scenarios, we prefer using powers of 2 for data chunking or capacity limits - because we can use extremely fast and Gas-efficient `and` operations to replace expensive modulo operations.

## 3. OR Operation: Data Packing

Now that we have `shr`, `shl`, and `and`, let's discuss another operation: `or`.

This technique is about combination.

### 3.1 The Significance of Variable Packing

In Solidity, to save storage costs, we try to compress small numbers into a single slot - this process is called **variable packing**.

### 3.2 OR Operation Rules

The `or` rule is simple: if there's a `1` at any corresponding position, the result is `1`.

### 3.3 Packing Example

Let's explain with an example:

We want to pack two small numbers into one byte:

```
Number A = 3 = 0000 0011
Number B = 4 = 0000 0100
```

We want to place B in the high bits and A in the low bits.

First, we need to move B to the left (high bits), then use `or` to combine them:

```
Step 1: shl(4, B) = 0100 0000
Step 2: or(A, B_shifted)

  0 1 0 0 0 0 0 0  (B after left shift)
| 0 0 0 0 0 0 1 1  (A)
------------------
  0 1 0 0 0 0 1 1
```

First use `shl(4, B)`, then use `or(A, B_shifted)` to get the result above.

The result is `0100 0011`, which in decimal is `1 + 2 + 64 = 67`.

We've successfully compressed two numbers `3` and `4` into a single number `67`!

> **Correction**: The original text incorrectly stated "compressed into the number 37", but it should be `67`.

This means we can save one `sstore` operation, saving a significant amount of Gas!

### 3.4 Unpacking Operations

After packing, if we want to retrieve the original values, we need to unpack.

Now we've read `0100 0011` (i.e., `67`) from the contract.

**Extracting Low Bits (A):**

First, we need to restore A by taking only the last 4 bits.

We need to use `and` with `0000 1111` as a mask to extract the last 4 bits.

```
Using and(67, 15), because 0000 1111 in decimal is 15

  0 1 0 0 0 0 1 1
& 0 0 0 0 1 1 1 1
-----------------
  0 0 0 0 0 0 1 1
```

The extracted result is `0000 0011`, which is `3` - correct!

**Extracting High Bits (B):**

How do we extract the high bits?

We don't need `and` here - just shift the data right. A's value will be pushed away, and `0`s will be added on the left, directly giving us B's value.

```
Using shr(4, 67)

0100 0011 → 0000 0100
```

We get `0000 0100`, which is `4` - B's value is also retrieved!

**Summary:**

Now we've mastered the core of EVM data compression:

- **Packing**: Use `shl` to shift, use `or` to pack
- **Unpacking**: Use `and` to extract low bits, use `shr` to extract high bits

## 4. XOR Operation: Involution and Encryption

Let's cover one more thing: `XOR`, symbolized as `^`.

### 4.1 XOR Operation Rules

This is interesting - it returns `1` if the two bits are different, and `0` if they're the same.

### 4.2 Zero Law

In blockchain and cryptography, XOR has an important property: the **zero law**.

Any number XORed with itself always equals `0`.

### 4.3 Involution Example

Suppose we have a number `S = 9` (`1001`) and a key `K = 7` (`0111`).

**Encryption:**

```
Calculate S XOR K (9 ^ 7)

  1 0 0 1
^ 0 1 1 1
---------
  1 1 1 0
```

The value is `1110`, which in decimal is `14` - this is our ciphertext `C`.

> **Correction**: The original text incorrectly stated the result was `0001` (decimal `1`), but actually `9 ^ 7 = 14`.

**Decryption:**

Now, if we XOR our ciphertext C with K again, what do we get?

```
C XOR K (14 ^ 7)

  1 1 1 0
^ 0 1 1 1
---------
  1 0 0 1
```

We've restored `S`! This is XOR's **involution property**: `A ^ B ^ B = A`

### 4.4 Application Scenarios

In smart contracts, this property is typically used for:

1. Simple encryption
2. Generating random seeds
3. More importantly, it can swap two variables without using a temporary variable, saving a stack slot

## 5. Signed Integers and SAR Operation

### 5.1 The Problem with Negative Numbers

When we use `shr`, we treat all numbers as positive. Its rule is to always fill empty positions on the left with `0`.

However, at the Solidity and EVM level, negative numbers are implemented using **Two's Complement**, which simply means the highest bit of a negative number's binary representation is always `1`.

If we use `shr` on negative numbers, we'll encounter bugs - the highest bit will become `0`.

### 5.2 SAR Operation (Signed Arithmetic Shift Right)

Therefore, we have a special operation for handling negative numbers: `sar` (Signed Arithmetic Shift Right).

`sar` is specifically designed for signed integers.

When using `sar`, it sets a number in the first bit:

- If it was originally `0`, it writes `0`
- If it was originally `1`, it writes `1`

This ensures that positive numbers remain positive after division, and negative numbers remain negative.

### 5.3 Two's Complement Principle

Let's look at an example to understand how it works.

Assume we only have 8 bits for easier demonstration.

Suppose we have the number `-4`, represented in binary as `x = 1111 1100`.

Why is this?

In the world of two's complement, the highest bit has special weight - it represents the negative of that highest bit.

For example, in our example `1111 1100`, the highest bit represents `-128`.

Let's calculate: `4 + 8 + 16 + 32 + 64 - 128 = -4`! The result is correct.

### 5.4 SHR vs SAR Comparison

**Using shr:**

```
shr(1, x) = 0111 1110
```

The highest bit becomes `0`, turning it into a positive number! This is wrong.

**Using sar:**

```
sar(1, x) = 1111 1110
```

Now `sar` keeps writing `1` in the highest bit - completely correct.

`1111 1110` according to our calculation: `2 + 4 + 8 + 16 + 32 + 64 - 128 = -2`, the result is also correct.

### 5.5 Special Case: -1

Let's discuss `-1`: `1111 1111`

If we execute `sar(1, -1)`, what's the result?

Still `1111 1111`.

Because `sar` also rounds down, `-1 / 2 = -0.5` rounded down is still `-1`.

This ensures mathematical consistency - regardless of sign, `sar` is equivalent to dividing by `2^n` and rounding down.

## 6. Difference Between int and uint

Why do we need to learn about negative numbers?

Because Solidity has both `int` and `uint` types.

### 6.1 int8 (Signed Integer)

As we discussed above, the 8th bit is `-128`. If the first bit is `1`, the maximum value can only represent `-1`.

To represent positive numbers, the first bit must be `0`.

In `int8`, the maximum positive number is `127`. Want to store `128`? No way - it will overflow.

**Range: `-128` to `127`**

### 6.2 uint8 (Unsigned Integer)

The `u` stands for Unsigned. In this setting, we stipulate that negative numbers are not allowed.

So the first bit doesn't represent `-128`, it represents `+128`.

At this point, `1111 1111` becomes:

```
128 + 64 + 32 + 16 + 8 + 4 + 2 + 1 = 255
```

**Range: `0` to `255`**

Got it!

## Summary

In this lesson, we learned about bitwise operation optimization techniques in the EVM:

1. **shl/shr**: For fast multiplication/division by powers of 2
2. **and**: For modulo operations and data masking
3. **or**: For data packing
4. **xor**: For encryption and variable swapping
5. **sar**: For right-shifting signed integers

Mastering these techniques can significantly save Gas fees in smart contract development!
