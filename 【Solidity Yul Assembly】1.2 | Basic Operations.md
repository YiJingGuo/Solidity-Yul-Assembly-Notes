for 循环与 if 语句。

## for 循环
以下是一个简单的检查是否是素数的方法，把输入的数 x 与 2 到 x 的一半的每个数字进行模运算，看模是否是 0.
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
可以看出 for 的使用和平时差不多，都有起始位置，终止条件，计数器，每次迭代的代码。  
也可以写成以下的样子，不过该有 {} 的地方需要有。
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

## if 语句
**if 后面是 非0 的数则是true，是 0 则是 false.**  
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

**通常会使用 iszero 来判断**，iszero(0) 这个将返回 32 字节的 000..01.  
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

**not(0) 的意思不是“非0”，而是将每位都取反**，故 not(0) 将返回`0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff`, 从而使结果判断为真。  
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

**只有 if 语句，没有 else 语句。**  
`lt(x, y)` 用于判断 x 是否小于 y. 如果 x < y，则为1，否则为0.
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
如果想看完整的指令列表，请点击[这里](https://docs.soliditylang.org/en/develop/yul.html#evm-dialect)。

**总结：**

1. **for 循环**:
   - 用法类似其他编程语言，包括起始位置、终止条件、计数器、和迭代内容。
   - 示例展示了如何用 `for` 循环检查一个数是否为素数。

2. **if 语句**:
   - 判断条件为非零即为 `true`，为零即为 `false`。
   - 示例说明了 `if` 的基本用法，以及如何通过 `iszero` 判断条件为 `false`。

3. **没有 else 语句**:
   - Yul 中没有 `else`，可以通过多重 `if` 实现类似 `else` 的逻辑。

