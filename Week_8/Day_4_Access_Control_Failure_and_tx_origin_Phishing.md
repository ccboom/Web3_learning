# Classic Vulnerability: Access Control Failure and tx.origin Phishing

The Parity Wallet hack is a textbook case in the field of smart contract security, perfectly demonstrating the catastrophic consequences of access control failures.
Today, we will talk about vulnerabilities related to access control and contract design architecture.

## 1. tx.origin Phishing

Let's start with the first step: `tx.origin` phishing.

To understand phishing attacks, one must first distinguish between two easily confused identities in Ethereum:

1.  **`msg.sender`**: The address that directly calls the contract.
2.  **`tx.origin`**: The original initiator of the transaction.

Let's look at a scenario of chained calls:

Suppose you (User A) buy Token C through a Router Contract B of a decentralized exchange.
The call chain is like this: `User A -> Router Contract B -> Token Contract C`

Now, try to analyze, from the perspective of **Token Contract C** at the very end, who does it see as `msg.sender` and `tx.origin` respectively?

For Contract C:
*   **`msg.sender`**: Is the one directly knocking on the door, which is **Router Contract B**.
*   **`tx.origin`**: Is the original initiator of the transaction, which is **User A**.

### Phishing Scenario Reproduction

Suppose you have a smart contract wallet for saving money. For fund security, the developer wrote a line of code like this in the transfer function:

```solidity
// Only the transaction initiated by the wallet owner can transfer funds
require(tx.origin == owner);
```

A hacker creates a **malicious contract** disguised as an airdrop claim. You (UserA = owner) accidentally call this malicious contract. The content of this malicious contract is: once called, it immediately calls your smart contract wallet to transfer money away.

At this time, the call chain becomes: `owner (You) -> Malicious Contract B -> Wallet Contract C`

At this moment, who is `tx.origin`?
That's right, **it's still you**.

The code runs `require(tx.origin == owner);`, checks it, confirms it is indeed you, and lets it pass.
As a result, the hacker transfers the money away.

Although in this case, it is the malicious contract taking the money, the wallet only checked who the source was. It mistakenly thought you were operating it yourself, so it opened the door, and the funds were drained instantly.

### How to Defend

To plug this vulnerability, we need to modify the code to ensure that **only you yourself** can directly operate the money, refusing any contract acting as a middleman. What should you use as a check?

That's right, it is **`msg.sender`**.

We only need to know who is directly knocking on the door. This is why modern contracts always use `msg.sender` for authentication.

---

## 2. Parity Wallet Hack and Access Control

Next, let's look at a real-world case and analyze the code logic of the Parity Wallet.

To save Gas, the Parity Wallet adopted an architecture of **Wallet Contract + Library Contract**.
The Wallet Contract itself does not write complex logic but uses `delegatecall` to execute the code of the Library Contract.

In order to handle various functions, the Parity Wallet wrote a logic: as long as someone calls it, and it does not have the called function itself, it will forward this request to the Library Contract for processing via `delegatecall`.

In that public Library Contract, there is a function used for initialization:

```solidity
// Code snippet in Library Contract
function initWallet(address[] _owners, uint _required, uint _daylimit) {
    // This line sets the list of wallet owners
    // Note: There is no check like require(msg.sender == owner) here!
    initMultiowned(_owners, _required);
    
    // Set daily transfer limit
    initDaylimit(_daylimit);
}
```

The developer originally thought that this function would only be used once just after creation, and no one would use it afterwards.
The hacker didn't think so. He simply sent a request to the victim's wallet, saying: "Initialize it and set the owner to me." The wallet did exactly that.
Then the hacker easily transferred 30 million USD worth of ETH away.

## 3. Modern Defense Mechanism: Initializer Pattern

Since we have seen the vulnerability, we should learn how to defend against it.

In ordinary Solidity contracts, we usually use `constructor` for initialization. It has perfect characteristics: it runs only once at the moment of contract deployment and then disappears forever.

