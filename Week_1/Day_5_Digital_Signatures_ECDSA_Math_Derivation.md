# ECDSA Digital Signature Explained

Today we will learn about Digital Signatures: Mathematical Derivation of ECDSA Signature and Verification Process (r, s, v)

**Goal**: Be able to calculate the signature verification process by hand.

---

## Part 1: Signature Generation

### Elliptic Curve Elements

We just learned a few elements on the secp256k1 curve:

1. **G**: The generator point of the elliptic curve
2. **n**: The order of the curve, i.e., $n \times G = O$ (Point at Infinity)
3. **d**: The signer's private key
4. **z**: The hash value of the message to be signed (Message Hash, converted to an integer after truncation)

### Generate Temporary Random Number

The most critical step in ECDSA signature is generating a temporary/secret random number, which we call **k**.

With this random number k and the generator point G learned yesterday, we can calculate a temporary point R on the curve:

$$R = k \times G$$

Now assume the coordinates of R are $(x_R, y_R)$.

### Generate Signature (r, s)

The ECDSA signature result consists of two integers $(r, s)$:

**r**: Directly take the x-coordinate of point R, modulo the order n.

$$r = x_R \bmod n$$

(This step turns a point on the elliptic curve into a pure number)

**s**: This is the core step, which combines the message hash ($z$), private key ($d$), and random number ($k$):

$$s = k^{-1} (z + r \cdot d) \bmod n$$

The signature comprises these two combined: $(r, s)$. Verification only happens when the receiver gets these two numbers.

### Understanding Modular Inverse

First, let's look at $k^{-1}$. Would you directly think that $k^{-1}$ is simply 1 / k?

Then you are wrong. Intuitively it is like that, but in elliptic curve cryptography it's different. We cannot use decimals or fractions; all calculations must be performed within the integer range, and results must be between [0, n-1].

If we simply do division, like 1 / 2 = 0.5, it becomes a decimal, which computers cannot handle in this context.

We use **Modular Inverse** to handle it.

In modular arithmetic, $k^{-1}$ refers to an integer x such that when multiplied by k, the result modulo n equals 1. The formula is as follows:

$$k \cdot x \equiv 1 \pmod n$$

### Example Calculation

Let's take an example to understand: Suppose n = 7, k = 2.

We need to find an integer such that $2 \cdot x \equiv 1 \pmod 7$.

Calculate what it is? Correct, it is **4**.

So in the world of mod 7, multiplying by 4 is equivalent to dividing by 2. This is why we write $k^{-1}$ in the formula, but in actual calculation, we need to calculate this inverse integer first.

### Complete Mini ECDSA Operation

OK, now we have everything, let's put them together to complete a mini version of ECDSA operation.

Using the formula: $s = k^{-1} (z + r \cdot d) \bmod n$

We use some simple data to calculate, but actual blocks definitely won't use this data; it's just to keep the calculation simple:

**Assumptions:**

1. Modulus n = 7
2. Message hash z = 2
3. Private key d = 5
4. Random point R x-coordinate r = 2
5. Random number inverse $k^{-1} = 4$ (This is for k = 2 calculated just now)

Okay, now use the formula to calculate:

s = (4 × (2 + 2 × 5)) % 7 = 6

So in this mini blockchain system, the generated digital signature $(r, s) = (2, 6)$. Isn't it very simple?

---

## Part 2: Verification

Now we are not the signer holding the private key d; we are now an ordinary verifier, such as a miner node on the blockchain.

### Information Owned by the Verifier

What does the verifier have in hand? This is what we must know:

1. Message hash $z = 2$
2. Signature $(r, s) = (2, 6)$
3. Public key Q. Remember how Q comes? From Private Key × G, i.e., $Q = d \times G$. In our example d = 5, so Q = 5G.
4. Curve parameter G and order n = 7

**Verification Goal**: We want to calculate a point using these public numbers to see if it equals that R in the signature, that is, if the x-coordinate equals r.

### Verification Steps

**Step 1**: Calculate the modular inverse of s. We need this to unlock the signature. Let's denote it as w.

$$w = s^{-1} \bmod n$$

Given s = 6, n = 7, then naturally calculated w = 6.

**Step 2**: We need to calculate intermediate parameters, let's call them $u_1$ and $u_2$.

The formulas are as follows:

$$u_1 = z \cdot w \bmod n$$
$$u_2 = r \cdot w \bmod n$$

From the conditions provided earlier, we know $z=2, r=2, w=6, n=7$, so naturally calculate:

