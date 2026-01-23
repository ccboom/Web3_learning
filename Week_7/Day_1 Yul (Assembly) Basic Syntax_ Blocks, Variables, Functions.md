# Deep Dive into Yul (Assembly) - Under the Hood

Today we are going to reach under the hood and touch something truly deep —— **Yul (Assembly)**.

This is like switching from driving an automatic (Solidity) to a manual transmission (Assembly). There is a big difference, but manual control allows for finer manipulation and more precise control over Gas.

Today we will start with simple syntax: **Blocks**, **Variables**, and **Functions**.

## Entering Yul Mode

In Solidity contracts, we enter Yul mode via the `assembly {}` block. Inside these curly braces, all Solidity rules are invalid.

In Yul, there are no types like `bool` or `string`. Everything is a **32-byte (256-bit) number**.

- `true` is `1`
- `address` is just a numerical value
- `string` is just a memory pointer; it acts like a signpost leading to the room where the data is stored.

## Variable Declaration and Assignment

In Yul, we cannot write `uint x = 1` like in Solidity.

- We must use the keyword `let` for initialization.
- We must declare an initial value during initialization.
- The assignment symbol has changed from `=` to `:=`.
- When re-assigning, we don't need `let`; just use `:=`.

### Syntax Comparison Table

| Solidity | Yul (Assembly) | Description |
| :--- | :--- | :--- |
| `uint256 x = 123;` | `let x := 123` | Variable Declaration |
| `x = 456;` | `x := 456` | Re-assignment |
| `uint256 y;` (default 0) | `let y := 0` | Must explicitly initialize |
| `uint256 z = x + y;` | `let z := add(x, y)` | Use Opcode functions |

**Note**: Yul does not support operators like `+`, `-`, `*`. You must use EVM Opcodes, such as `add()`, `sub()`, `mul()`, etc.

## Blocks and Scope

The curly braces `{}` in Yul are not just for grouping code; they are more like a **pen for variables**.

A **Block** is a section of code wrapped in curly braces `{}` and is extremely important in Yul.

- Variables defined **inside a block** are **immediately destroyed** (popped from the stack) when the block ends, making them inaccessible from the outside.
- However, code **inside a block** can access variables from the **outside**.

This is crucial for Gas optimization because destroying unneeded variables as early as possible relieves pressure on the stack.

### Example

```solidity
assembly {
    // Assign a to 10
    let a := 10
    
    {
        // Open a block, define a temp variable b, assign to 20
        let b := 20
        // Let c equal a + b (can access external variable a)
        let c := add(a, b)
    }
    
    // After the block closes, both c and b are popped from the stack, so they are gone.
    
    // The following line will error because b is no longer in context
    // let d := b 
}
```

## Functions

Functions in Yul look very similar to Solidity functions, but with some key differences.

Syntax Structure:
```yul
function funcName(var1, var2, ...) -> return1, return2, ... {
    // Function body
}
```

- We **do not need to return** any values explicitly. Just **assign the result to the return variable**. When the function ends, that variable's value is the return value.
- Yul natively supports multiple return values. It effectively leaves data on the top of the stack, which is highly efficient.
- There are no modifiers like `public`. Yul functions are visible within the current `assembly` block.

### Example

```yul
assembly {
    // Define a square function
    function square(input) -> result {
        result := mul(input, input)
    }

    // Call it
    let val := 10
    let res := square(val) // res becomes 100
}
```
See? We just assigned the result to the `result` variable, and it automatically returns that value at the end. No `return` needed.

## Control Flow: If Statements

The key difference between `if` in Yul and Solidity is: **There is no `else`, and no `else if`**.

If you want to implement:
```python
if A:
    X = C
else:
    X = B
```

In Yul, you must write it like this (Default Value Pattern):
```yul
X := B
if A {
    X := C
}
```
Since there is no `else`, we write the `else` condition as the default initial value. When the `if` condition is met, it overwrites the default value. Alternatively, you can use two independent `if` statements.

To write `if`, we need the following comparison operators:

- `lt(a, b)`: **less than**
- `gt(a, b)`: **greater than**
- `eq(a, b)`: **equal**
- `iszero(a)`: Checks if zero (used for logical NOT, or checking if boolean is false)

### Example: Max Function

Since there is no `else`, we can write a `max` function like this:

```yul
function max(a, b) -> result {
    result := b
    if gt(a, b) {
        result := a
    }
}
```
A simple `max` function is done. Very simple, right?

## Loops

Yul loops are also different from Solidity. The structure is:

```yul
for { init } condition { post_iteration } {
    // Loop body
}
```

1.  **Init**: Executed once, at the start. E.g., `let a := 0`.
2.  **Condition**: Checked before every iteration. If true (non-zero), continue. E.g., `lt(a, 5)`.
3.  **Post-iteration**: Executed after every loop body. E.g., `a := add(a, 1)`.

### Example

```yul
// This is a loop that runs 5 times
for { let i := 0 } lt(i, 5) { i := add(i, 1) } {
    // Loop code
}
```

## Switch Statements

Since `if` has no `else`, what if we need multi-condition logic? **Switch** comes to the rescue.

In fact, `switch` is often more useful than `if` in Yul. Structure:

```yul
switch variable
case valueA {
    // Execute code
}
case valueB {
    // Execute code
}
default {
    // Executed when nothing matches, similar to else
}
```

**Note**: `case` must be followed by a **constant**. It cannot be a variable or expression.

### Example

```yul
function getStatus(code) -> status {
    switch code
    case 1 {
        status := 100 // Active
    }
    case 2 {
        status := 200 // Paused
    }
    default {
        status := 0   // Unknown
    }
}
```

## Practice Exercise: Permission Checker

Finally, let's do an exercise ourselves. We want to write an extremely simple "Permission Checker" function `checkAccess`.

**Requirements**:
1. Function receives one argument `userType`.
2. Returns one argument `accessLevel`.

**Logic**:
- If `userType` is 1 (Admin), set `accessLevel` to 99.
- If `userType` is 2 (User), set `accessLevel` to 10.
- **Special Case**: For any other user, we need to do some "extra checks" (simulated) inside a block `{}`:
    - First default `accessLevel` to 0.
    - If `userType` is greater than 5 (`gt(userType, 5)`), set `accessLevel` to 1 (Guest).
    - *(Hint: Use the `default` branch, and put an `if` inside it)*

### Answer

```yul
function checkAccess(userType) -> accessLevel {
    switch userType
    case 1 {
        accessLevel := 99
    }
    case 2 {
        accessLevel := 10
    }
    default {
        // Default level 0
        accessLevel := 0
        
        // Extra check block
        {
            if gt(userType, 5) {
                accessLevel := 1
            }
        }
    }
}
```
