# 【Solidity Yul Assembly】1.3 | Storage Slots

**读取和修改存储变量。**

在 Solidity 中，我们可以通过以下代码来存储变量：  
``` solidity
contract StorageBasics {
    uint256 x;

    function setX(uint256 newVal) external {
        x = newVal;
    }

    function getX() external view returns (uint256) {
        return x;
    }
}
```
但是在 Yul 中该如何实现呢？让我们先来看一个错误的实现版本：  
``` solidity
function getXYul() external pure returns (uint256 ret) {
    assembly {
        ret := x
    }
}
```
这个代码会报错：`TypeError: Only local variables are supported. To access storage variables, use the ".slot" and ".offset" suffixes.`
报错信息指出，只有局部变量可以这样访问，而要访问存储变量，我们需要使用 ".slot" 和 ".offset" 后缀。
## 使用 .slot 后缀获取变量存储槽
首先，我们可以通过 .slot 来获取变量的存储槽位置：
``` solidity
function getXYul() external pure returns (uint256 ret) {
    assembly {
        ret := x.slot
    }
}
```
在这里，ret 的值为 0，说明变量 x 被存储在第 0 个槽中。

## 使用 sload(slot) 读取存储数据
接下来，我们可以使用 sload 来读取指定槽中的数据：  
``` solidity
function getXYul() external view returns (uint256 ret) {
    assembly {
        ret := sload(x.slot)
    }
}
```
在这个例子中，我们使用 sload 读取了 x 所在的第 0 个槽中的数据。由于我们读取了链上的存储变量，函数修饰符也需要从 pure 改为 view。  

## 使用 sstore(slot, value) 修改存储数据
我们还可以使用 sstore 来修改指定槽中的数据。以下是一个例子：  
``` solidity
contract StorageBasics {
    uint256 x = 11;
    uint256 y = 22;
    uint256 z = 33;

    function getVarYul(uint256 slot) external view returns (uint256 ret) {
        assembly {
            ret := sload(slot)
        }
    }

    function setVarYul(uint256 slot, uint256 value) external {
        assembly {
            sstore(slot, value)
        }
    }
}
```
在这个示例中，我们演示了如何使用 setVarYul 函数直接修改指定槽中的数据。不过需要注意的是，这种操作存在一定的风险。  

## 变量打包和存储槽共享
最后，让我们来看一个变量打包的例子：  
``` solidity
contract StorageBasics {
    uint256 x = 11;
    uint256 y = 22;
    uint256 z = 33;
    uint128 a = 1;
    uint128 b = 2;

    function getSlot() external pure returns (uint256 slot) {
        assembly {
            slot := b.slot
        }
        // returns 3
    }

    function getVarYul(uint256 slot) external view returns (bytes32 ret){
        assembly {
            ret := sload(slot)
        }
        // returns 0x0000000000000000000000000000000100000000000000000000000000000002
    }
}
```
在这个例子中，变量 a 和 b 共享第 3 个存储槽。那么，如何读取和修改同一槽中的指定变量呢？我们将在下一讲中继续探讨这个问题。

**总结：**  
今天我们了解了如何在 Yul 中操作 Solidity 合约的存储变量，尤其是如何使用 sload 和 sstore 进行读写操作。我们还看到了多个变量共享一个存储槽的情况。虽然这些操作很强大，但也要小心使用，避免意外修改数据。接下来，我们会继续深入探讨在同一槽中精确读取和修改变量的方法。