[//]: # (title: 协程指南)

Kotlin 是一门仅在标准库中提供最基本底层 API 以便各种其他<!--
-->库能够利用协程的语言。与许多其他具有类似功能的语言不同，`async` 与 `await`
在 Kotlin 中并不是关键字，甚至都不是标准库的一部分。此外，Kotlin 的
_挂起函数_ 概念为异步操作提供了比
future 与 promise 更安全、更不易出错的抽象。

`kotlinx.coroutines` 是由 JetBrains 开发的功能丰富的协程库。它包含<!--
-->本指南中涵盖的很多启用高级协程的原语，包括 `launch`、 `async` 等等。

本文是关于 `kotlinx.coroutines` 核心特性的指南，包含一系列示例，并分为不同的主题。

为了使用协程以及按照本指南中的示例演练，需要添加对 `kotlinx-coroutines-core` 模块的依赖，如<!--
-->[项目中的 README 文件](../README.md#using-in-your-projects)所述。

## 目录

* [基础](basics.md)
* [取消与超时](cancellation-and-timeouts.md)
* [组合挂起函数](composing-suspending-functions.md)
* [协程上下文与调度器](coroutine-context-and-dispatchers.md)
* [异步流](flow.md)
* [通道](channels.md)
* [异常处理与监督](exception-handling.md)
* [共享的可变状态与并发](shared-mutable-state-and-concurrency.md)
* [Select 表达式（实验性的）](select-expression.md)

## 其他参考资料

* [使用协程进行 UI 编程指南](../ui/coroutines-guide-ui.md)
* [协程设计文档（KEEP）](https://github.com/Kotlin-zh/KEEP/blob/master/proposals/coroutines.md)
* [完整的 kotlinx.coroutines API 参考文档](https://kotlin.github.io/kotlinx.coroutines)
