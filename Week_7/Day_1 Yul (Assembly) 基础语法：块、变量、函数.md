# 深入 Yul (Assembly) - 引擎盖之下的世界

我们今天要把手伸到引擎盖以下，接触一些真正深层的东西 —— **Yul (Assembly)**。

这就像开车从自动挡的 Solidity 变成手动挡的 Assembly。中间差了很多，但是手动挡可以操控得更细腻，对于 Gas 的把控也更精准。

我们今天先学习简单的语法：**块 (Block)** / **变量 (Variable)** / **函数 (Function)**。

## 进入 Yul 模式

在 Solidity 合约中，我们通过 `assembly {}` 代码块进入 Yul 模式。在这个花括号中，Solidity 的规则全部失效。

在 Yul 中，没有 `bool`、`string` 这些类型，所有东西全部都是 **32字节（256位）的数字**。

- `true` 就是 `1`
- `address` 就是一个数值
- `string` 只是一个内存指针，相当于路牌，跟着这个路牌可以找到房间继而找到数据

## 变量声明与赋值

我们在 Yul 中不能像 Solidity 一样写 `uint x = 1`。

- 我们初始化必须使用关键字 `let`。
- 并且初始化的时候必须声明初始的值。
- 赋值符号也变了，不是 `=` 而变成了 `:=`。
- 重新赋值的时候不需要使用 `let`，直接 `:=` 即可。

### 语法对比表

| Solidity | Yul (Assembly) | 说明 |
| :--- | :--- | :--- |
| `uint256 x = 123;` | `let x := 123` | 变量声明 |
| `x = 456;` | `x := 456` | 重新赋值 |
| `uint256 y;` (默认0) | `let y := 0` | 必须显式初始化 |
| `uint256 z = x + y;` | `let z := add(x, y)` | 使用操作码函数 |

**注意**：Yul 中不支持 `+` `-` `*` 等运算符号，必须使用 EVM 的操作码 (Opcode)，比如 `add()`, `sub()`, `mul()` 等等。

## 块 (Block) 与 作用域

Yul 中的花括号 `{}` 不仅仅是用来分组代码的，它更像是**变量的羊圈**。

**块**是用花括号 `{}` 包裹起来的一段代码区域，在 Yul 中极其重要。

- 在一个**块内部**定义的变量，出了这个块就会**立即销毁**（从栈里弹出），外部无法访问。
- 但是**块内部**的代码可以访问**外部**的变量。

这对于 Gas 优化非常重要，因为尽早销毁不需要的变量能减轻堆栈的压力。

### 示例

```solidity
assembly {
    // 赋值 a 为 10
    let a := 10
    
    {
        // 打开一个块，定义一个临时变量 b，赋值为 20
        let b := 20
        // 让 c 等于 a + b (可以访问外部变量 a)
        let c := add(a, b)
    }
    
    // 块关闭之后，c 和 b 都会被弹出栈，所以就相当于没有了
    
    // 下面这行会报错，因为 b 已经不在上下文中了
    // let d := b 
}
```

## 函数 (Function)

Yul 中的函数和 Solidity 中的函数长得很像，但有几个关键的区别。

语法结构：
```yul
function 函数名(变量1, 变量2, ...) -> 返回值1, 返回值2, ... {
    // 函数体
}
```

- 我们**不需要 return** 任何值，只需要将结果**赋值给返回的变量**就可以了。函数结束时，该变量的值就是返回值。
- Yul 原生支持多值返回，其实是把数据留在了栈顶，这个操作非常高效。
- 没有 `public` 等修饰符，Yul 函数在当前的 `assembly` 块中都是可见的。

### 示例

```yul
assembly {
    // 定义一个计算平方的函数
    function square(input) -> result {
        result := mul(input, input)
    }

    // 调用它
    let val := 10
    let res := square(val) // res 变成 100
}
```
看到没有，直接把结果赋值给了 `result` 变量，结束的时候就自动返回这个值了，不需要 `return`。

## 流程控制：If 语句

Yul 中的 `if` 和 Solidity 的关键区别在于：**没有 `else`, 也没有 `else if`**。

如果你想实现：
```python
if A:
    X = C
else:
    X = B
```

在 Yul 中只能这么写（设置默认值模式）：
```yul
X := B
if A {
    X := C
}
```
因为没有 `else`，所以我们直接把 `else` 的情况写为默认初始值。当 `if` 条件成立的时候，就会替换掉默认的值来达到效果。当然，你也可以写两个独立的 `if` 来构成逻辑。

为了写 `if`，我们需要用到以下比较操作符：

- `lt(a, b)`: **less than**，判断 a 是否小于 b
- `gt(a, b)`: **greater than**，判断 a 是否大于 b
- `eq(a, b)`: **equal**，判断 a 是否等于 b
- `iszero(a)`: 判断是否为 0 (用于取反，或者检查 bool 假)

### 示例：Max 函数

因为没有 `else`，我们可以这样写 `max` 函数：

```yul
function max(a, b) -> result {
    result := b
    if gt(a, b) {
        result := a
    }
}
```
这样一个简单的 `max` 函数就完成了，是不是非常简单？

## 循环 (Loops)

Yul 的循环也和 Solidity 循环不一样，结构如下：

```yul
for { 初始化 } 循环条件 { 迭代后的操作 } {
    // 循环内部
}
```

1.  **初始化**：只做一次，在循环开始的时候。比如 `let a := 0`
2.  **循环条件**：每次循环前判断，如果符合条件 (非0) 就继续。比如 `lt(a, 5)`
3.  **迭代后操作**：在每次循环体执行完之后要干的事。比如 `a := add(a, 1)`

### 示例

```yul
// 这是一个循环五次的逻辑
for { let i := 0 } lt(i, 5) { i := add(i, 1) } {
    // 循环体代码
}
```

## Switch 语句

不是说 `if` 没有 `else` 吗？这时候要多情况判断应该怎么办？**Switch** 就登场了。

其实在 Yul 中，`switch` 比 `if` 更加好用。结构如下：

```yul
switch 变量
case 值A {
    // 执行代码
}
case 值B {
    // 执行代码
}
default {
    // 都不匹配的时候执行，类似于 else
}
```

**注意**：`case` 后面必须跟的是**常量**，不能是变量或者表达式。

### 示例

```yul
function getStatus(code) -> status {
    switch code
    case 1 {
        status := 100 // 激活状态
    }
    case 2 {
        status := 200 // 暂停状态
    }
    default {
        status := 0   // 未知状态
    }
}
```

## 实战练习：权限检查器

最后我们自己做一个练习。我们要写一个极其简单的“权限检查器”函数 `checkAccess`。

**需求**：
1. 函数接收一个参数 `userType`（用户类型）。
2. 返回一个参数 `accessLevel`（权限等级）。

**逻辑如下**：
- 如果 `userType` 是 1 (Admin)，`accessLevel` 设为 99。
- 如果 `userType` 是 2 (User)，`accessLevel` 设为 10。
- **特殊情况**：如果是其他用户，我们需要在一个代码块 `{}` 里做点“额外检查”（模拟）：
    - 先默认 `accessLevel` 为 0。
    - 如果 `userType` 大于 5 (`gt(userType, 5)`)，把 `accessLevel` 设为 1 (Guest)。
    - *(提示：这里可以用 `default` 分支，然后在里面用 `if`)*

### 参考答案

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
        // 默认权限为 0
        accessLevel := 0
        
        // 额外检查块
        {
            if gt(userType, 5) {
                accessLevel := 1
            }
        }
    }
}
```
