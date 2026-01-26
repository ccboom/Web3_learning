# In-depth Study of Cryptography Basics: Hash Functions

**Keywords:** SHA-256 Principles, Collision Resistance, Avalanche Effect

---

## 1. SHA-256

SHA-256 is the core algorithm of Bitcoin mining.

Simply put, it serves as both a **shredder** and a **fingerprint extractor**. Regardless of what your input is—whether it's 10,000 TB of data or just 1 bit—it will output a fixed-length value, and this process is irreversible.

For example, whether the input is the letter `s` or the entire movie *Inception*, the output will definitely be a 256-bit Hash value.
What we usually see is the hexadecimal Hash value. Since 1 hexadecimal character requires 4 bits to represent, 256 bits of binary translates to 64 digits of hexadecimal. This is consistent with the length we usually see.

### Core Mechanism: How does it work?

How to turn arbitrary data into a fixed length?
First, the input data is not processed all at once ("you can't eat a fat man in one bite"). The data is sliced into **512-bit** blocks, which are then read one by one to finally calculate the result we need.

#### Why is the result 256 bits, but the block size is 512 bits?

There are two main reasons for this:

1. **Historical and Architectural Reasons:**
    When SHA-256 was designed in 2001, mainstream computers were 32-bit. 512 bits can be exactly divided by 32:
    $$512 / 32 = 16$$
    This means that a block can be exactly sliced into 16 Words, which makes processing very "comfortable" for the machine, requiring no complex splitting or piecing together.

2. **Balance between Efficiency and Security (Blender Theory):**
    We can compare the processing to **mixing in a blender**.
    * **Internal State (Container):** 256 bits.
    * **Input Data (Ingredients):** 512 bits.

    This is a dynamic processing process. If the input data is too small (adding ingredients with a spoon), the operation efficiency is extremely low; if the input data is too large (pouring water with a 10M diameter pipe), it is difficult to digest thoroughly and preserve details (prevent collisions).
    **512 bits is the balance point found by engineers**: it ensures sufficient speed while guaranteeing that the current structure can make the data sufficiently random and secure.

### Padding

Real-world data length is irregular; for example, the word `web` has only 3 characters (24 bits). How to turn it into a 512-bit block?
Directly padding with `0`s won't work. For example, `101` padded becomes `101000...`, and `1010` padded becomes `101000...` as well. This would cause different inputs to produce the same Hash, which is absolutely not allowed.

**Solution:**

1. **Padding Bit:** First, append a `1` to the end of the data as a data end marker.
2. **Padding Zeros:** Then append `0`s until a specific length is met.
3. **Recording Length:** Record the length of the original data in the last **64 bits**.

> **Why use 64 bits to record length?**
> This is a limitation of SHA-256, meaning it cannot process files larger than $2^{64}$. But for the vast majority of data in the world today, this is more than enough.

### Compression Function and Initial Vector (IV)

We can imagine the compression function of SHA-256 as a **dough**.
Holding a 256-bit "dough", adding 512 bits of "spices" (data blocks), kneading continuously, and finally, it becomes 256 bits again. The size hasn't changed, but the flavor (characteristics) has completely changed—a massive quantity of features has been compressed into a fixed fingerprint.

**Where does the Initial Vector come from?**
It cannot be all set to `0`, otherwise, the calculation result of the first block would be very regular and easy to crack.
We need an Initial Vector (IV) that must satisfy two characteristics:

1. **Very chaotic:** Looks like random numbers.
2. **Absolutely objective:** Cannot be a special value set artificially (to prevent suspicion of having a backdoor).

Therefore, SHA-256 chose **the first 32 bits of the fractional parts of the square roots of the first 8 prime numbers**.
For example, the first prime number is 2:
$$\sqrt{2} = 1.41421356...$$
Converted to hexadecimal, it is `0x6a09e667`. This is the mysterious number we often see in code:

```python
H0 = 0x6a09e667
H1 = 0xbb67ae85
...
```

## 2. Collision Resistance

So-called collision means inputting two completely different things, but the output results are unexpectedly the same.
Mathematically expressed as:

$$Input A \neq Input B$$
$$Hash(A) = Hash(B)$$

This is like in facial recognition, you and I are identified as the same person. This is extremely dangerous!

In cryptography, we need to honestly admit: **Collision absolutely exists**.
This is the **Pigeonhole Principle**:

* **Input:** Infinite (any data, x/n/a book/all music).
* **Output:** Finite (at most $2^{256}$ possibilities).
Since we have to fit infinite things into finite boxes, inevitably two things will squeeze into the same box.

But what we emphasize is **Resistance** (Collision Resistance), not **Freedom** (Collision Free).
Although collisions theoretically exist, you **simply cannot find them within a realistic timeframe**.

