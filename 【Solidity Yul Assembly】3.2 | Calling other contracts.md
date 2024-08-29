# 【Solidity Yul Assembly】3.2 | Calling other contracts

在 EVM 中，有四种与 call 相关的指令，它们分别是：`call`、`callcode`、`delegatecall` 和 `staticcall`。  
1. call(g, a, v, in, insize, out, outsize)
   - g: 提供给被调用合约的 gas 数量。
   - a: 被调用合约的地址。
   - v: 发送给被调用合约的以太币数量（单位为 wei）。
   - in: 输入数据的内存起始位置。
   - insize: 输入数据的长度。 mem[in…(in+insize)) 
   - out: 输出数据的内存起始位置。
   - outsize: 输出数据的长度。 mem[out…(out+outsize))

2. callcode(g, a, v, in, insize, out, outsize)
    `callcode` 与 `call` 的区别在于，`callcode` 在调用合约时只使用被调用合约的代码，但保持在当前合约的上下文中运行。这意味着状态变量的读写和主网币的转移都会发生在调用合约的上下文中，而不是被调用合约的上下文。

3. delegatecall(g, a, in, insize, out, outsize)
    `delegatecall` 类似于 `callcode`，也是在当前合约的上下文中运行被调用合约的代码，但与 `callcode` 的区别在于，`delegatecall` 保留了 `caller` 和 `callvalue`。这意味着被调用合约可以访问调用者地址和发送的主网币数额。

4. staticcall(g, a, in, insize, out, outsize)
    `staticcall` 与 `call` 类似，主要区别在于 `staticcall` 不允许修改合约状态，因而具有较低的 `gas` 成本。它通常用于查询合约状态而不做修改的场景。

## call 的使用示例
``` solidity
contract OtherContract {
    // 函数选择器: "0c55699c": "x()"
    uint256 public  x;

    // 函数选择器: "4018d9aa": "setX(uint256)"
    function setX(uint256 _x) external {
        x = _x;
    }
}

contract ExternalCalls {
    // 使用 call 调用 setX 函数
    function externalStateChangingCall(address _a) external payable {
        assembly {
            mstore(0x00, 0x4018d9aa)
            mstore(0x20, 999)
            let success := call(gas(), _a, callvalue(), 28, add(4, 32), 0x00, 0x00)
            if iszero(success) {
                revert(0,0)
            }
        }
    }
}
```
在这个例子中，`externalStateChangingCall` 函数使用了 `call` 指令来调用另一个合约的 `setX` 函数，并传递一个参数。`gas()`用于获取当前执行上下文中剩余的 gas 数量。`callvalue()` 表示当前调用发送的主网币数量，这里使用了 `payable` ，如果没有使用 `payable` 修饰符，该值一定为 0，可以直接填写 0。由于把 `0x4018d9aa` 和 `999` 分别放到内存槽 `0x00` 和 `0x20` 中，此时内存中为 000...4018d9aa000...999，所以这里起始位置 `in` 填的是 28（从`4018d9aa`开始读取）。另外，`insize` 是 36，这是因为我们有 4 字节的函数选择器和 32 字节的参数，总共 36 字节。由于 setX 没有返回值，所以最后两个字段都为 0x00。

## staticcall 的使用示例
### 基本用法
``` solidity
contract OtherContract {
    // 函数选择器: "9a884bde": "get21()",
    function get21() external pure returns(uint256) {
        return 21;
    }
}

contract ExternalCalls {
    // 使用 staticcall 调用 get21 函数
    function externalViewCallNoArgs(address _a) external view returns (uint256) {
        assembly {
            mstore(0x00, 0x9a884bde)
            // 000000000000000000000000000000000000000000000000000000009a884bde
            //                                                         |       |
            //                                                         28      32
            let success := staticcall(gas(), _a, 28, 4, 0x00, 0x20)
            if iszero(success) {
                revert(0,0)
            }
            return(0x00, 0x20)
        }
    }
    // 返回值为 21
}
```
分别部署以上两个合约，然后调用 `externalViewCallNoArgs` 函数并且填入 `OtherContract` 合约地址，可以看到返回了 21。`mstore(0x00, 0x9a884bde)` 先把需要调用的函数选择器放入内存槽 0x00 中。`staticcall(gas(), _a, 28, 32, 0x00, 0x20)` 由于是只读访问，所以没有 `call(g, a, v, in, insize, out, outsize)` 的字段 `v`。调用后有返回值，0x00 到 (0x00 + 0x20) 是输出数据的内存存放地址。

### 使用 revert 返回数据
在 2.4 节中，我们说了 revert 也可以返回数据，在此处填坑，以下就是使用 revert 返回数据的例子。
``` solidity
contract OtherContract {
    // 函数选择器: "73712595": "revertWith999()",
    function revertWith999() external pure returns (uint256) {
        assembly {
            mstore(0x00, 999)
            revert(0x00, 0x20)
        }
    }
}

contract ExternalCalls {
    // 调用 OtherContract 合约的 revertWith999 函数
    function getViaRevert(address _a) external view returns (uint256) {
        assembly{
            mstore(0x00, 0x73712595)
            pop(staticcall(gas(), _a, 28, 4, 0x00, 0x20))
            return(0x00, 0x20)
        }
    }
    // 返回值为 999
}
```
`revertWith999()` 函数总是回滚，并返回一个 32 字节的 999。  
- mstore(0x00, 999)：在内存位置 0x00 处存储值 999。
- revert(0x00, 0x20)：回滚，并返回从内存位置 0x00 开始的 32 字节数据。这个数据就是 999。
在 `getViaRevert` 中，我们使用 `pop()` 丢弃 `staticcall` 返回的布尔值，因为在这里我们不关心调用是否成功，只想获取 `revert` 的返回值。

