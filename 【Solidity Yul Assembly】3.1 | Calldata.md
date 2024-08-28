# 【Solidity Yul Assembly】3.1 | Calldata

## 约定俗成
- Solidity 的普及已经促成了一种关于如何使用交易数据 (tx.data) 的约定。
- 当向一个钱包地址发送交易时，通常不会在 tx.data 中添加任何数据，除非你想发送一条消息给该地址的所有者。
- 而在向一个智能合约发送交易时，tx.data 的前四个字节（即前8个十六进制字符）用来指定你要调用的函数。这四个字节被称为函数选择器（function selector）。随后的字节是按ABI编码的函数参数。关于ABI的更多信息，可以参考[这里](https://docs.soliditylang.org/en/develop/abi-spec.html#abi)。
- Solidity 期望函数选择器后的字节长度始终是 32 的倍数。这是一种约定，便于解析交易数据。如果发送的字节长度不是 32 的倍数，Solidity会忽略多余的字节。
- 相较之下，Yul 合约可以根据需要自由地处理任意长度的交易数据。

## 概述
- 函数选择器是函数签名经过 Keccak-256 哈希计算后的前四个字节。
- 例子1(ERC20)
  - balanceOf(address _address) -> keccak256("balanceOf(address)") -> 0x70a08231
  - balanceOf(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4) -> 0x70a082310000000000000000000000005B38Da6a701c568545dCfcB03FcB875f56beddC4
- 例子2(ERC1155)
  - balanceOf(address _address, uint256 id) -> keccak256("balanceOf(address,uint256)") -> 0x00fdd58e
  - balanceOf(0x5B38Da6a701c568545dCfcB03FcB875f56beddC4,5) -> 0x00fdd58e0000000000000000000000005B38Da6a701c568545dCfcB03FcB875f56beddC40000000000000000000000000000000000000000000000000000000000000005
- 其实无论函数中参数的类型，ABI 总是按照 32 字节编码。

## ABI 规范
- 前端通过 ABI 规范来了解如何格式化交易数据以与智能合约进行通信。
- 在 Solidity 中，函数选择器和参数编码通常是由接口（interface）或者使用 abi.encodeWithSignature("balanceOf(address)",0x...) 方法隐式地创建的。这使得开发者无需手动编写函数选择器和参数编码的逻辑，编译器会自动处理这些细节。
- 然而，如果在 Yul 中对 Solidity 合约进行外部调用，则需要手动编写代码来构造符合ABI规范的函数选择器和参数编码，同时处理 ABI 的编码和解码。这要求你深入了解 ABI 规范的细节，并在 Yul 中实现相关功能，而不是依赖 Solidity 编译器的自动处理。

**总结：**  
本节介绍了 tx.data 在 Solidity 约定俗成的用法，但对于 Yul 合约来说，并没有这些规定，会更加自由，但同时也需要开发者手动编写关于函数选择器的逻辑。从本章开始的后面几节，我们将学习关于合约调用相关的指令和 ABI 编码以及关于 Yul 合约怎么处理函数调用。