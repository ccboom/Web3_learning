# secp256k1 Elliptic Curve Explained

Deep dive into the core of Bitcoin and many blockchain systems: the secp256k1 curve.

## Learning Path

1. Curve Equations
2. Understanding the Generator Point G

---

## Part 1: Curve Equations

### Basic Equation

The name secp256k1 sounds very complicated and hard to understand, but the core equation is actually very simple. It is a special case of the Weierstrass equation.

**Weierstrass Equation:**

$$y^2 = x^3 + ax + b$$

**secp256k1 Equation:**

$$y^2 = x^3 + 7$$

That is, setting $a = 0$ and $b = 7$ in the equation.

![Elliptic Curve Schematic](https://github.com/user-attachments/assets/2ac1a54e-5f0c-4cc4-89cf-25e961a48066)

### Concept of Finite Fields

In cryptography, we cannot use continuous real numbers but must operate on a finite field.

**What is a Finite Field?**

- A field refers to a collection of numbers where addition, subtraction, multiplication, and division can be performed freely.
- Real numbers (all decimals, fractions) are infinite fields.
- Finite field means that the numbers in this collection are finite.

We operate on the finite field $(0, 1, 2, 3, ..., P-1)$.

### Curve Changes in Finite Fields

If we restrict this equation to a finite field, that is, modulo a huge prime number $P$ (take the remainder), what happens to the smooth curve geometrically?

**The answer is: It will become a messy cluster of scattered points.**

![Scatter Plot in Finite Field](https://github.com/user-attachments/assets/5f3b2ab3-708b-4e1b-ba48-8aa2544fd11b)

### Why is it not a line anymore?

This goes back to the finite field we just mentioned:

1. **No Middle Ground**: There are decimals between two integers on a normal curve, but there are only integers in the finite field we use now, so the line is broken.
2. **Teleportation**: Whenever the curve runs out of bounds (greater than $P$), it will be grabbed back to the bottom by the modulo operation, causing the position of the points to jump around without any pattern.

### Perfection of Mathematical Structure

Although we can't see it as a curve with our eyes, we still use it for encryption operations, which is adding two points together. Do you think we can still use the original mathematical formulas? For example, calculating the slope:

$$slope = \frac{y_2 - y_1}{x_2 - x_1}$$

**The answer is of course YES.** Although it looks like a mess in the image, on the mathematical level, the algebraic structure is still perfect.

---

## Operations on Elliptic Curves

To understand BTC public key generation ($K_{\text{public}} = k_{\text{private}} \times G$), we need to master two movement methods on the curve: Addition ($P+Q$) and Point Doubling ($P+P$).

### Geometric Addition Rule

On an elliptic curve, addition is not simple coordinate addition: $(x+x, y+y)$, but a geometric game.

![Elliptic Curve Addition Schematic](https://github.com/user-attachments/assets/c3e8f7bc-aa35-493b-993e-2a7f46038623)

**Addition Steps:**

1. **Connect**: To calculate $P+Q$, draw a line passing through the two points.
2. **Intersect**: The line drawn by these two points will intersect at a third point (point $-R$).
3. **Flip**: Reflect this intersection point along the X-axis to get $R$, which is the result.

This is the origin of the formula $P + Q = R$.

### Point Doubling

Next is the most critical step in generating a public key. Imagine a special scenario: If we want to calculate $P+P$, what is the line passing through the two points at this time?

Thinking about the picture above, $Q$ moves closer and closer to $P$ along the curve until it overlaps with point $P$. At this time, the line passing through them becomes: **The Tangent Line**.

This is not just a geometric concept; this is the core accelerator in BTC public key generation. This operation is called **Point Doubling**.

The rule for calculating $P + P$ is to turn the connecting line in ordinary addition into a tangent line. The magical properties of elliptic curves ensure that unless $P$ is some special point, this tangent line will definitely extend and cross the curve again at another place, which we call $-R$. After flipping, it becomes $R$, which is $2P$, as shown below:

![Point Doubling Schematic](https://github.com/user-attachments/assets/973e3d43-72a7-4557-b1a6-b4c7455f245d)

### Why is it special algebraically?

We know the slope formula is $m = \frac{y_2 - y_1}{x_2 - x_1}$.

- **For $P + Q$**: Because they are two points, the coordinates are different, so the denominator is not 0 and can be calculated directly.
- **But for $P+P$**: The two points overlap. If the above formula is used, it becomes $m = \frac{0}{0}$.

What to do? Now we have to use derivatives. Mathematicians calculate the rate of change of the curve at a certain moment through differentiation.

For the secp256k1 curve ($y^2 = x^3 + 7$), the tangent slope formula at point $P(x, y)$ is:

$$m = \frac{3x^2}{2y}$$

**Note:** When calculating in a finite field, dividing by $2y$ becomes multiplying by the "modular inverse" of $2y$.

No need to memorize this formula, just understand that when calculating $P + P$, the computer uses a different algebraic formula than $P + Q$.

### Why do we spend so much effort talking about Point Doubling?

Our private key $k$ is a 256-bit number. If we use the public key $K = k \times G$, $G+G+G+G+...$ $k$ times, even a supercomputer cannot finish the calculation.

But now with Point Doubling ($2P$), we can cheat.

**If $k$ is set to 100 now:**

We can split 100 into $64+32+4$, so $100G = 64G + 32G + 4G$.

- $G + G = \mathbf{2G}$ (Use Point Doubling once)
- $2G + 2G = \mathbf{4G}$ (Use Point Doubling again)
- $4G + 4G = 8G$
- ...
- $32G + 32G = \mathbf{64G}$

We only need 6 Point Doubling operations to get 64G, and then add the 32G and 4G we obtained earlier, which is very simple.

This step allows us to cross a huge space to calculate the public key in extremely short time, while making it impossible to reverse calculate how many steps were jumped. Very nice.

---

## Part 2: Generator Point G

### Definition of Generator Point

In secp256k1, $G$ is actually an agreed-upon, fixed point.

Satoshi Nakamoto did not invent this point but used the point recommended by the SECG standard. Its coordinates $(x, y)$ on the curve are two huge numbers (represented in hexadecimal):

$$G_x = \texttt{79BE667E F9DCBBAC 55A06295 CE870B07 029BFCDB 2DCE28D9 59F2815B 16F81798}$$

$$G_y = \texttt{483ADA77 26A3C465 5DA4FBFC 0E1108A8 FD17B448 A6855419 9C47D08F FB10D4B8}$$

### Public Key Generation

**How is the public key calculated?**

The private key $k$ is actually a randomly selected number, and the public key $K$ is calculated.

**The formula is:**

$$K = k \times G$$

That is: **Public Key = Private Key × Generator Point**

This means we start from point $G$ on the curve and perform $k$ additions:

$$G + G + G + \cdots + G \quad \text{(Total } k \text{ additions)}$$

---

## Why is it secure?

This is the most brilliant part of the entire cryptography.

### Forward Calculation: Very Easy

You have the private key $k$ and want to calculate the public key. Although $k$ is large (256-bit integer), the computer doesn't stupidly add one by one. It uses the tangent trick we just mentioned to accelerate calculation exponentially (like $2G$, $4G$, $8G$...), basically finishing in milliseconds.

### Reverse Calculation: Basically Hopeless

This involves the finite field scatter plot we mentioned before. If I only give you the starting point $G$ and the ending point $K$ (public key), asking you how many times it was added?

Because points jump around in this finite field, there is no simple algebraic pattern. You cannot use division like $K / G$. The only way is to try one by one:

- Is it 1 time? No.
- Is it 2 times? No.
- ...

### Why is multiplication usable, but division not?

Seems very counterintuitive, right? Because $K$ and $G$ are not numbers at all, they are 2D coordinate points.

Actually, when trying to calculate $k = K \div G$, you are doing this operation:

$$k = \frac{(x_K, y_K)}{(x_G, y_G)}$$

Mathematically, there is no definition for dividing one point by another.

You can add two points, multiply a point by an integer, but you have never learned to divide two points to get an integer. This is like asking "what is banana divided by apple", which is meaningless in mathematics.

### Security Guarantee

Since $k$ is a 256-bit number, the probability of cracking it so far is basically 0. This is the **Elliptic Curve Discrete Logarithm Problem (ECDLP)**, the cornerstone of Bitcoin's unbreakability.

---

# Part 3: $n$ — Order of the Curve

That is $n \times G = O$ (Point at Infinity)

---

## What is $n$ (Order)?

Simply put:

- **G** is the starting point
- **K** is the ending point
- **n** is the **total length of the one-way circular track**

### 🏃 Image Analogy

You can imagine it as an athlete:

> Starting from **O (Point at Infinity)**, taking a fixed distance **G** every step, running continuously.

**Definition of n**: After this athlete runs **n steps**, he finishes exactly one lap and returns to the starting point **O**.

This is the meaning of the formula $n \times G = O$.

---

## Why is $n$ important?

### 1. It is the ceiling of the private key

- $k$ is a random integer, this integer **cannot be infinitely large**
- If $n + 1$ is chosen, it is equivalent to running just one step
- So **BTC private key must be between $1$ and $n-1$**

### 2. What is $n$ for secp256k1?

It is such a huge number:

```
FFFFFFFF FFFFFFFF FFFFFFFF FFFFFFFE BAAEDCE6 AF48A03B BFD25E8C D0364141
```

> Roughly $1.1579 \times 10^{77}$

## Summary

### Core learning today

1. **Curve Equation**: $y^2 = x^3 + 7$ (Becomes scatter points under finite field)

2. **Operation Rules**:
   - Adding different points $P+Q$: Connect → Find Intersection → Flip
   - Adding same points $P+P$: Tangent → Find Intersection → Flip

3. **Generator Point G**: The only starting point for public key generation
