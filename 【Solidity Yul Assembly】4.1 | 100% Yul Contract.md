# 【Solidity Yul Assembly】4.1 | 100% Yul Contract

本节将通过简单的纯 Yul 合约示例，来讲解相关的知识点。  
``` yul
object "Simple1" {
    // 构造函数
    code {
        // 复制 runtime 对象的字节码到内存中，然后返回该字节码。
        // runtime 对象的代码将成为合约部署后的代码（即运行时代码）
        datacopy(0x00, dataoffset("runtime"), datasize("runtime"))
        return(0x00, datasize("runtime"))
    }

    // 部署后合约的实际字节码
    object "runtime" {
        code {
            mstore(0x00, 2)
            return(0x00, 0x20)
        }
    }
}
```
## 代码解释
Yul 中的合约可以由一个主对象和多个子对象组成。子对象可以是要部署的代码，也可以是这个合约可以创建的其他合约。

每个对象都有一个 `code` 部分，定义了当对象被执行时所运行的具体代码。比如，`Simple1` 对象中的 `code` 部分可以视作构造函数。而 `runtime` 对象中的 `code` 则是在合约调用时实际运行的代码。

Yul 中的每个命名对象或数据部分都会被序列化，并且可以通过特定的内置函数 `datacopy`、`dataoffset` 和 `datasize` 来访问。在 `Simple1` 对象中的 `code` 部分中，使用 `datacopy` 将 `runtime` 对象的字节码复制到内存地址 `0x00` 开始的位置。`dataoffset("runtime")` 获取了 `runtime` 对象的字节码的起始位置，`datasize("runtime")` 获取了字节码的大小。然后通过 `return` 返回这些字节码。这些字节码在合约部署后将作为合约在区块链上的实际存储内容。  

在这个例子中，部署后的代码非常简单：它只是在调用时返回一个 2。

## 在合约字节码中存储数据
```
object "Simple2" {
    code {
        datacopy(0, dataoffset("runtime"), datasize("runtime"))
        return(0, datasize("runtime"))
    }

    object "runtime" {
        code {
            // 返回 "Message" 数据
            datacopy(0x00, dataoffset("Message"), datasize("Message"))
            return(0x00, datasize("Message"))
        }

        // 存储在合约字节码中
        data "Message" "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. "

    }
}
```
这段 Yul 代码定义了一个简单的合约 `Simple2`，其功能是返回存储在合约字节码中的一段字符串。`data` 指令定义了合约字节码中存储的数据部分。

## 编译和调用
在 Remix 中编译时，需要选择 Yul 作为编译语言才能成功编译纯 Yul 合约。但是，在部署后如何验证合约功能呢？我们可以编写另一个合约来调用它，并返回结果。
``` solidity
interface ISimple {
    function itDoesntMatterBecauseItJustReturns2() external view returns (uint256);
}

contract CallSimple {
    ISimple public target;

    constructor(ISimple _target) {
        target = _target;
    }

    function callSimpleUint() external view returns (uint256) {
        return target.itDoesntMatterBecauseItJustReturns2();
    }

    function callSimpleString() external view returns (string memory) {
        (bool success, bytes memory result) = address(target).staticcall("");
        require(success, "failed");
        return string(result);
    }
}
```
在第一个例子中，Yul 合约不关心你的 calldata 的内容，因此我们任意定义了一个接口名称 `itDoesntMatterBecauseItJustReturns2`。调用的时候 calldata 中会有对应的函数选择器，当然这里接口名字换成其他的都可以。  

在部署纯 Yul 合约 `Simple1` 和 `CallSimple` 合约之后，调用 `callSimpleUint` 函数可以看到返回的是 2，这说明合约功能正常。  

同样地，部署 `Simple2` 合约，调用 `callSimpleString` 将返回长字符串。在 `callSimpleString` 函数中，我们无需调用合约接口，而是使用 `staticcall`。

## 纯 Yul 合约的建议
- 适用的情况
  - 部署许多小合约，且这些合约的字节码形式易于阅读
  - 进行 arbitrage 或者 MEV 提高效率时。
- 不适用的情况
  - 合约需要被验证。在浏览器上还不能对纯 Yul 合约进行验证和开源。
  - 中型或大型合约。因为使用大量的 Yul 代码，犯错的概率也会越大。
