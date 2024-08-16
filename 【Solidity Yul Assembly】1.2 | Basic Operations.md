# 【Solidity Yul Assembly】1.2 | Basic Operations

**for 循环与 if 语句。**

在 Solidity 的内联汇编（Yul）中，for 循环和 if 语句的使用与我们在其他编程语言中常见的语法类似，但有一些细微的差异。本文将通过具体示例介绍这些语句的用法。  
## for 循环
以下是一个用于检查某个数 x 是否为素数的简单方法。我们将 x 与 2 到 x 的一半之间的每个数字进行取模运算，以确定是否存在一个数可以整除 x。  
``` solidity
function isPrime(uint256 x) public pure returns (bool p) {
    p = true;
    assembly {
        let halfX := add(div(x, 2), 1) // x / 2 + 1
        for { let i := 2 } lt(i, halfX) { i := add(i, 1) }
        {
            if iszero(mod(x, i)){
                p := 0
                break 
            }
        }
    }
}
```
从上面的代码可以看出，for 循环的使用与其他编程语言非常相似：包括起始位置、终止条件、计数器以及每次迭代时要执行的代码块。我们也可以将其写成如下形式：  
``` solidity
let i := 2
for { } lt(i, halfX) { }
{
    if iszero(mod(x, i)){
        p := 0
        break 
    }
}

i := add(i, 1)
```
不过需要注意，{} 依然是必须的。  

## if 语句
在内联汇编中，if 语句的判断条件是基于数值是否为零来决定的。任何非零值都会被认为是 true，而零则被认为是 false。  
``` solidity
function isTruthy() external pure returns (uint256 result) {
    result = 2;
    assembly {
        if 2 {  // true
            result := 1
        }
    }

    return result;  // returns 1
}

function isFalsy() external pure returns (uint256 result) {
    result = 1;
    assembly {
        if 0 {  // false
            result := 2
        }
    }

    return result; // returns 1
}
```
通常情况下，会使用 iszero 来判断条件是否为 false。iszero(0) 会返回 32 字节的 000...01。  
``` solidity
function negation() external pure returns (uint256 result) {
    result = 1;
    assembly {
        if iszero(0) {
            result := 2
        }
    }

    return result; // returns 2
}
```
需要注意的是，not(0) 的含义并不是“非 0”，而是对每一位进行取反运算。因此，not(0) 将返回 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff，从而使条件判断为 true。   
``` solidity
function unsafeNegationPart1() external pure returns (uint256 result) {
    result = 1;
    assembly {
        if not(0) {
            result := 2 
        }
    }

    return result; // returns 2
}
```

在内联汇编中，if 语句并没有 else 分支。  
例如，判断两个数 x 和 y 之间的最大值：  
``` solidity
function max(uint256 x, uint256 y) external pure returns (uint256 maximum) {
    assembly {
        if lt(x, y) {
            maximum := y
        }
        if iszero(lt(x, y)) {  // there are no else statements
            maximum := x   
        }
    }
}
```
lt(x, y) 用于判断 x 是否小于 y，如果 x < y，则返回 1；否则返回 0。  
如果想看完整的指令列表，请点击[这里](https://docs.soliditylang.org/en/develop/yul.html#evm-dialect)。

**总结：**
在 Solidity 的内联汇编中，for 循环和 if 语句的用法很直观。for 循环用来重复执行一段代码，而 if 语句用来做条件判断。虽然这些语法跟其他编程语言很像，但在内联汇编里，条件判断依赖于数值是否为零，iszero 函数很常用。