But in the combination of **Wallet + Library** like Parity, we cannot use `constructor` directly to initialize wallet data.
Because if we write a `constructor` in the Library Contract to set the owner, it only applies to the Library Contract and is of no help to our Wallet Contract.

This poses a huge challenge: the constructor of the Library Contract has no effect on the Wallet Contract. Therefore, developers were forced to give up using the safe `constructor` and switched to a normal function for the wallet to call, in order to initialize the wallet data via `delegatecall`.

This is precisely the root cause of the Parity vulnerability: **Using a normal function to replace the constructor, but forgetting to put "body armor" on it.**

### Initializer Implementation

To solve this problem, modern smart contract development has introduced a standardized defense mechanism, which can run once like a constructor and is compatible with the proxy pattern.

We usually define a boolean variable or a version number.

Think about it, to prevent hackers from calling `initWallet` again like attacking Parity, what should we check first in the `initWallet` function? What operation should be performed after initialization is completed?

We should first check if it is the first run, and then immediately change it to non-first run after running.
This is the core logic for solving the initialization problem of proxy contracts. We usually call it: **Initializer Pattern**.

This is like giving the house a key that can only be used once. After entering the door, the key breaks itself, and no one can come in anymore.

In code implementation, it is usually called `initializer`. Let's look at the code:

```solidity
// Define a switch, default is false (0)
bool private _initialized;

// Define a modifier, equivalent to a "security gate" added to the function
modifier initializer() {
    // 1. Check: Is it the first run? (Can enter if false)
    require(!_initialized, "Contract is already initialized");

    // 2. Lock: Set to true (1), cannot run anymore
    _initialized = true;

    // 3. Execute: Continue running the code in the original function
    _; 
}

// The initWallet here adds the "security gate" above
function initWallet(address[] _owners) public initializer {
    // Real initialization logic...
    initMultiowned(_owners, ...);
}
```

### Defense Effect

With this layer of protection, let's go back to the scene where it was just hacked.

If Parity had added a protection lock at that time, what would happen to the contract when the hacker tried to call `initWallet`?
The transaction would **Revert**!

Because in the normal wallet creation process, `initWallet` has already been called once, and the `_initialized` variable has been set to `true`. When the hacker tries to call it again:

1.  The `initializer` modifier runs.
2.  Checks `require(!_initialized)`.
3.  Finds that `!true` is `false`.
4.  **Boom!** Transaction fails, triggers Revert.

All state changes (including the hacker's attempt to change the owner to himself) will be undone, just as if nothing happened. This is the core defense mechanism of modern proxy contracts.

## Summary

In this chapter, we delved into two classic access control-related vulnerabilities:

1.  **tx.origin Phishing Attack**:
    *   **Principle**: Exploiting the contract's blind trust in the transaction initiator (`tx.origin`). Hackers can induce users to initiate transactions through an intermediate malicious contract. Although `msg.sender` is the malicious contract, `tx.origin` is still the user.
    *   **Defense**: Always use `msg.sender` instead of `tx.origin` during authorization. `tx.origin` is only used in very few specific scenarios (such as restricting non-contract calls `require(tx.origin == msg.sender)`).

2.  **Parity Wallet Multi-sig Vulnerability (Access Control + Delegatecall)**:
    *   **Principle**: Using a proxy pattern to save Gas, but the initialization function (`initWallet`) was exposed as a normal public function and lacked the protection of "can only be called once". Hackers directly called this function to take over the ownership of the wallet.
    *   **Defense**: Use the **Initializer** pattern. Lock the initialization function through state variables (such as `_initialized`) to ensure that it can only be executed once during the contract life cycle. This is the core logic of the OpenZeppelin `Initializable` contract and a standard practice for writing Upgradeable Contracts.

The core of security lies in: Never trust external input, never assume the calling scenario of a function, and clearly define the permission boundaries of every critical function.
