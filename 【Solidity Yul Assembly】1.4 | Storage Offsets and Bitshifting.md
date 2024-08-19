# 【Solidity Yul Assembly】1.4 | Storage Offsets and Bitshifting

## 读取同一个槽中的指定数据
首先来看一个简单的示例合约：
``` solidity
contract StorageBasics {
    uint128 public C = 4;
    uint96 public D = 6;
    uint16 public E = 8;
    uint8 public F = 1;

    function readBySlot(uint256 slot) external view returns (bytes32 value){
        assembly {
            value := sload(slot)
        }
    }
}
```
在这个合约中，C、D、E 和 F 共享第 0 个存储槽。我们可以通过 readBySlot 函数读取第 0 个槽的内容，其结果如下：
`0x0001000800000000000000000000000600000000000000000000000000000004`
接下来，我们可以通过`.offset`读取 E 的偏移量：
``` solidity
function getOffsetE() external pure returns (uint256 slot, uint256 offset) {
    assembly {
        slot := E.slot
        offset := E.offset
    }
}
```
获取 E 的偏移量为 28，意味着我们需要将槽右移 28 个字节（224位，正好为 128+96 ）能读取到 E 的值。以下是具体的实现：
``` solidity
function readE() external view returns (bytes32 e) {
    assembly {
        let value := sload(E.slot)
        let shifted := shr(mul(E.offset, 8), value)

        // equivalent to
        // 0x000000000000000000000000000000000000000000000000000000000000ffff
        e := and(0xffff, shifted)
    }
}
```
读取 E 的做法是，将槽 0 中的内容读出，然后对该值右移 28 * 8 位。右移后，由于 E 前面还有值 F, 故需要用掩码把 E 前面的值全成为 0.
读出 E 的值为`0x0000000000000000000000000000000000000000000000000000000000000008`
我们也可以使用除法来代替右移，但需要注意的是，除法的 gas 费消耗比右移大：
``` solidity
function readEalt() external view returns (bytes32 e) {
    assembly {
        let value := sload(E.slot)
        let shifted := div(value, 0x100000000000000000000000000000000000000000000000000000000) // 1后面56个0
        e := and(0xffff, shifted)
    }
}
```

## 修改同一个槽中的指定数据
接下来，我们来看如何修改槽中的指定数据。首先回顾下 and/or 的逻辑： 
- V and 00 = 00
- V and FF = V
- V or 00 = V
  
在修改槽中数据时，我们通常遵循以下步骤：
1. 使用 0xfff...000...fff 掩码和槽中数据进行“与”运算，将指定位置的数据清零。
2. 将修改后的数据位移到刚清零的位置，然后与第一步处理后的数据进行“或”运算，将新数据写入槽中。

以下是具体实现：
``` solidity
function writeToE(uint16 newE) external {
    assembly{
        // newE = 0x000000000000000000000000000000000000000000000000000000000000000a
        let c := sload(E.slot) // slot 0
        // c = 0x0001000800000000000000000000000600000000000000000000000000000004
        let clearedE := and(c, 0xffff0000ffffffffffffffffffffffffffffffff)
        // mask     = 0xffff0000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff
        // c        = 0x0001000800000000000000000000000600000000000000000000000000000004
        // clearedE = 0x0001000000000000000000000000000600000000000000000000000000000004
        let shiftedNewE := shl(mul(E.offset, 8), newE)
        // shiftedNewE = 0x0000000a00000000000000000000000000000000000000000000000000000000
        let newVal := or(shiftedNewE, clearedE)
        // shiftedNewE = 0x0000000a00000000000000000000000000000000000000000000000000000000
        // clearedE    = 0x0001000000000000000000000000000600000000000000000000000000000004
        // newVal      = 0x0001000a00000000000000000000000600000000000000000000000000000004
        sstore(C.slot, newVal)
    }
}
```
我们首先读取槽中的值 c，然后使用掩码将 E 的位置清零。接着，将新值 newE 进行左移操作，将其对齐到 E 的位置。最后，通过“或”运算将新值写入槽中。

**总结**
在 Solidity 中处理存储槽的读取与修改时，位移和掩码操作是非常重要的工具。通过这些操作，可以精准地控制存储数据的位置和内容。

