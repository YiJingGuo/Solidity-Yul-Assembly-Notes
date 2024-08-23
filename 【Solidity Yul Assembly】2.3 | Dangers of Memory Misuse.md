# 【Solidity Yul Assembly】2.3 | Dangers of Memory Misuse

## 注意事项
- **内存指针管理**：如果你不遵循 Solidity 的内存布局和空闲内存指针规则，你可能会遇到一些严重的 bug。
- **内存解包行为**：内存不会尝试将小于 32 字节的数据类型打包在一起。当数据从存储加载到内存时，即使这些数据在存储中是打包的（如 `uint8` 或 `bool`），在加载到内存中后，每个数据项都会占用完整的 32 字节空间。

### 示例一：内存指针的误用
``` solidity
function breakFreeMemoryPointer(uint256[1] memory foo) external view returns (uint256) {
    assembly {
        mstore(0x40, 0x80) // 人为修改空闲内存指针
    }
    uint256[1] memory bar = [uint256(6)];
    return foo[0]; // 返回 foo[0] 的值
    // input [1000]
    // return 6
}
```
假如我们传入参数 [1000]，正常情况下，这个参数会被写入内存槽 `0x80`，空闲内存指针此时指向 `0xa0`。然而，我们将空闲内存指针强制设为 `0x80`，此时内存中再装入定长数组`bar`，其值 `6` 会被写入到 `0x80` 处，覆盖原有的值。因此，`return foo[0]` 将会返回 `0x80` 地址处的内容，即 `6`。

### 示例二：内存中的数据解包
``` solidity
uint8[] foo = [1,2,3,4,5,6];
function unpacked() external {
    uint8[] memory bar = foo;
}
```
![](/img/yul-2.3/1.png)
在这个示例中，虽然 `foo` 在存储槽中是紧密打包在一个槽中，但当它被读取到内存中时，数据会被解包。每个 `uint8` 数据类型的元素在内存中都会单独占用 32 字节空间。