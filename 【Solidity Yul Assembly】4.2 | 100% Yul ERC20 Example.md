# 【Solidity Yul Assembly】4.2 | 100% Yul ERC20 Example
在本节中，我们将详细官方文档中的 100% Yul 实现的 [ERC20 合约](https://docs.soliditylang.org/en/latest/yul.html#complete-erc20-example)。为了方便理解，我们会逐步讲解，并在适当的位置附上代码。  

首先，来看合约的构造函数部分：  
```
code {
    // Store the creator in slot zero.
    sstore(0, caller())

    // Deploy the contract
    datacopy(0, dataoffset("runtime"), datasize("runtime"))
    return(0, datasize("runtime"))
}
```
这里的构造函数与上节中的例子实现有所不同，主要是将合约创建者的地址存储在存储槽 0 中。这是为了记录合约的 owner。  

接下来，在 `switch` 语句中实现了函数选择器的功能，它负责 ERC20 的各项操作逻辑：
```
// Protection against sending Ether
require(iszero(callvalue()))

// Dispatcher
switch selector()
case 0x70a08231 /* "balanceOf(address)" */ {
    returnUint(balanceOf(decodeAsAddress(0)))
}
case 0x18160ddd /* "totalSupply()" */ {
    returnUint(totalSupply())
}
case 0xa9059cbb /* "transfer(address,uint256)" */ {
    transfer(decodeAsAddress(0), decodeAsUint(1))
    returnTrue()
}
case 0x23b872dd /* "transferFrom(address,address,uint256)" */ {
    transferFrom(decodeAsAddress(0), decodeAsAddress(1), decodeAsUint(2))
    returnTrue()
}
case 0x095ea7b3 /* "approve(address,uint256)" */ {
    approve(decodeAsAddress(0), decodeAsUint(1))
    returnTrue()
}
case 0xdd62ed3e /* "allowance(address,address)" */ {
    returnUint(allowance(decodeAsAddress(0), decodeAsAddress(1)))
}
case 0x40c10f19 /* "mint(address,uint256)" */ {
    mint(decodeAsAddress(0), decodeAsUint(1))
    returnTrue()
}
default {
    revert(0, 0)
}

...
function selector() -> s {
    s := div(calldataload(0), 0x100000000000000000000000000000000000000000000000000000000)
}
...
```
以上代码展示了 ERC20 合约的各项功能，包括 `balanceOf`、`totalSupply`、`transfer`、`transferFrom` 等常见的 ERC20 操作。这些函数都是 `non-payable` 的，这意味着它们不能接受主网币。因此，在调用这些函数之前，合约会检查交易是否附带了主网币。如果附带了主网币，交易将被拒绝。顺带一提，在 Solidity 中直接使用 `payable` 关键字修饰函数可以节省 gas 费用。  

`selector` 函数用于从 calldata 中提取函数选择器。它通过 `calldataload(0)` 从 calldata 的起始位置（偏移量 0）读取 32 字节的数据。接着，将读取的数据右移 224 位，只保留最左侧的 4 字节（即函数选择器部分）。

再来看 runtime 对象中的代码：  
```
object "runtime" {
    code {
        // Protection against sending Ether
        require(iszero(callvalue()))
        ...
        function require(condition) {
            if iszero(condition) { revert(0, 0) }
        }
    }
```
这里实现了一个简单的 `require` 函数。如果传入的 `condition` 为 0，合约将调用 revert 来回滚交易。require 并不是 Yul 的内置函数，而是此处自定义的。

接下来，我们来详细看各个 ERC20 函数的实现。
## 1. `totalSupply()`
```
...
case 0x18160ddd /* "totalSupply()" */ {
    returnUint(totalSupply())
}
...
function returnUint(v) {
    mstore(0, v)
    return(0, 0x20)
}

function totalSupply() -> supply {
    supply := sload(totalSupplyPos())
}

function totalSupplyPos() -> p { p := 1 }

```
`totalSupply()` 的返回值通过 `returnUint(v)` 函数返回。  
`returnUint(v)` 是把值 `v` 存储在内存槽 0 中，之后 `return` 内存槽 0 中的数据。回顾之前说的，`return` 是结束当前函数的调用并归还控制权和结果，和 `leave` 是不同的。  
`totalSupply()` 函数从存储槽 `totalSupplyPos()` 读取数据，`totalSupplyPos()` 始终返回槽位 1。因此，总供应量存储在存储槽 1 中。由于 Yul 中不支持直接使用变量名来表示存储位置，所以通过定义函数 `totalSupplyPos()` 来返回指定的槽位，
顺便看下，`returnTrue()` 函数，就是把 1 放在内存中再返回，于是总为真。
```
function returnTrue() {
    returnUint(1)
}
```

## 2. `mint(address,uint256)`
```
...
case 0x40c10f19 /* "mint(address,uint256)" */ {
    mint(decodeAsAddress(0), decodeAsUint(1))
    returnTrue()
}
...

function mint(account, amount) {
    require(calledByOwner())

    mintTokens(amount)
    addToBalance(account, amount)
    emitTransfer(0, account, amount)
}

```
**1. 检查调用者是否为合约所有者**  
首先，mint 函数通过 `calledByOwner()` 检查当前调用者是否是合约的所有者：  
```
function require(condition) {
    if iszero(condition) { revert(0, 0) }
}

function calledByOwner() -> cbo {
    cbo := eq(owner(), caller())
}

function owner() -> o {
    o := sload(ownerPos())
}

function ownerPos() -> p { p := 0 }
```
`owner()` 函数从存储槽 0 中读取合约所有者的地址，而 `calledByOwner()` 会将读取到的地址与当前调用者 `caller()` 进行对比，确保只有所有者可以调用 `mint` 函数。

**2. 增加总供应量**  
通过检查后，mintTokens 函数会增加代币的总供应量：  
```
function mintTokens(amount) {
    sstore(totalSupplyPos(), safeAdd(totalSupply(), amount))
}

function safeAdd(a, b) -> r {
    r := add(a, b)
    if or(lt(r, a), lt(r, b)) { revert(0, 0) }
}
```
`mintTokens` 函数首先读取当前的总供应量，将其与铸造的数量相加，然后将结果存储到存储槽 `totalSupplyPos()` 中。同时，`safeAdd` 函数通过检查加法结果是否溢出，确保加法操作的安全性。

**3. 增加账户余额**  
接下来，addToBalance 函数会将铸造的代币分配到指定账户：  
```
function addToBalance(account, amount) {
    let offset := accountToStorageOffset(account)
    sstore(offset, safeAdd(sload(offset), amount))
}

function accountToStorageOffset(account) -> offset {
    offset := add(0x1000, account)
}
```
此处，由于我们不需要完全遵从 Solidity，所以不是按照 `keccak256(key, slot)` 对应的槽来存放用户的余额，而是直接 `0x1000 + address` 来存放不同用户所对应的余额。加 0x1000 的目的是防止和之前的槽冲突。

**4. 触发 Transfer 事件**  
最后，`emitTransfer` 函数触发转账事件，将新铸造的代币转移到目标账户：
```
function emitTransfer(from, to, amount) {
    let signatureHash := 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
    emitEvent(signatureHash, from, to, amount)
}

function emitEvent(signatureHash, indexed1, indexed2, nonIndexed) {
    mstore(0, nonIndexed)
    log3(0, 0x20, signatureHash, indexed1, indexed2)
}
```
该函数使用 `log3` 记录三个 `topics`：事件签名、`from` 地址和 `to` 地址，而 `amount` 则存储在日志的 `data` 部分。  

## 3. `balanceOf(address)`
```
... 
case 0x70a08231 /* "balanceOf(address)" */ {
    returnUint(balanceOf(decodeAsAddress(0)))
}
...
function balanceOf(account) -> bal {
    bal := sload(accountToStorageOffset(account))
}

function decodeAsAddress(offset) -> v {
    v := decodeAsUint(offset)
    if iszero(iszero(and(v, not(0xffffffffffffffffffffffffffffffffffffffff)))) {
        revert(0, 0)
    }
}

function decodeAsUint(offset) -> v {
    let pos := add(4, mul(offset, 0x20))
    if lt(calldatasize(), add(pos, 0x20)) {
        revert(0, 0)
    }
    v := calldataload(pos)
}


```
- `decodeAsUint` 解码获取 uint。跳过 calldata 中的前 4 个字节，从你指定的参数索引获取 32 字节，并且会检查输入的 calldata 是否有足够长度。  
- `decodeAsAddress` 解码获取地址。检查获取的地址，正常情况下，读出的 `v` 是个地址，所以前 12 个字节是 0 。`not(0xffffffffffffffffffffffffffffffffffffffff)` 生成一个掩码，该掩码的前 12 个字节全为 1，后 20 个字节全为 0。`and(v, not(0xffffffffffffffffffffffffffffffffffffffff))` 操作将 v 的前 12 个字节与 1 做与操作，若前 12 个字节不全为 0，则函数会 revert，意味着传入的地址无效。 

## 4. `transfer(address,uint256)`
```
...
case 0xa9059cbb /* "transfer(address,uint256)" */ {
    transfer(decodeAsAddress(0), decodeAsUint(1))
    returnTrue()
}
...

function transfer(to, amount) {
    executeTransfer(caller(), to, amount)
}

function executeTransfer(from, to, amount) {
    revertIfZeroAddress(to)
    deductFromBalance(from, amount)
    addToBalance(to, amount)
    emitTransfer(from, to, amount)
}

function revertIfZeroAddress(addr) {
    require(addr)
}

function require(condition) {
    if iszero(condition) { revert(0, 0) }
}

function deductFromBalance(account, amount) {
    let offset := accountToStorageOffset(account)
    let bal := sload(offset)
    require(lte(amount, bal))
    sstore(offset, sub(bal, amount))
}

function lte(a, b) -> r {
    r := iszero(gt(a, b))
}

function addToBalance(account, amount) {
    let offset := accountToStorageOffset(account)
    sstore(offset, safeAdd(sload(offset), amount))
}

```
我们在之前解释过 `addToBalance` 和 `emitTransfer`，所以我们来看其他的。  
- `revertIfZeroAddress` 是如果 to 地址是 0 地址，则 require 的 condition 为 0 ，则 revert 。  
- `deductFromBalance` 从存用户余额的槽中获取余额，并且判断转出的金额是否小于或等于余额。`lte` 是自定义函数，其中 `gt(a, b)` 是 Yul 的内置函数，用于检查 a 是否大于 b。如果 a 大于 b，则 gt(a, b) 返回 1，否则返回 0。最后把扣除后的余额存储回存放余额的槽。这里使用的是常规的减法，是因为已经判断了转出的金额。

## 5. `allowance(address,address)`
```
...
case 0xdd62ed3e /* "allowance(address,address)" */ {
    returnUint(allowance(decodeAsAddress(0), decodeAsAddress(1)))
}
... 

function allowance(account, spender) -> amount {
    amount := sload(allowanceStorageOffset(account, spender))
}

function allowanceStorageOffset(account, spender) -> offset {
    offset := accountToStorageOffset(account)
    mstore(0, offset)
    mstore(0x20, spender)
    offset := keccak256(0, 0x40)
}

```
`allowanceStorageOffset` 函数负责计算存储 `allowance` 的存储槽位置。计算方式也和 Solidity 的不同，此处为 keccak256("存储余额的槽", spender 地址)。

## 6. `approve(address,uint256)`
```
...
case 0x095ea7b3 /* "approve(address,uint256)" */ {
    approve(decodeAsAddress(0), decodeAsUint(1))
    returnTrue()
}
...

function approve(spender, amount) {
    revertIfZeroAddress(spender)
    setAllowance(caller(), spender, amount)
    emitApproval(caller(), spender, amount)
}

function setAllowance(account, spender, amount) {
    sstore(allowanceStorageOffset(account, spender), amount)
}

function emitApproval(from, spender, amount) {
    let signatureHash := 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925
    emitEvent(signatureHash, from, spender, amount)
}

```
`setAllowance` 函数将账户 `account` 授予 `spender` 的 `amount` 存储在 `allowance` 的存储槽中。  

## 7. `transferFrom(address,address,uint256)`
```
...
case 0x23b872dd /* "transferFrom(address,address,uint256)" */ {
    transferFrom(decodeAsAddress(0), decodeAsAddress(1), decodeAsUint(2))
    returnTrue()
}
...

function transferFrom(from, to, amount) {
    decreaseAllowanceBy(from, caller(), amount)
    executeTransfer(from, to, amount)
}

function decreaseAllowanceBy(account, spender, amount) {
    let offset := allowanceStorageOffset(account, spender)
    let currentAllowance := sload(offset)
    require(lte(amount, currentAllowance))
    sstore(offset, sub(currentAllowance, amount))
}

```
`decreaseAllowanceBy` 函数减少授权额度，首先从存储槽中读取当前 `allowance`，检查要转移的 `amount` 是否小于或等于当前 `allowance`，然后更新存储扣除后的值。


**总结：**  
以上就是整个 100% Yul ERC20 的分析了，可以看出还是比较简单的，因为用到了很多有着直观命名的小函数。并且在整个合约中，没有用到空闲自由自由指针，是因为我们没有用到数组和结构体等，但是在 Solidity 中将会不断维护这个指针，带来额外的开销。