$$u_1 = 5$$
$$u_2 = 5$$

**Step 3**: Having all conditions, we now calculate the final point P by combining the curve's base point G and public key Q through these two numbers.

The formula is:

$$P = u_1 \cdot G + u_2 \cdot Q$$

Although in production environments, it involves very complex point addition operations, in our mini system, we can verify using algebraic logic.

Remember public key $Q$? Treat it as $5$ Gs added together because private key d = 5, so Q = 5G.

Using the formula above, we get: P = 5G + 25G = 30G % 7 = 2G

Perfect, calculated point P = 2G.

Remember? Scroll up a bit, we just assumed k = 2, so point R = 2G.

The calculated point P and point R are completely consistent, which means this signature is valid.

### Mathematical Principle Derivation

Are you confused after a bunch of calculations? Why can $u_1$ and $u_2$ restore $k \cdot G$?

Let's summarize the formula: $P = u_1 \cdot G + u_2 \cdot Q \stackrel{?}{=} k \cdot G$, right?

First review what is the relationship between public key $Q$ and private key $d$? Turn P into a form containing only $G$, $u_1$, $u_2$, and $d$ as follows:

$$P = u_1 G + u_2 Q$$
$$P = u_1 G + u_2 (d G)$$
$$P = (u_1 + u_2 d)G$$

Now it's clear.

We just need to prove that $(u_1 + u_2 d)$ equals k.

Known parameter definitions:

$$u_1 = z \cdot w$$
$$u_2 = r \cdot w$$

Then extracting the common factor $w$, the formula becomes:

$$P = \left[ (z + r \cdot d) \cdot w \right] G$$

Finally, finally, what does this middle part $(z + r \cdot d) \cdot w$ actually represent?

Let's look back at the original formula for s:

$$s = k^{-1} (z + r \cdot d) \bmod n$$

In the definition stage, we also said the definition of $w$ is $w = s^{-1}$. Using the algebraic transformation above, we finally obtained:

$$k = s^{-1} (z + r \cdot d) = (z + r \cdot d) w$$

Substitute into the formula just now, it becomes $P = [\mathbf{k}] \cdot G$.

That proves point $P$ is identically equal to the random point $R$ generated by the signer originally ($R = k \cdot G$). Verification successful!

---

## Part 3: The Mysterious Parameter v

Looking back at our title using, did you find it strange? One parameter v is missing?

Let's talk about the mysterious $v$ (sometimes called RecID).

In the reasoning just now, we verified the signature with a known public key Q. But in BTC and ETH, sometimes to save space, we do not transmit the public key Q when sending transactions because it is too long.

So we will use mathematics to reverse recover the public key.

Here is a problem. The elliptic curve is symmetric. For a specific x-coordinate, there will be two points, up and down, on the curve. This is where v comes into play.

It is like a simple signpost (usually 27 or 28), telling the verifier whether point R is the one in the upper half or the lower half.

With V, the verifier can uniquely determine point R, thereby reverse calculating the public key Q, and checking if the Q address has sufficient balance.

### Two Verification Modes

So is it calculating R knowing Q, or calculating Q after calculating R?

#### Mode 1: Standard ECDSA Verification

Traditional cryptography process, used when space saving is not needed.

- **Input**: Message $z$, Signature $(r, s)$, Signer's Public Key $Q$.
- **Process**: Verifier already has $Q$. Calculate a point $P$ using $Q$ and $(r, s)$. Judge: Is x-coordinate of $P$ equal to $r$?
- **Disadvantage**: The person sending the transaction must send the long public key $Q$ (64 bytes or 33 bytes) to the verifier, which takes up more block space and leads to higher fees (Gas).

#### Mode 2: Public Key Recovery Mode

This is an engineering optimization scheme designed to save money.

- **Input**: Message $z$, Signature $(r, s, v)$.
- **Note**: The verifier does not have public key $Q$. The verifier only knows the sender's address (Address, which is a small segment after hashing the public key).
- **Goal**: Verifier asks: "Who signed this message? Is the address corresponding to the calculated public key the one with sufficient balance in my records?"

At this time, the calculation is reversed:

1. **Recover point $R$ using $r$ and $v$**: With $x$ ($r$) and $y$ (determined by $v$), we restore the complete random point $R$.

