## Solidity Yul Assembly 学习笔记
**刨根问底探链上真相，品味坎坷悟Web3人生。**

这是我之前学习 Yul 的笔记，看的是[Jeffrey Scholz](https://www.udemy.com/user/jeffrey-scholz/)的课程,未来一段时间我将重新整理更新发出来。  
  
在这系列文章中，我们将深入探讨 Solidity 的内联汇编（Yul）。你可能会问：“我学会 Solidity 不就能写大部分合约了吗？为什么还需要学习内联汇编？”的确，大部分合约的编写完全可以通过 Solidity 完成。但内联汇编是 Solidity 的一个重要补充，它让你更深入地理解底层操作和合约优化。

起初，我也曾对内联汇编感到困惑，尽管我曾尝试过，但很快就忘记了。中文资料少且零散，这使得学习内联汇编变得更加困难。后来，找到了 Jeffrey Scholz 较为系统的讲解 Yul 的课程，此系列文章为我当时的学习笔记整理而来。学习 Yul 让我对存储、内存、栈、合约调用以及 ABI 编码有了更深入的理解。

即使你未来可能不会直接编写内联汇编代码，但掌握这些知识对编写更高效的 Solidity 合约是非常有帮助的。希望这系列文章能帮助你更好地理解内联汇编的基础及其在合约中的应用。
  
- 推特：
[![X (formerly Twitter) Follow](https://img.shields.io/twitter/follow/crypto_yi)](https://twitter.com/crypto_yi)（欢迎关注）
  
收藏请点点 Star! 如果发现我有写错误的，欢迎随时帮我改正，谢谢！  


