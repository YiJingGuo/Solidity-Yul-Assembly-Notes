# 【Solidity Yul Assembly】3.5 | Receiving contract calls

本文将通过两个合约示例进行演示：一个合约几乎完全使用 Yul 编写，作为被调用方；另一个是标准的 Solidity 合约，作为调用方。  
以下是两个示例合约的代码：  
``` solidity
contract CalldataDemo{
    fallback(bytes calldata data) external returns (bytes memory returnData) {
        assembly {
            let cd := calldataload(0) // always loads 32 bytes
            // d2178b0800000000000000000000000000000000000000000000000000000000
            let selector := shr(0xe0, cd) // shift right 224 bits to get the last 4 bytes(32 bits)
            // 00000000000000000000000000000000000000000000000000000000d2178b08

            // unlike other languages, switch doed not "fall through"
            switch selector
            case 0xd2178b08 /* get2() */{
                returnUint(2)
            }
            case 0xba88df04 /* get99(uint256) */ {
                returnUint(getNotSoSecretValue())
            }
            default {
                revert(0,0)
            }

            function returnUint(v) {
                mstore(0, v)
                return(0, 0x20)
            }

            function getNotSoSecretValue() -> r {
                if lt(calldatasize(), 36) {
                    revert(0,0)
                }

                let arg1 := calldataload(4)
                if eq(arg1, 8) {
                    r := 88
                    leave
                }
                r := 99
            }
        }
    }
}

interface ICalldataDemo {
    function get2() external view returns (uint256);
    function get99(uint256) external view returns (uint256);
}

contract CallDemo {
    ICalldataDemo public target;

    constructor(ICalldataDemo _a) {
        target = _a;
    }

    function callGet2() external view returns (uint256) {
        return target.get2();
    }

    function callGet99(uint256 arg) external view returns (uint256) {
        return target.get99(arg);
    }
}
```
## 合约调用流程
1. 首先，部署 `CalldataDemo` 合约并获取其地址。
2. 然后，部署 `CallDemo` 合约，并将 `CalldataDemo` 的地址传递给它。
3. 调用 `CallDemo` 合约的 `callGet2` 方法，预期返回 2。
4. 调用 `callGet99` 方法并传入参数 8，预期返回 88；传入参数 9，预期返回 99。

## 合约解释
- `calldataload(0)`：从 calldata 的第 0 字节开始读取 32 字节的数据。
- `let selector := shr(0xe0, cd)`：将读取到的 calldata 数据右移 224 位，从而获取前 4 个字节（32 位），这些字节通常是函数选择器。
``` solidity
// unlike other languages, switch doed not "fall through"
switch selector
case 0xd2178b08 /* get2() */{
    returnUint(2)
}
case 0xba88df04 /* get99(uint256) */ {
    returnUint(getNotSoSecretValue())
}
default {
    revert(0,0)
}
```
- 在 Yul 中，`switch` 语句不会“贯穿”到下一个 `case`。每个 `case` 执行完后，都会自动中断，无需显式的 `break` 语句。
- Yul 允许定义内部函数，并可以在这些函数中使用 `return` 语句。这不仅是用于返回数据，还可以用于结束当前函数的执行并将控制权交还给调用者。
- `calldataload(4)` 用于跳过前 4 个字节（通常是函数选择器），读取从第 4 字节开始的 32 字节数据。
``` solidity
if eq(arg1, 8) {
    r := 88
    leave
}
r := 99
```
- Yul 中没有 `else` 语句。`leave` 语句用于提前退出当前函数。

**总结：**  
本节介绍了如何使用 Yul 编写合约并被 Solidity 合约调用。正如我们在之前的文章中提到的，使用 Yul 需要手动编写处理函数选择器等功能的逻辑。

