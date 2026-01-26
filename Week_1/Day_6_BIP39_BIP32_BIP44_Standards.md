# BIP Standards Explained: Mnemonic, HD Wallet and Path Specification

This week's learning is coming to an end. Today we start learning BIP-39 (mnemonic), BIP-32 (HD wallet), and BIP-44 (path) standards, aiming to understand hierarchical derivation.

## Origin of BIP

**BIP** is actually the abbreviation for **Bitcoin Improvement Proposal**.

Bitcoin is open source, and anyone can make suggestions. When you write a technical standard and it is adopted by the community, the administrator will assign it a number. The numbers have no strict order, just an ID code.

Let's first view these three standards as a complete key management pipeline:

1. **BIP-39 (Raw Material)**: Converts hard-to-remember random numbers into human-readable mnemonics and generates the original seed.
2. **BIP-32 (Core)**: Defines how to use this seed, combined with HMAC-SHA512 and ECC, to propagate child keys like a tree.
3. **BIP-44 (Specification)**: Since infinite keys can be generated, BIP-44 defines the specific structure of this tree, which branch represents ETH, which represents BTC.

Let's learn in order, starting from BIP-39.

---

## Part 1: BIP-39

The goal of BIP-39 is very clear. It converts random numbers (entropy) that computers like into words (mnemonics) that humans like, and finally back into the seed that computers need.

Let's treat it as a pipeline:

**Entropy → Mnemonic → Seed**

### Entropy → Mnemonic

BIP-39 usually uses a fixed wordlist of 2048 words. If we have 2048 words, how many bits of information can each word represent?

Let's reason it out:

- If there are only two words: e.g., so, last, we only need 1 bit to distinguish: so = 0, last = 1
- But if we have four words, we need 2 bits: so = 00, last = 01, go = 10, now = 11
- Assuming 8 words, we need 3 bits

The rule is $2^n = N$, where n is the number of bits and N is the number of words.

So 2048 words can be distinguished by **11 bits**, because $2^{11} = 2048$.

This is like slicing a long string of 0s and 1s into groups of 11, and then each group can deduce a word.

### Composition of 12 Words

Now, BIP-39 most commonly uses a 12-word mnemonic, so 12 words use a length of 132 bits.

Recall that the computer generates a 128-length random number (entropy), but we need 132 length here. What to do? There are 4 bits extra.

Actually, these extra 4 bits are crucial. Their function is to prevent mistakes during copying and inputting. In cryptography, we call it **Checksum**.

We use the learned SHA-256 to guarantee:

1. **Calculate**: Calculate the SHA-256 hash value of the 128-bit random number.
2. **Truncate**: Take the first 4 bits as the checksum.
3. **Concatenate**: Append these 4 bits to the end of 128.

Now the data becomes: 128-bit Entropy + 4-bit Checksum = 132 bits.

So based on the above deduction, the 12th word is a mixture of the random number and the checksum. The structure is as follows:

```
Total Length: 132 bits
(Contains 128-bit Entropy + 4-bit Checksum)

<------------------------ 128 bits Initial Random Number ------------------------> <--- 4b --->
+-----------+-----------+-----+-----------+------------------------------+------------+
|  11 bits  |  11 bits  | ... |  11 bits  |            7 bits            |   4 bits   |
+-----------+-----------+-----+-----------+------------------------------+------------+
|   Word 1  |   Word 2  | ... |  Word 11  |                  Word 12                  |
+-----------+-----------+-----+-----------+-------------------------------------------+
      ^           ^                 ^                    ^                      ^
    Pure        Pure              Pure           End of Random Number       Checksum(CS)
   Random      Random            Random
```

This is why if you try to replace the last word with another word randomly, it will prompt an invalid mnemonic.

### Mnemonic → Seed

OK, we understand how to turn it into 12 words. Now we need to turn these 12 words into a 512-bit binary seed. This is the core of BIP-32 later, which is also used to sprout from this 512-bit seed.

This step uses a function called **PBKDF2**, which is based on HMAC-SHA512.

#### What is HMAC-SHA512?

1. **SHA-512**: Same as SHA-256, but the output becomes 512 bits.
2. **HMAC**: Full name is Hash-based Message Authentication Code. Ordinary hash is hash(content), HMAC is hash(key + content). Ordinary hash is the same for anyone who calculates it, but HMAC requires a key. If you don't know the key, you can't calculate the correct hash value.

In the process of BIP-39 seed generation, the PBKDF2 function we use utilizes HMAC-SHA512 to repeatedly stir the mnemonic 2048 times, making it difficult to crack.

### Role of Passphrase

In the process of generating the seed, PBKDF2 needs a **salt** to increase security.

BIP-39 stipulates that this salt consists of two parts:

1. Fixed string "mnemonic"
2. User-optional passphrase (password)

How does this passphrase work?

其实 regardless of whether the user sets a passphrase or not, the algorithm will always concatenate a salt:

