# 【Solidity Yul Assembly】2.2 | How Solidity Uses Memory

## Solidity 是如何使用内存的
- Solidity 将地址 [0x00 - 0x20) 和 [0x20 - 0x40) 分配为哈希操作的“临时空间”。
- Solidity 预留地址 [0x40 - 0x60) 作为“空闲内存指针”，通常指向当前未被使用的内存区域。
- 地址 [0x60 - 0x80) 保持为空，作为 32 字节的零值插槽，用于需要零值的场合。
- 内存使用从 [0x80 - ...) 开始。

- Solidity 使用内存的场景：
  - `abi.encode` 和 `abi.encodePacked`.
  - 结构体和数组（但是你需要明确使用 `memory` 关键字）。
  - 在函数参数中，使用了 `memory` 关键字的结构体或数组。
  - 内存中的对象是按顺序排列的，因此数组不能像存储（storage）那样进行动态添加元素（push）。

- 在 Yul 中，
  - 变量存储在内存中的位置即为数组的起始位置。
  - 对于动态数组，需额外添加 32 字节（即 0x20）以跳过数组长度信息，从而访问数组的实际元素。这是因为在 Solidity 中，动态数组会存储数组长度（32 字节），随后才是实际的数组元素。

## 结构体
``` solidity
contract Memory {
    struct Point {
        uint256 x;
        uint256 y;
    }

    event MemoryPointer(bytes32);
    event MemoryPointerMsize(bytes32, bytes32);

    function memPointerV1() external {
        bytes32 x40;
        assembly {
            x40 := mload(0x40)
        }
        emit MemoryPointer(x40);  // 0x0000000000000000000000000000000000000000000000000000000000000080
        
        Point memory p = Point({x: 1, y: 2});
        assembly {
            x40 := mload(0x40)
        }
        emit MemoryPointer(x40);  // 0x00000000000000000000000000000000000000000000000000000000000000c0
    }
}
```
通过以上代码可以看出，“空闲内存指针”本来指向 0x80，分配了结构体后，指向了 0xc0。这个差值为 64 字节，即 2 个 32 字节，正好是结构体中的 `x` 和 `y`。 

``` solidity
function memPointerV2() external {
    bytes32 x40;
    bytes32 _msize;
    assembly {
        x40 := mload(0x40)
        _msize := msize()
    }
    emit MemoryPointerMsize(x40, _msize);
    // 0x0000000000000000000000000000000000000000000000000000000000000080
    // 0x0000000000000000000000000000000000000000000000000000000000000060
    
    Point memory p = Point({x: 1, y: 2});
    assembly {
        x40 := mload(0x40)
        _msize := msize()
    }
    emit MemoryPointerMsize(x40, _msize);
    // 0x00000000000000000000000000000000000000000000000000000000000000c0
    // 0x00000000000000000000000000000000000000000000000000000000000000c0

    assembly {
        pop(mload(0xff))
        x40 := mload(0x40)
        _msize := msize()
    }
    emit MemoryPointerMsize(x40, _msize);
    // 0x00000000000000000000000000000000000000000000000000000000000000c0
    // 0x0000000000000000000000000000000000000000000000000000000000000120
}
```
`msize()` 返回最大可访问的内存地址。当内存中未存放数据时，该值为 0x60；而当存放了一个结构体后，该值与“空闲内存指针”的值一致，为 0xc0。  
在最后一部分代码中使用 `pop(mload(0xff))` 的主要目的是为了读取内存槽 0xff 的数据，即[0xff - 0x120)，让可访问空间变大，因此此时 `msize()` 变为 0x120。

## 定长数组
``` solidity
function fixedArray() external {
    bytes32 x40;
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40);  // 0x0000000000000000000000000000000000000000000000000000000000000080

    uint256[2] memory arr = [uint256(5), uint256(6)];
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40);  // 0x00000000000000000000000000000000000000000000000000000000000000c0
}
```
和结构体类似，存放数组后，“空闲内存指针”从 0x80 指向 0xc0，差值为 64 字节，说明定长数组不会保存数组长度信息。  

## ABI ENCODE
``` solidity
function abiEncode() external {
    bytes32 x40;
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40);  // 0x0000000000000000000000000000000000000000000000000000000000000080

    abi.encode(uint256(5), uint256(19));
    assembly{
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40);  // 0x00000000000000000000000000000000000000000000000000000000000000e0
}
```
同样是两个 uint256, 但是空闲内存指针指向了 0xe0, 这是为什么呢？
![](/img/yul-2.2/1.png)
因为在 bytes 类型的 abi 编码规则中，会先记录字节长度，可以看到此处在内存槽 0x80 处存储的是 0x40 （即 64）。
``` solidity
function abiEncode2() external {
    bytes32 x40;
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40);  // 0x0000000000000000000000000000000000000000000000000000000000000080

    abi.encode(uint256(5), uint128(19));
    assembly{
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40);  // 0x00000000000000000000000000000000000000000000000000000000000000e0
}
```
即使在 abi.encode(uint256(5), uint128(19)) 中使用了 uint128(19)，结果依然与之前相同。这是由于 abi.encode 的填充规则所致。

## ABI ENCODE PACKED
``` solidity
function abiEncodePacked() external {
    bytes32 x40;
    assembly {
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40);  // 0x0000000000000000000000000000000000000000000000000000000000000080

    abi.encodePacked(uint256(5), uint128(19));
    assembly{
        x40 := mload(0x40)
    }
    emit MemoryPointer(x40);  // 0x00000000000000000000000000000000000000000000000000000000000000d0
}
```
空闲内存指针指向了 0xd0, 比之前的 0xe0 要小，这和 `abi.encodePacked` 有关系，我们来看看内存布局情况。
![](/img/yul-2.2/2.png)
可以看到使用 `abi.encodePacked` 编码后的数据如下：
```
0x80:
0x0000000000000000000000000000000000000000000000000000000000000030
0xa0:
0x0000000000000000000000000000000000000000000000000000000000000005
0xc0:
0x0000000000000000000000000000001300000000000000000000000000000000
```
其中，0x30 是字节长度（即 48）。

## 函数参数中的数组
``` solidity
event Debug(bytes32, bytes32, bytes32, bytes32);
function args(uint256[] memory arr) external {
    bytes32 location;
    bytes32 len;
    bytes32 valueAtIndex0;
    bytes32 valueAtIndex1;
    assembly {
        location := arr // 获取 arr 数组在内存中的起始位置。
        len := mload(arr) // 内存槽 arr 存放的是数组的长度。
        valueAtIndex0 := mload(add(arr, 0x20)) // 跳过存长度的内存槽，依次读取数据。
        valueAtIndex1 := mload(add(arr, 0x40))
    }
    emit Debug(location, len, valueAtIndex0, valueAtIndex1);
    // input: [1 ,2]
    // 	"0": "0x0000000000000000000000000000000000000000000000000000000000000080",
	//  "1": "0x0000000000000000000000000000000000000000000000000000000000000002",
	//  "2": "0x0000000000000000000000000000000000000000000000000000000000000001",
	//  "3": "0x0000000000000000000000000000000000000000000000000000000000000002"
}
```

**总结**  
在本文中，我们介绍了 Solidity 内存的四个关键预留槽位，并详细讲解了结构体、定长数组、ABI 编码以及函数参数中的数组如何在内存中存储。