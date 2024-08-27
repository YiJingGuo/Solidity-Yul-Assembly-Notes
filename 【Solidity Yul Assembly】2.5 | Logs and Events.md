# 【Solidity Yul Assembly】2.5 | Logs and Events

## log 指令
以下是[Yul 文档](https://docs.soliditylang.org/en/latest/yul.html#evm-dialect)中摘出来的部分。  
|            指令            |                      解释                        | 
| -------------------------- | ------------------------------------------------ |
| log0(p, s)                 | log data mem[p…(p+s))                            |
| log1(p, s, t1)             | log data mem[p…(p+s)) with topic t1              |
| log2(p, s, t1, t2)         | log data mem[p…(p+s)) with topics t1, t2         |
| log3(p, s, t1, t2, t3)     | log data mem[p…(p+s)) with topics t1, t2, t3     |
| log4(p, s, t1, t2, t3, t4) | log data mem[p…(p+s)) with topics t1, t2, t3, t4 |  

总体来说，事件的哈希值通常作为第一个 `topic (topics[0])`，而所有用 `indexed` 修饰的字段会分别作为其他 `topics（如 topics[1], topics[2], 等）`存储在日志中。这些 `topics` 用于有效地过滤和检索特定事件。而未被 `indexed` 修饰的字段的数据则会被存储在日志的 `data` 部分，这些数据在内存中指定的范围内取出。  
这么说可能有些看不懂，看以下两个例子就明白了。

## 完全使用 indexed 修饰的事件
``` solidity
event SomeLog(uint256 indexed a, uint256 indexed b);

function emitLog() external {
    emit SomeLog(5, 6);
}

function yulEmitLog() external {
    assembly {
        // keccak256("SomeLog(uint256,uint256)")
        let signature := 0xc200138117cf199dd335a2c6079a6e1be01e6592b6a76d4b5fc31b169df819cc
        log3(0, 0, signature, 5, 6)
    }
}
```
SomeLog 事件中的两个字段均被 `indexed` 修饰，因此日志记录指令 `log3` 中内存读取范围为 0 到 0，`t1` 位置存放的是事件的哈希值。将该合约部署到测试网后，查看日志信息如下：  
![](/img/yul-2.5/1.png)

## 包含未使用 indexed 修饰的字段的事件
``` solidity
event SomeLogV2(uint256 indexed a, bool);

function v2EmitLog() external {
    emit SomeLogV2(5, true);
}

function v2YulEmitLog() external {
    assembly {
        // keccak256("SomeLogV2(uint256,bool)")
        let signature := 0x113cea0e4d6903d772af04edb841b17a164bff0f0d88609aedd1c4ac9b0c15c2
        mstore(0x00, 1)
        log2(0, 0x20, signature, 5)
    }
}
```
在 `SomeLogV2` 事件中，`bool` 类型的字段没有被 `indexed` 修饰，因此需要先将其存入内存中，此处存放在了内存槽 `0x00` 的位置。这里多解释一下，有些读者可能会问，在 Solidity 中不是内存从 0x80 开始使用的吗，但注意，这里函数没有使用到 Solidity 部分的代码，所以不需要遵从 Solidity 的默认内存管理方式。  
然后事件中 `bool` 类型的字段通过日志指令记录到 `data` 字段中。  

![](/img/yul-2.5/2.png)

**总结：**  
通过本章的学习，我们对日志有了更深入的理解。比如，为什么打开浏览器，事件的哈希值常常是 `topics[0]`。也能理解“匿名事件”只不过使用的是 `log0(p, s)` 指令。