2. **Reverse Calculate Public Key Q**:

   Review the most basic equation:

   $$s \cdot R = z \cdot G + r \cdot Q$$

   (Note: This is actually $s = k^{-1}(z + rd)$ multiplied by $k$ on both sides and then multiplied by $G$)

   Our goal is to solve for $Q$. That is, to put $Q$ on the left side of the equation and throw everything else to the right.

   Move terms: $r \cdot Q = s \cdot R - z \cdot G$

   Divide both sides by $r$ (that is, multiply by the inverse of $r$, $r^{-1}$):

   $$Q = r^{-1} (s \cdot R - z \cdot G)$$

   Look! In this formula, $R$ is known (calculated in the first step), $s, z, G, r$ are all known. So, we can verify $Q$ directly.

3. **Verify Address**: After calculating the public key Q, the verifier hashes Q, takes the last 20 bytes to get the address, and then checks if this calculated address is the initiator of this transaction?

Now you understand!

---

## Summary

### Core Summary of ECDSA Signature and Verification Process

#### 0. Core Elements Preparation

- **Global Parameters**: $G$ (Generator Point), $n$ (Order of the Curve).
- **Secret Data**: $d$ (Private Key), $k$ (**Random Number**, extremely important, cannot be leaked, cannot be reused).
- **Public Data**: $Q$ (Public Key, $Q=dG$), $z$ (Message Hash).

#### 1. Signature Generation: Manufacture $(r, s)$

**Goal**: Use private key $d$ to "stamp" message $z$, generating a pair of integers $(r, s)$.

1. **Calculate Random Point $R$**:
   Use random number $k$ to calculate point on curve: $R = k \times G$.
2. **Get $r$**:
   Take x-coordinate of point $R$: $r = x_R \bmod n$
3. **Calculate $s$** (Core Formula):
   Combine private key, message, and random number:

   $$s = k^{-1} (z + r \cdot d) \bmod n$$

   *(Note: $k^{-1}$ is modular inverse, not decimal division)*

**Output**: Signature result $(r, s)$.

#### 2. Standard Verification: Verify $(r, s)$

**Scenario**: Verifier knows sender's public key $Q$.
**Goal**: Confirm signature was signed with private key $d$ corresponding to $Q$.

1. **Calculate Inverse $w$**:
   Unlock reciprocal of $s$: $w = s^{-1} \bmod n$
2. **Calculate Intermediate Components $u_1, u_2$**:

   $$u_1 = z \cdot w \bmod n$$
   $$u_2 = r \cdot w \bmod n$$

3. **Reconstruct Point $P$**:

   $$P = u_1 \cdot G + u_2 \cdot Q$$

4. **Final Determination**:
   Check if x-coordinate of point $P$ equals $r$.

   $$x_P \stackrel{?}{=} r$$

**Mathematical Principle**:
Algebraic derivation proves $P = k \cdot G = R$, meaning the verifier restored the random point originally generated by the signer.

#### 3. Advanced Content: Parameter $v$ and Public Key Recovery

**Scenario**: Ethereum/Bitcoin transactions (to save space transmitting long public key $Q$).
**Input**: Signature $(r, s, v)$ + Message $z$.

- **Role of $v$ (RecID)**:
  Since elliptic curves are symmetric, one x-coordinate corresponds to two y-coordinates. $v$ (like 27/28) tells the verifier if point $R$ is the "upper one" or "lower one".
- **Recovery Process**:
  1. Fully restore random point $R$ using $r$ and $v$.
  2. **Reverse Calculate Public Key $Q$**:

     $$Q = r^{-1} (s \cdot R - z \cdot G)$$

  3. **Verify Identity**:
     Calculate $Hash(Q)$ to get address, compare if this address is the initiator of the transaction.

### A Table Comparing Two Modes

| Mode | Standard Verification (Standard) | Public Key Recovery (Recovery) |
| :--- | :--- | :--- |
| **Main Use** | Traditional Secure Communication | Blockchain Transaction (BTC/ETH) |
| **Input Data** | $z, (r, s), \mathbf{Q}$ | $z, (r, s, \mathbf{v})$ |
| **Core Logic** | Use $Q$ to calculate a point, see if equal to $R$ | Use $R$ to calculate a point, see if equal to $Q$ |
| **Advantage** | Intuitive Logic | **Save Space** (No need to store/transmit public key) |

### Memory Mnemonic

- **Signature** is: Random number $k$ hides on both sides, one becomes $r$ (coordinate), one becomes $s$ (formula confusion).
- **Verification** is: Calculate $w$ (inverse of s), piece together $u_1, u_2$, restore point $R$.
- **$v$** is: Guidepost, with $r, s, v$, can calculate who you are without public key.
