# W7D2: Memory Operations in Yul

## EVM Memory Overview

We've already covered EVM memory before.

Imagine it as an infinitely extending grid-like tape, where each cell has a size of **1 byte** (i.e., 8 bits in binary, such as `0xFF`).

Each cell is numbered, starting from 0, followed by 1, 2, 3...

Moreover, this tape is **disposable** - once used, it's discarded, and a new one is taken the next time you need it.

## Memory Operation Tools

Now we have three tools to operate on this tape.

### Write Tools

First, let's talk about the write tools that can write data into the cells.

#### 1. `mstore(p, v)`

- `p` refers to **position**, indicating where to start placing data
- `v` refers to **value**, the data you want to store

The EVM is a 256-bit machine, so its favorite operating unit is **32 bytes**.

When you use `mstore`, it's like holding a large stamp and pressing it down. Since this stamp covers 32 cells, every time it stamps from position `p` to `p+31`, all data in those cells will be overwritten, regardless of whether there was data before.

#### 2. `mstore8(p, v)`

This is like using a personal seal.

It only stamps on the single cell at position `p`, writing the last byte of value `v`, without affecting neighboring cells.

### Memory Overwrite Example

Now, if we take a brand new tape:

1. First, I use `mstore(0, AAAAA..AA)` to stamp
2. Then I use `mstore(1, BBBB..BB)` to stamp again. What happens?

**Result**: Only cell 0 contains A, and cells 1 through 32 are all overwritten with B!

This isn't right! I don't know where the data is. I can't rely on memory alone - there will be times when I forget.

## Free Memory Pointer

This brings us to something new: **Free Memory Pointer**

We can't search for empty positions every time - that's too troublesome. Instead, we designate one cell on the tape to record which cell onwards is empty and available for writing.

- We set its position at **`0x40`** (which is 64 in decimal)
- The initial value is usually **`0x80`** (which is 128 in decimal), indicating that the first 128 cells are occupied

### Memory Layout

What are the first 64 bytes for? Why not record at position 0?

#### `0x00-0x3F`: Hash Scratch Space

This area is the scratch space for hash operations.

Every time we use `keccak256`, we need a place to compute. We can't allocate new memory each time - that would be very expensive.

So from the beginning, we designate the first 64 cells as scratch space, saving the cost of allocating memory each time.

#### `0x40-0x5F`: Free Memory Pointer Area

- `0x40`: Stores the free memory pointer
- `0x41-0x5F`: Empty, usually unused

#### `0x60-0x7F`: Zero Slot

Always remains 0, serving as an initial value.

## Standard Memory Operation Workflow

So the standard memory operation workflow is:

### 1. Find Position
First check `0x40` to see where the free space starts

### 2. Store Data
Use `mstore` or `mstore8` to store data

### 3. Update Position
Update `0x40` to the new position after writing data, making it convenient for the next operation

**Example**: Suppose we just occupied 32 bytes (which is `0x20` in hexadecimal), and the original position recorded in the cell was `0x80`. What should we change the position to now?

**Answer**: It should be `0x20 + 0x80 = 0xA0`

Now the empty cells start from `0xA0` onwards.

## Read Tool

### `mload(p)`

`mstore` is stamping, while `mload` is retrieving data from position `p` to examine it.

However, when using `mload`, it will retrieve all **32 cells** of data starting from position `p` at once.

It copies this data to your stack for use, **without changing** any data in the cells on the tape itself.

The operation to read `0x40` is: `mload(0x40)`

## Updating the Free Pointer

Alright, we've calculated that the new position is `0xA0`, so we use `mstore` to store `0xA0` at position `0x40`:

```solidity
mstore(0x40, 0xA0)
```

## Complete Code Example

Let's use code to explain the memory operation workflow:

```solidity
assembly {
  // Step 1: Find position - read the free position from 0x40
  let freeptr := mload(0x40)
  
  // Step 2: Write data - write the data starting from the free position
  mstore(freeptr, 0xaaaa...)
  
  // Step 3: Update position - update 0x40 to the latest free position
  // Remember to use the add() function because Yul doesn't have the + operator
  mstore(0x40, add(freeptr, 0x20))
}
```

---

**Summary**:
- EVM memory is a temporary byte array
- `mstore(p, v)` writes 32 bytes, `mstore8(p, v)` writes 1 byte
- `mload(p)` reads 32 bytes
- Use the Free Memory Pointer (`0x40`) to track free memory positions
- Standard workflow: Read pointer → Write data → Update pointer
