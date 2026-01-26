# Introduction to Public Key Cryptography: Differences Between RSA and ECC (Elliptic Curve)

---

## Part 1: Trapdoor Function

### What is a Trapdoor Function?

Simply put: **Calculation is very simple, but reverse calculation is extremely complicated**. A function operation that is basically impossible to complete without special information. This is the design philosophy of trapdoor functions.

### Relationship between Trapdoor Functions and RSA/ECC

- **Trapdoor Function** = Soul (Design Philosophy)
- **RSA/ECC** = Body (Implementation Method)

Now that we know the relationship between the two, let's look at what RSA and ECC are respectively.

---

## Part 2: Core Principles of RSA and ECC

### RSA: Large Integer Factorization

The core of RSA is actually **Large Integer Factorization**. We can imagine it as **Paint Mixing**:

1. **Mixing Paints**: Now there are two colors of paint (two large prime numbers, forming the private key). We pour these two paints into a bucket, stir them, and get a new color (which is the public key).

2. **Reverse Difficulty**: Now that we have the mixed paint, if you are asked which two colors formed it? It is basically impossible to calculate.

3. **Private Key Advantage**: However, if you have one of the original colors (private key), then using chemical or physical methods, it is relatively easy to deduce the other color.

**Summary**: RSA relies on **irreversibility caused by destructive structure** (fusing two large numbers into one).

### ECC: Elliptic Curve Discrete Logarithm

The core of ECC is the **Elliptic Curve Discrete Logarithm Problem**. We can imagine it as **playing billiards on a weird table**:

1. **Forward Calculation**: We start hitting the ball from point G. The ball bounces around following a special rule (elliptic curve equation). Assuming it bounces k times (k is the private key), then it is very easy to know which point is the final landing point (Point P, which is the public key).

   ```
   Hit once: Ball at point A
   Hit twice: Ball at point B
   ...
   Hit k times: Ball at point P
   ```

2. **Reverse Difficulty**: Now tell you the starting point G and the ending point P of the ball. The question is: How many times did I hit it just now? What is k?

   On this billiard table, the trajectory of the ball is extremely chaotic and irregular. It is unrealistic to simply backtrack from point P. You can only clumsily try one by one from 1 until you find point P.

3. **Trapdoor**: The trapdoor here is the number of times K. Even if you know the starting point and the ending point, for Bill, it is impossible to know how many bounces it went through in between.

**Summary**: Forward calculation is very fast, but reverse derivation is extremely difficult, and one can only try like a headless fly. This is called the **Discrete Logarithm Problem (DLP)** in mathematics.

---

## Part 3: Mathematical Formulas

### RSA: Large Integer Factorization

RSA relies on the multiplication we learned in elementary school, but the numbers are extremely large.

**Making a Lock**: Take two super large prime numbers P and q (private key), multiply them to get N (part of the public key).

$$N = p \times q$$

**Trapdoor**: The whole world knows N, but no one can easily split it into p and q.

### ECC: Elliptic Curve Discrete Logarithm

ECC relies on point addition operations on the billiard table.

**Making a Lock**: Choose a base point G and an integer k (private key). Do scalar multiplication k times on this curve to get the end point P (public key).

$$P = k \cdot G$$

> Note: The $\cdot$ here represents scalar multiplication on the elliptic curve, not ordinary multiplication.

**Trapdoor**: The whole world knows G and P, but no one can calculate what k is.

---

## Part 4: Key Differences Between the Two

### Difference in Cracking Resistance

Although both look like some kind of product of two numbers, the cracking resistance is completely different.

Let's assume RSA's N is 77, you will quickly calculate it is the product of 7×11, but now RSA standards usually require N to have at least 2048 bits.

**Question**: Based on your understanding of computers, do you think hackers are helpless if the number is large enough? Or is the cost of attack too high, making cracking meaningless?

**Answer**: Obviously the latter. We are betting that attackers don't like to spend equipment, time, and electricity bills to crack it because the value is too low.

### The Biggest Difference in Mathematical Properties

#### RSA's Weakness

Mathematicians are too smart; they have already found some algorithmic shortcuts, such as the famous **"General Number Field Sieve" (GNFS)**. This doesn't mean it can crack it in a second, but it is much faster than trying one by one from 1.

**Metaphor**: It's like looking for salt on a lawn. Although it's hard to find, attackers invented a filter that can filter out more than half of the grass, making it easier to find.

#### ECC's Advantage

Currently, for the elliptic curve discrete logarithm, the mathematical community has not yet found efficient shortcuts. Attackers facing ECC can only honestly try one by one.

**Metaphor**: There is no filter on the lawn, and you can only look for it one by one.

### Huge Difference in Engineering Applications

This leads to huge differences in their engineering applications.

When we use RSA, we have to consider the existence of filters, while using ECC does not execute this need. So to achieve the same security level, **RSA's bit length must be much longer than ECC's**:

| Security Strength | RSA Key Length | ECC Key Length |
|---------|------------|------------|
| 80 (No longer safe) | 1024-bit | 160-bit |
| 112 (Basically safe) | 2048-bit | 224-bit |
| 128 (Mainstream standard) | 3072-bit | 256-bit |
| 256 (Military grade) | 15360-bit | 512-bit |

Now the secp256K1 curve used by BTC and ETH is a 256-bit ECC private key. If you want to use RSA to achieve the same level, then you need a **3072-bit key**!

---

## Part 5: Why Does Blockchain Use ECC?

### 1. Storage Space

- **RSA**: 3072 bits, about 384 bytes
- **ECC**: 256 bits, about 32 bytes

**A ten-fold difference**! In a system like blockchain where every byte stored permanently requires a fee, using RSA could increase transaction fees by more than 10 times, which is a devastating blow.

### 2. Speed

The smaller the data, the faster the network transmission (block synchronization), making it less prone to congestion. In terms of computing speed, ECC's signing and key generation speeds are also far faster than RSA, which also greatly improves user experience.

---

## Learning Summary

Today we learned three modules:

✅ **Physical Intuition**: Mixing Paint vs. Crazy Billiards

✅ **Mathematical Principles**: Large Number Factorization vs. Discrete Logarithm

✅ **Engineering Efficiency**: Why Blockchain Must Use ECC

---

See you tomorrow.