$$\text{Salt} = \text{"mnemonic"} + \text{User Set Passphrase}$$

Here are 2 cases:

1. **Leave blank**: $\text{Salt} = \text{"mnemonic"}$, calculating directly will get seed A. Most default wallets are like this.
2. **If passphrase is set**, suppose it is abc, $\text{Salt} = \text{"mnemonicabc"}$. Because of the avalanche effect, a slight change in input leads to a vastly different output result, so another seed B will be obtained.

This has two benefits:

1. **Security**: If the mnemonic is lost, but your passphrase is unknown, then what is recovered is the blank seed A, while the real seed B holds the assets.
2. **Hidden Wallet**: You can generate countless mutually non-interfering wallets under one mnemonic by setting different passphrases.

OK, now you basically understand BIP-39.

---

## Part 2: BIP-32

We now have a 512-bit seed. We need to use BIP-32 to generate the root node of this seed.

### Seed Split

There is a mathematical allocation problem here: When we learned ECC, we said the standard private key is usually a 256-bit integer, but what we have is 512 bits.

BIP-32 stipulates that the root node needs not only a private key but also something called **Chain Code**, which is also 256 bits, used to assist in generating child private keys.

So we cut the 512-bit seed in the middle:

- **Left 256 bits** → Master Private Key, controls asset core
- **Right 256 bits** → Master Chain Code, used to generate descendant genes

Adding these two together produces the so-called **Extended Private Key (xprv)**.

### Child Key Derivation

Now we are at the root, with parent chain code and parent private key. We want to generate the first child account, 0.

This is the most powerful part of BIP-32. It uses the **HMAC-SHA512** function again:

- **Input**: 1. Parent Chain Code 2. Parent Public Key 3. Child Index (e.g., 0)
- **Output**: Another 512-bit hash value, which is also split into 2
  - **Right 256** → Directly becomes Child Chain Code
  - **Left 256** → This is an intermediate value, which must be combined with the parent private key to calculate the child private key

The private key is essentially a huge integer. Based on the knowledge we learned, to combine the intermediate value and the parent private key to generate a new integer, we should use addition, which is simple:

$$\text{Child Private Key} = (\text{Parent Private Key} + \text{Left 256-bit Intermediate Value}) \bmod n$$

Note: Must take modulo n. If it exceeds the maximum range of the curve, ensure it is within the legal range.

### Magical Property of Public Key Derivation

Here we have to mention another thing, public key derivation, which is also a magical property of BIP-32.

There is a mathematical law in ECC: **Homomorphism**

- If $Private Key C = Private Key A + Private Key B$
- Then $Public Key Point C = Public Key Point A + Public Key Point B$

This means that even if we only have the parent public key and chain code, and don't know the parent private key, we can rely on the corresponding offset point and directly do addition:

$$\text{Child Public Key Point} = \text{Parent Public Key Point} + (\text{Left 256 bits} \times G)$$

This is also one of the selling points of HD wallets!

### Practical Application Scenarios

So think about it now, in what scenarios can this be used?

For example: We have now opened an exponentially popular e-commerce website. Thousands of users place orders every day. To know who placed which order, we generate a unique payment address for each order.

At this time, there are two choices:

1. **Plan A**: We put the extended private key on the server and let the server generate for users.
2. **Plan B**: We put the extended private key in a safe and put the Extended Public Key (xpub) on the server.

Anyone with eyes would choose the latter. As long as we have the parent public key, we can use the characteristics mentioned above to calculate the addresses of all child private keys. This way, we can use xpub to continuously generate new payment addresses for customers.

If one day, a hacker attacks the server and takes away the xpub, can the hacker transfer the money?

The answer is definitely no. He only has the public key but not the private key, so he can only look but not touch.

This is the so-called **Watch-only Wallet**.

### Necessity of Hardened Derivation

Although xpub is very cool to use, there is a fatal weakness in mathematics.

Didn't we use Child Private Key = Parent Private Key + Intermediate Value (calculated from Parent Public Key and Chain Code) to calculate before?

What if this happens: A hacker stole the xpub and also obtained a child private key through some means.

Now the hacker has the xpub and can calculate the intermediate value. Then he has the child private key and the intermediate value. According to the formula above, wouldn't he be able to calculate the parent private key?

Once the parent private key is calculated, all branches are doomed.

To solve this problem, BIP-32 invented **Hardened Derivation**, often represented by `'` or `h` in the path. We will mention it when we talk about paths later.

Hardened derivation can prevent hackers because it completely changes the material for generating child keys:

- **Normal Derivation**: Raw Material = Parent Public Key + Index
- **Hardened Derivation**: Raw Material = Parent Private Key + Index

So now let's think: The hacker has xpub (which only contains parent public key, no parent private key) and also stole a child private key.

Now the hacker wants to reverse deduce the parent private key: Parent Private Key = Child Private Key - Intermediate Value