### 处理多个参数
``` solidity
contract OtherContract {
    // 函数选择器: "196e6d84": "multiply(uint128,uint16)",
    function multiply(uint128 _x, uint16 _y) external pure returns (uint256) {
        return _x * _y;
    }
}

contract ExternalCalls {
    function callMultiply(address _a) external view returns (uint256 result) {
        assembly {
            let mptr := mload(0x40)
            mstore(mptr, 0x196e6d84)
            mstore(add(mptr, 0x20), 3)
            mstore(add(mptr, 0x40), 11)
            mstore(0x40, add(mptr, 0x60)) // 更新空闲内存指针 3 x 32 bytes
            // 00000000000000000000000000000000000000000000000000000000196e6d84
            // 0000000000000000000000000000000000000000000000000000000000000003
            // 000000000000000000000000000000000000000000000000000000000000000b
            let success := staticcall(gas(), _a, add(mptr, 28), 68, 0x00, 0x20)
            if iszero(success) {
                revert(0, 0)
            }
            result := mload(0x00)
        }
    }
    // 返回值为 33
}
```
在这个例子也可以看出 `multiply(uint128 _x, uint16 _y)` 参数不是 uint256 类型的，但根据 ABI 编码要求，参数在内存中要对齐为 32 字节。`insize` 是 68，是因为函数选择器 4 字节 + 两个参数 32 * 2 字节。

### 处理未知返回长度
``` solidity
contract OtherContract {
    // 函数选择器: "7c70b4db": "variableReturnLength(uint256)",
    function variableReturnLength(uint256 len) external pure returns (bytes memory) {
        bytes memory ret = new bytes(len);
        for (uint256 i = 0; i < ret.length; i++) {
            ret[i] = 0xab;
        }
        return ret;
    }
}

contract ExternalCalls {
    function unknownReturnSize(address _a, uint256 amount) external view returns (bytes memory) {
        assembly {
            mstore(0x00, 0x7c70b4db)
            mstore(0x20, amount)

            let success := staticcall(gas(), _a, 28, add(4, 32), 0x00, 0x00)
            if iszero(success) {
                revert(0, 0)
            }

            returndatacopy(0, 0, returndatasize())
            return(0, returndatasize())
        }
    }
}
```
在这个例子中，`variableReturnLength` 函数根据输入的 `len` 返回一个字节数组，数组的每个字节都为 `0xab`。  
`returndatasize()` 用于获取最近一次返回数据的字节长度，而 `returndatacopy(t, f, s)` 则用于将返回数据的一部分复制到内存中：
   - t: 目标内存位置，表示将要复制到的内存位置。
   - f: 源数据位置，表示要从返回数据中复制的起始位置。
   - s: 复制的字节数，表示要复制的数据长度。
  
## delegatecall 的使用示例
在了解 delegatecall 的用法时，我们可以参考 OpenZeppelin 的代理模式实现。以下是 OpenZeppelin 实现的一部分代码：
``` solidity
function _delegate(address implementation) internal virtual {
        assembly {
            // Copy msg.data. We take full control of memory in this inline assembly
            // block because it will not return to Solidity code. We overwrite the
            // Solidity scratch pad at memory position 0.
            calldatacopy(0, 0, calldatasize())

            // Call the implementation.
            // out and outsize are 0 because we don't know the size yet.
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)

            // Copy the returned data.
            returndatacopy(0, 0, returndatasize())

            switch result
            // delegatecall returns 0 on error.
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }
```
- `calldatasize()`: 返回当前调用的 calldata 的大小。
- `calldatacopy(t, f, s)` : 将 calldata 复制到内存中。t 是目标内存位置，f 是复制的起始位置，s 是数据长度。  
代码中提到的注释解释了这样的问题：如果 calldata 数据很大，超过了 64 字节，会覆盖 0x40 槽之后的位置。实际上，由于这是纯 Yul 代码段，接下来的操作不涉及 Solidity 代码，因此可以直接使用全部内存空间。
- `delegatecall(g, a, in, insize, out, outsize)`: 这里的 out 和 outsize 保持为 0，因为返回值大小未知。
- `returndatacopy(0, 0, returndatasize())`：将 delegatecall 的返回数据复制到内存中。参数 (0, 0, returndatasize()) 表示从位置 0 开始复制，复制到内存位置 0，长度为 returndatasize()。

**总结：**  
我们本节学习了 `call`、`callcode`、`delegatecall` 和 `staticcall` 四种与合约调用相关的指令。展示了如何使用这些指令进行跨合约调用、状态修改、查询、处理多个参数、处理未知返回长度等操作。此外，还介绍了 OpenZeppelin 的代理模式中的 delegatecall 实现。