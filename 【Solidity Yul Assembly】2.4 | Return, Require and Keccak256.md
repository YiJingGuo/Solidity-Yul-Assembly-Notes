# 【Solidity Yul Assembly】2.4 | Return, Require and Keccak256

## return
``` solidity
function return2and4() external pure returns (uint256, uint256) {
    assembly{
        mstore(0x00, 2)
        mstore(0x20, 4)
        return(0x00, 0x40)
    }
    // returns 2 4
}
```
在 `assembly` 里，`return(p, s)` 是一条指令，表示结束执行并返回结果。这里的 `p` 是内存的起始位置，而 `s` 则表示数据的长度。上述代码会将内存地址 `0x00` 到 `0x40` 的数据返回，其中包括两个 32 字节的整数：2 和 4。通过这种方式，可以直接从内存中读取和返回所需的数据。

## revert
``` solidity
function requireV1() external view {
    require(msg.sender == 0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2);
}

function requireV2() external view {
    assembly {
        if iszero(eq(caller(), 0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2)) {
            revert(0,0)
        }
    }
}
```
`revert(p, s)` 指令用于中止执行并恢复所有状态变化。它与 `return` 类似，也可以返回数据，在后续章节将会体现。

## keccak256
``` solidity
function hashV1() external pure returns (bytes32) {
    bytes memory toBeHashed = abi.encode(1,2,3);
    return keccak256(toBeHashed);
    // returns 0x6e0c627900b24bd432fe7b1f713f1b0744091a646a9fe4a65a18dfed21f2949c
}

function hashV2() external pure returns (bytes32) {
    assembly {
        let freeMemoryPointer := mload(0x40)

        // store 1, 2, 3 in memory
        mstore(freeMemoryPointer, 1)
        mstore(add(freeMemoryPointer, 0x20), 2)
        mstore(add(freeMemoryPointer, 0x40), 3)

        // update memory pointer
        mstore(0x40, add(freeMemoryPointer, 0x60))

        mstore(0x00, keccak256(freeMemoryPointer, 0x60))
        return(0x00, 0x20)
    }
    // returns 0x6e0c627900b24bd432fe7b1f713f1b0744091a646a9fe4a65a18dfed21f2949c
}
```
`keccak256(p, n)` 用于对内存地址 `p` 到 `p + n` 之间的数据进行哈希处理。在 `hashV2` 函数中，数据 `1, 2, 3` 被存储在内存中，然后使用 `keccak256` 进行哈希计算。结果被存储在内存地址 `0x00`，并返回给调用者。  

**总结**
本章介绍的三条指令 `return(p, s)`、`revert(p, s)` 和 `keccak256(p, n)` 都涉及内存操作，通过指定内存起始地址 `p` 和长度参数 `s` 或 `n` 来确定数据范围。`return` 和 `revert` 都用于结束函数执行，但目的不同：`return` 正常退出并返回数据，而 `revert` 异常退出并恢复状态，同时可能返回错误信息。两者都返回内存数据，但 `revert` 还会回滚状态变化。而 `keccak256` 用于哈希计算，通过指定的内存地址和数据长度，对数据进行哈希运算，并返回一个 32 字节的哈希值。
