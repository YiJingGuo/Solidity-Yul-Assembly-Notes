## 【Solidity Yul Assembly】1.1 | Types

**Yul 没有类型，或者说只有一种类型 —— u256 或者说是 bytes32.  **

我们直接来看例子：    
``` solidity
function getNumber() external pure returns (uint256) {
    return 42;
}
```
在 Yul 中该怎么写呢？    
``` solidity
function getNumber() external pure returns (uint256) {
    uint256 x;

    assembly {
        x := 42
    }

    return x;
}
```
从上面代码可以看出：Yul 代码是没有分号结尾的。

以下是一段错误的代码：  
``` solidity
function demoString() external pure returns (string memory){
    string memory myString = "";

    assembly {
        myString := "hello world"
    }

    return myString;
}
```
由于 `string memory myString` 存储在内存中，并不在栈中，所以 `myString` 其实是栈上指向内存位置的指针。`myString := "hello world"`相当于把 "hello world" 赋值给一个指针，所以当然是错的。  
  
接下来看看正确的写法：  
``` solidity
function demoString() external pure returns (bytes32){
    bytes32 myString = "";

    assembly {
        myString := "hello world"
    }

    return myString;
}
```
返回值为 "hello world" 对应的 bytes32, 稍后可以将在 Solidity 中做转化。bytes32 总是存储在栈上。  

做转化的写法：  
``` solidity
function demoString() external pure returns (string memory){
    bytes32 myString = "";

    assembly {
        myString := "hello world"
    }

    return string(abi.encode(myString));
}
```
  
如果字面量超过了 32 字节，编译器则会报错，之后我们将解释怎么处理。  
可以看出，存储在栈中的都是 bytes32 只是 Solidity 会根据不同的类型做出不同的解释。  

再看看几个例子。  
``` solidity
function representation() external pure returns(bool) {
    bool x;

    assembly {
        x := 1
    }
    return x;  // return true
}
```
``` solidity
function representation() external pure returns(address) {
    address x;

    assembly {
        x := 1
    }
    return x;  // return 0x0000000000000000000000000000000000000001
}
```

### 总结：
1. Yul 统一类型: 在 Yul 中，所有数据类型都统一为 bytes32，无论在 Solidity 中是 uint256、string memory 还是 address，它们在 Yul 中的表示形式都是 bytes32。  
2. 无类型操作: 由于 Yul 中没有显式的类型概念，因此在操作数据时，开发者需要了解不同类型在内存和栈中的表示方式。  
3. 字符串处理: 直接将字符串赋值给栈中的变量会导致错误，需要先将字符串转换为 bytes32，并在必要时在 Solidity 中进行类型转换。  

