# 【Solidity Yul Assembly】4.3 | What We Didn't Cover

虽然在之前的课中我们已经涵盖了 Yul 的大部分常用指令，但这次我们会简要介绍那些使用频率较低的指令，为本系列课程画上句号。  

| 指令                | 描述                                                                                          |
|---------------------|-----------------------------------------------------------------------------------------------|
| `sdiv(x, y)`        | `x / y`, for signed numbers in two’s complement, 0 if `y == 0`                                |
| `smod(x, y)`        | `x % y`, for signed numbers in two’s complement, 0 if `y == 0`                                |
| `slt(x, y)`         | 1 if `x < y`, 0 otherwise, for signed numbers in two’s complement                             |
| `sgt(x, y)`         | 1 if `x > y`, 0 otherwise, for signed numbers in two’s complement                             |
| `sar(x, y)`         | signed arithmetic shift right `y` by `x` bits                                                 |
| `signextend(i, x)`  | sign extend from `(i*8+7)th` bit counting from least significant                              |
| `addmod(x, y, m)`   | `(x + y) % m` with arbitrary precision arithmetic, 0 if `m == 0`                              |
| `mulmod(x, y, m)`   | `(x * y) % m` with arbitrary precision arithmetic, 0 if `m == 0`                              |
| `byte(n, x)`        | `nth` byte of `x`, where the most significant byte is the 0th byte                            |
| `stop()`            | stop execution, identical to `return(0, 0)`                                                   |
| `invalid()`         | end execution with invalid instruction                                                        |

### 1. `sdiv(x, y)`
- 执行有符号整数除法，适用于二进制补码形式的有符号数。
- 若 `y == 0`，结果返回 0（防止除以零的错误）。
- 例如，`sdiv(10, 3)` 会返回 `3`，而 `sdiv(-10, 3)` 会返回 `-3`。
    
### 2. `smod(x, y)`
- 进行有符号整数取模运算，适用于二进制补码的有符号数。
- 计算 `x` 除以 `y` 后的余数。
- 如果 `y == 0`，返回值为 0。
- 例如，`smod(10, 3)` 返回 `1`，而 `smod(-10, 3)` 返回 `-1`。
    
### 3. `slt(x, y)`
- 如果 `x < y`，返回 `1`，否则返回 `0`，适用于二进制补码的有符号数。
- 例如，`slt(5, 10)` 返回 `1`，而 `slt(10, 5)` 返回 `0`。
    
### 4. `sgt(x, y)`
- 如果 `x > y`，返回 `1`，否则返回 `0`，适用于二进制补码的有符号数。
- 例如，`sgt(-3, -5)` 返回 1，而 `sgt(-5, -3)` 返回 0。

### 5. `sar(x, y)`
- 对 `x` 进行算术右移 `y` 位，保持符号位不变。
- 正数右移时左侧补 `0`，负数右移时左侧补符号位 `1`。
- 例如，`sar(16, 2)` 会将 `16` (二进制为 `10000`) 右移 2 位，结果为 `4` (二进制为 `100`)；`sar(-16, 2)` 会将 `-16` 的二进制右移 `2` 位，结果为 `-4`。

### 6. `signextend(i, x)`
- 对 `x` 进行符号扩展，从第 `(i*8 + 7)` 位起进行符号填充。
- 这用于将较小的整数扩展到较大的整数类型，保持符号位一致。
- 例如，假设 `x` 是 `0x7F`（7 位），扩展后，若是符号扩展，最高位为 0，结果仍为正；若最高位为 1（如 `0xFF`），扩展后该值会成为负数。

### 7. `addmod(x, y, m)`
- 计算 `(x + y) % m`，使用任意精度算术，若 `m == 0`，则结果为 0。
- 例如，`addmod(10, 5, 12)` 会返回 `3`，即 `(10 + 5) % 12 = 3`。

### 8. `mulmod(x, y, m)`
- 计算 `(x * y) % m`，使用任意精度算术，若 `m == 0`，则结果为 0。
- 例如，`mulmod(10, 5, 12)` 返回 `2`，即 `(10 * 5) % 12 = 2`。

### 9. `byte(n, x)`
- 获取 `x` 的第 `n` 字节。
- 例如，`byte(0, 0x12345678)` 返回 `0x12`，而 `byte(2, 0x12345678)` 返回 `0x56`。

### 10. `stop()`
- 停止执行，等同于 `return(0, 0)`。EVM 会立即停止执行，并且不会返回任何数据。

### 11. `invalid()`
- 通过无效指令结束执行，会引发异常。
- 有时为了给你编译合约的一些信息 Solidity 编译器会在 calldata 中插入元数据，所以为了防止元数据被意外执行，它将被加上 invalid 关键字。所以如果你想在 Yul 中做类似的事情，那么可以用到 `invalid()`。

**总结：**  
在最后一节课中，我们探讨了 Yul 中一些使用频率较低但功能强大的指令，如有符号整数运算、位移操作、整数模运算，以及特殊指令 stop() 和 invalid()等。这些指令虽然不常用于日常开发，但在特定场景中，它们能够提供更高的精确度和性能优化。  
  
至此，本系列课程圆满结束！如果你认真学习了整个课程，相信在面对 Solidity 合约中的内联汇编代码时，已经具备独立分析和理解的能力。而且，在编写 Solidity 合约时，你会更加得心应手，并能够开始尝试运用低级语言来优化合约的性能与安全性。希望这些知识能为你的区块链开发之路打下坚实的基础。