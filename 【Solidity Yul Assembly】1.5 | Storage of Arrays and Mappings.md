# 【Solidity Yul Assembly】1.5 | Storage of Arrays and Mappings

## 数组
### 固定长度的数组
对于固定长度的数组，处理方式相对简单。
``` solidity
contract StorageComplex {
    uint256[3] fixedArray;
    
    constructor() {
        fixedArray = [99, 999, 9999];
    }
    
    function fixedArrayView(uint256 index) external view returns (uint256 ret) {
        assembly {
            ret := sload(add(fixedArray.slot, index))
        }
    }
}
```
`uint256[3] fixedArray;` 就相当于声明了三个 `uint256` 类型的变量，这些变量依次占据从槽 0 开始的存储位置。

### 动态长度数组
动态长度的数组处理起来稍微复杂一些。
``` solidity
contract StorageBasics {
    uint256[] bigArray;
    uint8[] smallArray;

    constructor() {
        bigArray = [10, 20, 30];
        smallArray = [1, 2, 3];
    }

    function bigArrayLength() external view returns (uint256 ret) {
        assembly {
            ret := sload(bigArray.slot)
        }
    } // returns 3
}
```
可以看到槽 0 存放的数据是 3, 表示数组 bigArray 的当前长度。那么，数组中的元素存放在哪里呢？
计算方法为：`keccak256(slot) + index`, 即“存放长度的槽的哈希”就是开始存放元素的存储槽的位置，然后从这个位置开始依次存放元素。
``` solidity
function readBigArrayLocation(uint256 index) external view returns (uint256 ret) {
    uint256 slot;
    assembly {
        slot := bigArray.slot
    }
    bytes32 location = keccak256(abi.encode(slot));
    assembly {
        ret := sload(add(location, index))
    }
}
```
但是，对于非 uint256 类型的动态数组，几个元素会共用一个槽。
``` solidity
function smallArrayLength() external view returns (uint256 ret) {
    assembly {
        ret := sload(smallArray.slot)
    }
} // returns 3

function readSmallArrayLocation(uint256 index) external view returns (bytes32 ret) {
    uint256 slot;
    assembly {
        slot := smallArray.slot
    }
    bytes32 location = keccak256(abi.encode(slot));
    assembly {
        ret := sload(add(location, index))
    }
} // input 0 returns 0x0000000000000000000000000000000000000000000000000000000000030201
```
## 映射（Mapping）
### 单层映射
映射的存储方式与动态数组相似。不同之处在于，动态数组存储槽的位置是 `keccak256(slot)`，然后加上索引；而映射则是将 `slot` 和 `key` 连接后再做哈希处理，即 `keccak256(key, slot)`。    
``` solidity
contract StorageComplex {
    mapping(uint256 => uint256) public myMapping;
    constructor() {
        myMapping[10] = 5;
        myMapping[11] = 6;
    }
    function getMapping(uint256 key) external view returns (uint256 ret) {
        uint256 slot;
        assembly {
            slot := myMapping.slot
        }

        bytes32 location = keccak256(abi.encode(key, uint256(slot)));

        assembly {
            ret := sload(location)
        }
    } // input 10 returns 5
```
### 嵌套映射
对于嵌套映射，需要进行多次哈希计算，比如对于两次的嵌套计算方式为：`keccak256(key2, keccak256(key1, slot))`.
``` solidity
contract StorageComplex {
    mapping(uint256 => mapping(uint256 => uint256)) public nestedMapping;

    constructor() {
        nestedMapping[2][4] = 7;
    }

    function getNestedMapping() external view returns (uint256 ret) {
        uint256 slot;
        assembly {
            slot := nestedMapping.slot
        }

        bytes32 location = keccak256(abi.encode(uint256(4), keccak256(abi.encode(uint256(2), uint256(slot)))));

        assembly {
            ret := sload(location)
        }
    }  // returns 7
}
```
### 映射与数组的结合
直接来看例子：
``` solidity
contract StorageComplex {
    mapping(address => uint256[]) public addressToList;

    constructor() {
        addressToList[0x5B38Da6a701c568545dCfcB03FcB875f56beddC4] = [42, 1337, 777];
    }

    function lengthOfNestedList() external view returns (uint256 ret) {
        uint256 addressToListSlot;
        assembly {
            addressToListSlot := addressToList.slot
        }

        bytes32 location = keccak256(abi.encode(address(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4), uint256(addressToListSlot)));
        assembly {
            ret := sload(location)
        }
    }  // returns 3
}
```
以上 `lengthOfNestedList` 方法用于获取映射对应数组的长度，与单层映射类似，计算方式为 `keccak256(key, slot)`。不同之处在于，这个槽存储的是数组的长度，而单层映射中，这个槽存储的是具体的元素。  
那么，如何获取数组中的元素呢？  
回顾我们之前获取数组元素的方法：`keccak256(slot) + index`。这意味着我们需要先对存放数组长度的槽进行哈希计算，然后加上索引。也就是说把 `keccak256(key, mapping_slot)` 代入到 `slot` 中，那么，计算公式为：`keccak256(keccak256(key, mapping_slot)) + index`，这样我们就可以获取数组中的元素。
``` solidity
function getAddressToList(uint256 index) external  view returns (uint256 ret) {
    uint256 slot;
    assembly{
        slot := addressToList.slot
    }
    bytes32 location = keccak256(abi.encode(keccak256(abi.encode(address(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4), uint256(slot)))));
    assembly {
        ret := sload(add(location, index))
    }
}
```

**总结**

数组和映射的存储机制通过特定的计算公式来确定数据在存储槽中的位置：

1. **固定长度数组**：每个元素顺序占据连续的存储槽，依次递增。
   - 元素位置：`fixedArray.slot + index`

2. **动态长度数组**：有个存储槽保存数组的长度，数组元素的存储位置通过该槽的哈希与索引得出。
   - 数组长度：`sload(bigArray.slot)`
   - 元素位置：`keccak256(slot) + index`

3. **非 `uint256` 类型数组**：多个小元素可能共用一个槽，但计算元素位置的基本方式与 `uint256` 类型相同。

4. **映射**：映射的存储位置通过对 `key` 和 `slot` 进行哈希计算得出。
   - 元素位置：`keccak256(key, slot)`

5. **嵌套映射**：类似单层映射，但需要进行多次哈希计算。
   - 元素位置：`keccak256(key2, keccak256(key1, slot))`

6. **映射与数组的结合**：映射中的数组长度存储在通过哈希计算得出的槽中，数组元素的存储位置通过该槽的哈希与索引得出。
   - 数组长度：`sload(keccak256(key, mapping_slot))`
   - 元素位置：`keccak256(keccak256(key, mapping_slot)) + index`

