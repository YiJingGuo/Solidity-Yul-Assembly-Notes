# 【Solidity Yul Assembly】3.4 | Transfer of value

在 Solidity 中，实现合约向特定地址转账有多种方法。本文将介绍两种常见的实现方式，以及它们在 Yul 中的对应写法和 gas 费用的比较。
## 示例代码
``` solidity
contract WithdrawV1 {
    constructor() payable {}
    address public constant owner = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;
    function withdraw() external {
        (bool s,) = payable(owner).call{value: address(this).balance}("");
        require(s);
    }
}

contract WithdrawV2 {
    constructor() payable {}
    address public constant owner = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;
    function withdraw() external {
        assembly {
            let s := call(gas(), owner, selfbalance(), 0, 0, 0, 0)
            if iszero(s) { revert (0, 0) }
        }
    }
}
```
在上面的代码中：  
- `WithdrawV1` 合约使用 `Solidity` 的 `call` 方法向 `owner` 地址发送交易，转账金额为合约中的全部余额，消息内容为空。
- 对应的 Yul 写法在 `WithdrawV2` 合约中，用的是 `call(gas(), owner, selfbalance(), 0, 0, 0, 0)`。因为不需要返回数据，所以 `out` 和 `outsize` 都设为 0。

`(bool s,) = payable(owner).call{value: address(this).balance}("");` 语句的作用是向 `owner` 地址发送一笔金额为合约余额的交易，消息体为空。`(bool s, bytes memory returnArr)` 这种写法也可以用于获取返回值，但由于此处只是简单的转账给外部地址，返回值被忽略了。  

在 Solidity 中，另一个常用的转账方法是 `transfer`。例如，使用 `payable(owner).transfer(address(this).balance);` 也可以完成相同的功能。对应的 Yul 代码为：`let s := call(2300, owner, selfbalance(), 0, 0, 0, 0)`，其中 2300 固定 gas 限制用于防止接收方执行更多复杂操作，只能发送事件或简单的状态更新。  

## Gas 费比较
尽管 WithdrawV1 和 WithdrawV2 两种合约功能相同，但在部署和每次调用时的 gas 费用存在差异。  

1. 部署 `WithdrawV1`, 并且部署的时候发送 1 个 ether, 总花费 `156689` gas, 部署的字节码为：
``` bash
0x60806040526101e0806100136000396000f3fe608060405234801561001057600080fd5b50600436106100365760003560e01c80633ccfd60b1461003b5780638da5cb5b14610045575b600080fd5b610043610063565b005b61004d6100f0565b60405161005a9190610149565b60405180910390f35b6000735b38da6a701c568545dcfcb03fcb875f56beddc473ffffffffffffffffffffffffffffffffffffffff164760405161009d90610195565b60006040518083038185875af1925050503d80600081146100da576040519150601f19603f3d011682016040523d82523d6000602084013e6100df565b606091505b50509050806100ed57600080fd5b50565b735b38da6a701c568545dcfcb03fcb875f56beddc481565b600073ffffffffffffffffffffffffffffffffffffffff82169050919050565b600061013382610108565b9050919050565b61014381610128565b82525050565b600060208201905061015e600083018461013a565b92915050565b600081905092915050565b50565b600061017f600083610164565b915061018a8261016f565b600082019050919050565b60006101a082610172565b915081905091905056fea26469706673582212202fb4272bb834410eb632a8000f7cdb5b754d7005b224430b6fe12912b8aebfdf64736f6c63430008130033
```
共 499 个字节。  
调用 `withdraw` 函数时，花费 `28300` gas.

2. 部署 `WithdrawV2`, 并且部署的时候发送 1 个 ether, 总花费 `116939` gas, 部署的字节码为：
``` bash
0x6080604052610127806100136000396000f3fe6080604052348015600f57600080fd5b506004361060325760003560e01c80633ccfd60b1460375780638da5cb5b14603f575b600080fd5b603d6059565b005b60456083565b6040516050919060d8565b60405180910390f35b60008060008047735b38da6a701c568545dcfcb03fcb875f56beddc45af180608057600080fd5b50565b735b38da6a701c568545dcfcb03fcb875f56beddc481565b600073ffffffffffffffffffffffffffffffffffffffff82169050919050565b600060c482609b565b9050919050565b60d28160bb565b82525050565b600060208201905060eb600083018460cb565b9291505056fea2646970667358221220af42f5740f7b7833251ebbe9ff0f336b999780ca467cd1bcdcfd1248a8a0e4d064736f6c63430008130033
```
共 314 个字节。   
调用 `withdraw` 函数时，花费 `28027` gas.  

**总结：**  
- 部署 WithdrawV2 比 WithdrawV1 节省了 39,750 gas，字节码少了 185 字节。
- 调用 withdraw 函数时，WithdrawV2 比 WithdrawV1 节省了 273 gas。

通过以上比较，我们可以看出使用 Yul 编写的 WithdrawV2 合约在部署和执行时更为高效，这在合约需要频繁执行转账操作时尤其有利。