This is "computational security". To let you feel how exaggerated this "unable to find" is, here is a classic comparison:

> Even if all the sand on Earth were turned into supercomputers, and they calculated from the moment of the Big Bang until now, the probability of finding a collision pair is only one in several billion.

This is the sense of security SHA-256 gives us: it is imperfect mathematically, but indestructible in the physical world.

## 3. Avalanche Effect

Since we know that the Hash values of two files are almost impossible to be the same, what if we only modify one bit in the file?
For example, changing a binary number from `1` to `0`, the result would be:

1. Almost the same (only changed a little bit, after all, 99.99% of the original file hasn't changed)?
2. **Unrecognizable** (completely unable to see that they were originally very similar files)?

The answer is **2. Unrecognizable**. This is the **Avalanche Effect** in our cryptography.

> A small snowball rolling down a mountain will eventually become a giant snowball capable of destroying a car.

We changed a tiny value, and after continuous "kneading" operations, this tiny change gets continuously amplified.
Ultimately, the new and old Hash values may have **50% of the bits** changed (i.e., 0 becomes 1 or 1 becomes 0), causing the results to look completely different.

This is exactly the security guarantee of SHA-256. If changes followed a pattern, attackers could easily reverse engineer to get what they want. It is precisely because of the avalanche effect that attackers have no place to start.

### About Diffusion Efficiency

Returning to the 512-bit block issue again, if we set the block to 10000, while the internal state is still set to 256, the results would be vastly different.

This is the issue of **Diffusion** efficiency in cryptography:
If 10000 portions of spices are forced into 256 portions of dough, the influence of each spice will be greatly reduced, some essentially having no impact. This reduces system complexity, thereby increasing the success probability of reverse engineering attacks.

# SHA-256 Three-Layer Pyramid Architecture

## Layer 1: The Tools

These are the most fundamental screws; all calculations depend on them.

* **XOR ($\oplus$)**: Cryptography's "Magic Glue".
  * **Characteristics**: 1+1=0 (no carry), reversible.
  * **Function**: Used to mix data, allowing you to knead things in and also (under certain conditions) solve them out.

* **ROTR (Rotate Right)**: Information "Conveyor Belt".
  * **Characteristics**: Bits falling off the right side return to the left side.
  * **Function**: Ensures not a single bit is lost during movement, just changed position (Diffusion).

## Layer 2: The Components

Three core components assembled using the tools above.

* **$\Sigma$ (Sigma Function)**: "The Blender".
  * **Principle**: Through multiple ROTR and XOR operations, spreads the influence of one bit to 3 different positions.
  * <img width="837" height="211" alt="image" src="https://github.com/user-attachments/assets/6f92d4af-0c55-438f-83a2-465df9db1a69" />

* **$Ch$ (Choice Function)**: "Two-way Switch".
  * **Principle**: **$x$** decides whether to pick $y$ or $z$. Introduces non-linearity, making mathematical analysis very difficult.
    <img width="949" height="350" alt="image" src="https://github.com/user-attachments/assets/cfb37e19-ad8f-49d0-b904-b12c064f8355" />

* **$Maj$ (Majority Function)**: "Voting Machine".
  * **Principle**: Minority obeys majority. Ensures the output reflects the "mainstream" characteristics of the current state.
    <img width="1039" height="429" alt="image" src="https://github.com/user-attachments/assets/ed17bca4-95d4-489e-af11-9f48f3e31d01" />

## Layer 3: The Assembly Line

This is that complex 64-round loop chart.
<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/7ccb9e61-0531-4b46-a885-b53f2268f614" />

* **8 Workstations (A-H)**: We have 8 variables serving as temporary storage.
* **Flow Rules**:
  * Most workstations ($B, C, D, F, G, H$) simply take the data from the previous baton (Great Shift).
  * Only $A$ and $E$ are "Injection Points". They receive the new data after being crazily mixed by $\Sigma, Ch, Maj$ (as well as the message of this round $W_t$).
* **64 Rounds**:
  * Must turn enough for 64 rounds to ensure that the tiny bit of change injected into $A$ at the very beginning has enough time to flow to $H$, then be extracted and re-injected back into $A$, thereby thoroughly "polluting" the entire hash state (achieving the avalanche effect).

---

> **Summary in one sentence**
>
> SHA-256 thoroughly "crushes" your input data and evenly "smears" it into a 256-bit space through **64 rounds** of loops, using data-preserving operations like **XOR** and **ROTR**, combined with the complex logic of **$Ch$** and **$Maj$**.

---
*Source: NIST SHA-256 Standard*

Attachment: Mind Map
<img width="5381" height="6336" alt="NotebookLM Mind Map (1)" src="https://github.com/user-attachments/assets/30e8caf8-293a-4401-bcb4-2de75b826d61" />