- If it is normal derivation, the raw material is the parent public key, which the hacker happens to have!
- If it is hardened derivation, the raw material is the parent private key. He simply cannot calculate it because he only has the parent public key. It becomes very secure.

This is why you often see paths with `'` such as `44'`. This represents a firewall.

---

## Part 3: BIP-44

We now have the seed (BIP-39) and the scheme for generating the tree (BIP-32), but if the branches are not managed, they will grow chaotically.

**BIP-44** sets a standard for this tree, specifying what each value does. This is the path we mentioned above.

### Standard Path Structure

**Standard Path Example**: `m / purpose' / coin_type' / account' / change / address_index`

**Example, ETH Path**: `m / 44' / 60' / 0' / 0 / 0`

If you want to manage ETH and BTC in the same wallet at the same time, where do you think the distinction should be made?

The answer is **Layer 2**.

### Detailed Explanation of Each Level

1. **Layer 1 Purpose**: We write `44` to tell it that we are doing this according to BIP-44, don't mess with other things. If we use another standard, it will become something else, such as `49'`.

2. **Layer 2 Coin Type**:
   - Bitcoin (BTC) code is `0'`
   - Ethereum (ETH) code is `60'`
   - Litecoin (LTC) code is `2'`

   This is like a branch, each branch represents a currency.

3. **Layer 3 Account**: Hardened derivation, with `'`, just like opening several sub-accounts in a bank. `0'` is the savings account, `1'` is the pocket money account. Even if you give the xpub of the pocket money account to others, others cannot know the situation of your savings account. This is the beauty of layering.

4. **Layer 4 Change**: Usually only two:
   - `0` represents receiving address, used to collect money
   - `1` represents change address. This is a unique mechanism of BTC UTXO. When your transfer generates change, the wallet will automatically transfer the money here.

5. **Layer 5 Address Index**: 0, 1, 2, 3... to infinity.

### Practical Exercise

Check if you have learned it. We are now using an HD wallet. Use the second Bitcoin account to generate the first address for receiving payment. Please deduce the complete path.

**The answer is**: `m / 44' / 0' / 1' / 0 / 0`

---

## Summary

### 1. BIP-39: Raw Material Processing (Mnemonic & Seed)

**Task**: Turn random numbers understood by computers into words understood by humans, and finally into seeds understood by algorithms.

#### Entropy to Mnemonic

- **Core Logic**: 12 words = 132 bits of information.
- **Source**: 128 bits pure random number + 4 bits checksum (first 4 bits of SHA-256).
- **Principle**: 132 bits sliced into 12 parts, each part 11 bits. $2^{11} = 2048$, exactly corresponding to the 2048 wordlist.
- **Function**: Checksum prevents transcription errors (entering one wrong word will report an error).

#### Mnemonic to Seed

- **Function**: PBKDF2 (based on HMAC-SHA512).
- **Salt**: "mnemonic" + Passphrase.
- **Passphrase Function**:
  - **Security**: Prevent physical theft (having words without code cannot steal funds).
  - **Concealment**: The same set of mnemonics with different passwords generates completely different wallets (Seed B), interfering with each other.

### 2. BIP-32: Breeding Machine (Hierarchical Deterministic Derivation)

**Task**: Use one seed to spawn an infinite private key tree.

#### Birth of Root

- 512-bit seed is cut in half: Left 256 bits is Master Private Key, Right 256 bits is Master Chain Code. Together called xprv (Extended Private Key).

#### Child Key Derivation

- **Formula**: Child Private Key = (Parent Private Key + Intermediate Value) mod n.
- **Public Key Characteristic**: Using ECC homomorphism, child public keys can be derived directly using xpub (Extended Public Key) without private key.
- **Application Scenarios**: E-commerce collection, Watch-only Wallet (receive only, no pay).

#### Fatal Weakness and Fix (Hardened Derivation)

- **Weakness**: If a hacker has xpub + a child private key, they can reverse deduce the parent private key, leading to a total collapse.
- **Fix**: Hardened Derivation (with `'` symbol).
- **Principle**: Cut off the connection of "Parent Public Key" during derivation, forcing the use of "Parent Private Key" in calculation. Hackers don't have parent private key, cutting off the path of reverse deduction.

### 3. BIP-44: Address House Number (Path Specification)

**Task**: Set rules for this infinitely growing tree to prevent chaotic growth.

#### Standard Path

`m / purpose' / coin_type' / account' / change / address_index`

#### Layer Meaning

- **Purpose (44')**: I follow BIP-44 standard.
- **Coin (0' or 60')**: 0' is BTC, 60' is ETH. This is where currencies are distinguished.
- **Account (0', 1'...)**: Different sub-accounts in the bank (savings, pocket money), used Hardened Derivation, isolated from each other.
- **Change (0 or 1)**: 0 is external collection, 1 is internal change (UTXO feature).
- **Index (0, 1, 2...)**: The specific Nth receiving address.
