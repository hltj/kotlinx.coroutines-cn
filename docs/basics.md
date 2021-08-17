<!--- TEST_NAME BasicsGuideTest -->

**目录**

<!--- TOC -->

* [协程基础](#协程基础)
  * [第一个协程程序](#第一个协程程序)
  * [桥接阻塞与非阻塞的世界](#桥接阻塞与非阻塞的世界)
  * [等待一个作业](#等待一个作业)
  * [结构化的并发](#结构化的并发)
  * [作用域构建器](#作用域构建器)
  * [提取函数重构](#提取函数重构)
  * [协程很轻量](#协程很轻量)
  * [全局协程像守护线程](#全局协程像守护线程)

<!--- END -->

## 协程基础

这一部分包括基础的协程概念。

### 第一个协程程序

运行以下代码：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() {
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L) // 非阻塞的等待 1 秒钟（默认时间单位是毫秒）
        println("World!") // 在延迟后打印输出
    }
    println("Hello,") // 协程已在等待时主线程还在继续
    Thread.sleep(2000L) // 阻塞主线程 2 秒钟来保证 JVM 存活
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-01.kt)获取完整代码。

代码运行的结果：

```text
Hello,
World!
```

<!--- TEST -->

本质上，协程是轻量级的线程。
它们在某些 [CoroutineScope] 上下文中与 [launch] _协程构建器_ 一起启动。
这里我们在 [GlobalScope] 中启动了一个新的协程，这意味着新协程的<!--
-->生命周期只受整个应用程序的生命周期限制。

可以将
`GlobalScope.launch { …… }` 替换为 `thread { …… }`，并将 `delay(……)` 替换为 `Thread.sleep(……)` 达到同样目的。
试试看（不要忘记导入 `kotlin.concurrent.thread`）。

如果你首先将 `GlobalScope.launch` 替换为 `thread`，编译器会报以下错误：

```
Error: Kotlin: Suspend functions are only allowed to be called from a coroutine or another suspend function
```

这是因为 [delay] 是一个特殊的 _挂起函数_ ，它不会造成线程阻塞，但是会 _挂起_
协程，并且只能在协程中使用。

### 桥接阻塞与非阻塞的世界

第一个示例在同一段代码中混用了 _非阻塞的_ `delay(……)` 与 _阻塞的_ `Thread.sleep(……)`。
这容易让我们记混哪个是阻塞的、哪个是非阻塞的。
让我们显式使用 [runBlocking] 协程构建器来阻塞：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() {
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 主线程中的代码会立即执行
    runBlocking {     // 但是这个表达式阻塞了主线程
        delay(2000L)  // ……我们延迟 2 秒来保证 JVM 的存活
    } 
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-02.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

结果是相似的，但是这些代码只使用了非阻塞的函数 [delay]。
调用了 `runBlocking` 的主线程会一直 _阻塞_ 直到 `runBlocking` 内部的协程执行完毕。

这个示例可以使用更合乎惯用法的方式重写，使用 `runBlocking` 来包装
main 函数的执行：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> { // 开始执行主协程
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L)
        println("World!")
    }
    println("Hello,") // 主协程在这里会立即执行
    delay(2000L)      // 延迟 2 秒来保证 JVM 存活
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-03.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

这里的 `runBlocking<Unit> { …… }` 作为用来启动顶层主协程的适配器。
我们显式指定了其返回类型 `Unit`，因为在 Kotlin 中 `main` 函数必须返回 `Unit` 类型。

这也是为挂起函数编写单元测试的一种方式：

<!--- INCLUDE
import kotlinx.coroutines.*
-->

<div class="sample" markdown="1" theme="idea" data-highlight-only>
 
```kotlin
class MyTest {
    @Test
    fun testMySuspendingFunction() = runBlocking<Unit> {
        // 这里我们可以使用任何喜欢的断言风格来使用挂起函数
    }
}
```

</div>

<!--- CLEAR -->
 
### 等待一个作业

延迟一段时间来等待另一个协程运行并不是一个好的选择。让我们显式<!--
-->（以非阻塞方式）等待所启动的后台 [Job] 执行结束：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = GlobalScope.launch { // 启动一个新协程并保持对这个作业的引用
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join() // 等待直到子协程执行结束
//sampleEnd
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-04.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

现在，结果仍然相同，但是主协程与后台作业的持续时间<!--
-->没有任何关系了。好多了。

### 结构化的并发

协程的实际使用还有一些需要改进的地方。
当我们使用 `GlobalScope.launch` 时，我们会创建一个顶层协程。虽然它很轻量，但它运行时仍会<!--
-->消耗一些内存资源。如果我们忘记保持对新启动的<!--
-->协程的引用，它还会继续运行。如果协程中的代码挂起了会怎么样（例如，我们错误地<!--
-->延迟了太长时间），如果我们启动了太多的协程并导致内存不足会怎么样？
必须手动保持对所有已启动协程的引用并 [join][Job.join] 之很容易出错。

有一个更好的解决办法。我们可以在代码中使用结构化并发。
我们可以在执行操作所在的指定作用域内启动协程，
而不是像通常使用线程（线程总是全局的）那样在 [GlobalScope] 中启动。

在我们的示例中，我们使用 [runBlocking] 协程构建器将  `main` 函数转换为协程。
包括 `runBlocking` 在内的每个协程构建器都将 [CoroutineScope] 的实例添加到其代码块所在的作用域中。
我们可以在这个作用域中启动协程而无需显式 `join` 之，因为<!--
-->外部协程（示例中的 `runBlocking`）直到在其作用域中启动的所有协程<!--
-->都执行完毕后才会结束。因此，可以将我们的示例简化为：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // this: CoroutineScope
    launch { // 在 runBlocking 作用域中启动一个新协程
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-05.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->

### 作用域构建器

除了由不同的构建器提供协程作用域之外，还可以使用
[coroutineScope][_coroutineScope] 构建器声明自己的作用域。它会创建一个协程作用域并且在所有已启动子协程执行完毕之前不会结束。

[runBlocking] 与 [coroutineScope][_coroutineScope] 可能看起来很类似，因为它们都会等待其协程体以及所有子协程结束。
主要区别在于，[runBlocking] 方法会*阻塞*当前线程来等待，
而 [coroutineScope][_coroutineScope] 只是挂起，会释放底层线程用于其他用途。
由于存在这点差异，[runBlocking] 是常规函数，而 [coroutineScope][_coroutineScope] 是挂起函数。

可以通过以下示例来演示：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking { // this: CoroutineScope
    launch { 
        delay(200L)
        println("Task from runBlocking")
    }
    
    coroutineScope { // 创建一个协程作用域
        launch {
            delay(500L) 
            println("Task from nested launch")
        }
    
        delay(100L)
        println("Task from coroutine scope") // 这一行会在内嵌 launch 之前输出
    }
    
    println("Coroutine scope is over") // 这一行在内嵌 launch 执行完毕后才输出
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-06.kt)获取完整代码。

<!--- TEST
Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
-->

请注意，（当等待内嵌 launch 时）紧挨“Task from coroutine scope”消息之后，
就会执行并输出“Task from runBlocking”——尽管 [coroutineScope][_coroutineScope] 尚未结束。

### 提取函数重构

我们来将  `launch { …… }` 内部的代码块提取到独立的函数中。当你<!--
-->对这段代码执行“提取函数”重构时，你会得到一个带有 `suspend` 修饰符的新函数。
这是你的第一个*挂起函数*。在协程内部可以像普通函数一样使用挂起函数，
不过其额外特性是，同样<!--
-->可以使用其他挂起函数（如本例中的 `delay`）来*挂起*协程的执行。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch { doWorld() }
    println("Hello,")
}

// 这是你的第一个挂起函数
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-07.kt)获取完整代码。

<!--- TEST
Hello,
World!
-->


但是如果提取出的函数包含一个在当前作用域中调用的协程构建器的话，该怎么办？
在这种情况下，所提取函数上只有 `suspend` 修饰符是不够的。为 `CoroutineScope` 写一个 `doWorld` 扩展<!--
-->方法是其中一种解决方案，但这可能并非总是适用，因为它并没有使 API 更加清晰。
惯用的解决方案是要么显式将 `CoroutineScope` 作为包含该函数的类的一个字段，
要么当外部类实现了 `CoroutineScope` 时隐式取得。
作为最后的手段，可以使用 [CoroutineScope(coroutineContext)][CoroutineScope()]，不过这种方法结构上不安全，
因为你不能再控制该方法执行的作用域。只有私有 API 才能使用这个构建器。

### 协程很轻量

运行以下代码：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    repeat(100_000) { // 启动大量的协程
        launch {
            delay(5000L)
            print(".")
        }
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-08.kt)获取完整代码。

<!--- TEST lines.size == 1 && lines[0] == ".".repeat(100_000) -->

它启动了 10 万个协程，并且在 5 秒钟后，每个协程都输出一个点。

现在，尝试使用线程来实现。会发生什么？（很可能你的代码会产生某种内存不足的错误）

### 全局协程像守护线程

以下代码在 [GlobalScope] 中启动了一个长期运行的协程，该协程每秒输出“I'm sleeping”两次，之后<!--
-->在主函数中延迟一段时间后返回。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    GlobalScope.launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 在延迟后退出
//sampleEnd
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-basic-09.kt)获取完整代码。

你可以运行这个程序并看到它输出了以下三行后终止：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
```

<!--- TEST -->

在 [GlobalScope] 中启动的活动协程并不会使进程保活。它们就像守护线程。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[_coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html
<!--- END -->


