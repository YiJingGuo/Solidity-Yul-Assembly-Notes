# 【Solidity Yul Assembly】1.1 | Types

**在 Yul 中没有多种类型的概念，或者说只有一种类型——u256，也可以理解为 bytes32。**  

我们直接通过例子来学习。  
首先，让我们看看如何在 Yul 中实现一个简单的函数：  
``` solidity
function getNumber() external pure returns (uint256) {
    return 42;
}
```
在 Yul 中，这段代码可以这样写：  
``` solidity
function getNumber() external pure returns (uint256) {
    uint256 x;

    assembly {
        x := 42
    }

    return x;
}
```
从上面的代码可以看出，Yul 代码是没有分号结尾的。  

接下来我们来看一个与字符串相关的例子：  
``` solidity
function demoString() external pure returns (string memory){
    string memory myString = "";

    assembly {
        myString := "hello world"
    }

    return myString;
}
```
这段代码是错误的，因为 `string memory myString` 是存储在内存中的，而 Yul 中的赋值语句 `myString := "hello world"` 是把一个字面量赋值给一个栈上的指针，这显然是不对的。    
  
来看正确的写法：  
``` solidity
function demoString() external pure returns (bytes32){
    bytes32 myString = "";

    assembly {
        myString := "hello world"
    }

    return myString;
}
```
这个函数返回的是 "hello world" 对应的 bytes32。稍后我们可以在 Solidity 中将其转化为字符串类型。值得注意的是，bytes32 总是存储在栈上。    

如果我们想把这个 bytes32 转化为字符串，可以这样写：  
``` solidity
function demoString() external pure returns (string memory){
    bytes32 myString = "";

    assembly {
        myString := "hello world"
    }

    return string(abi.encode(myString));
}
```
需要注意的是，如果字面量超过了 32 字节，编译器会报错。后续我们会探讨如何处理这种情况。通过这些例子，我们可以看到，存储在栈中的数据其实都是 bytes32，Solidity 会根据不同的类型解释这些数据。  
再来看几个例子：  
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

**总结：**
通过这些例子，我们看到了 Yul 中的类型处理方式。Yul 中的所有数据都是以 bytes32 的形式存储在栈上，Solidity 则根据不同的上下文来解释这些数据。