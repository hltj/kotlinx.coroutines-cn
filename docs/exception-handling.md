<!--- TEST_NAME ExceptionsGuideTest -->

**目录**

<!--- TOC -->

* [异常处理](#异常处理)
  * [异常的传播](#异常的传播)
  * [CoroutineExceptionHandler](#coroutineexceptionhandler)
  * [取消与异常](#取消与异常)
  * [异常聚合](#异常聚合)
  * [监督](#监督)
    * [监督作业](#监督作业)
    * [监督作用域](#监督作用域)
    * [监督协程中的异常](#监督协程中的异常)

<!--- END -->

## 异常处理

本节内容涵盖了异常处理与在异常上取消。
我们已经知道被取消的协程会在挂起点抛出 [CancellationException]
并且它会被协程的机制所忽略。在这里我们会看看在取消过程中抛出异常或同一个协程的<!--
-->多个子协程抛出异常时会发生什么。

### 异常的传播

协程构建器有两种形式：自动传播异常（[launch] 与 [actor]）或<!--
-->向用户暴露异常（[async] 与 [produce]）。
当这些构建器用于创建一个*根*协程时，即该协程不是另一个协程的*子*协程，
前者这类构建器将异常视为**未捕获**异常，类似 Java 的 `Thread.uncaughtExceptionHandler`，
而后者则依赖用户来最终消费<!--
-->异常，例如通过 [await][Deferred.await] 或 [receive][ReceiveChannel.receive]<!--
-->（[produce] 与 [receive][ReceiveChannel.receive] 的相关内容包含于[通道](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/channels.md)章节）。

可以通过一个使用 [GlobalScope] 创建根协程的简单示例来进行演示：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = GlobalScope.launch { // launch 根协程
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // 我们将在控制台打印 Thread.defaultUncaughtExceptionHandler
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async { // async 根协程
        println("Throwing exception from async")
        throw ArithmeticException() // 没有打印任何东西，依赖用户去调用等待
    }
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-01.kt)获取完整代码。

这段代码的输出如下（[调试](https://github.com/hltj/kotlinx.coroutines-cn/blob/master/docs/coroutine-context-and-dispatchers.md#调试协程与线程)）：

```text
Throwing exception from launch
Exception in thread "DefaultDispatcher-worker-2 @coroutine#2" java.lang.IndexOutOfBoundsException
Joined failed job
Throwing exception from async
Caught ArithmeticException
```

<!--- TEST EXCEPTION-->

### CoroutineExceptionHandler

将**未捕获**异常打印到控制台的默认行为是可自定义的。
*根*协程中的 [CoroutineExceptionHandler] 上下文元素可以被用于这个根协程<!--
-->通用的 `catch` 块，及其所有可能自定义了异常处理的子协程。
它类似于 [`Thread.uncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) 。
你无法从 `CoroutineExceptionHandler` 的异常中恢复。当调用处理者的时候，协程已经完成<!--
-->并带有相应的异常。通常，该处理者用于<!--
-->记录异常，显示某种错误消息，终止和（或）重新启动应用程序。

在 JVM 中可以重定义一个全局的异常处理者来将所有的协程通过
[`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) 注册到 [CoroutineExceptionHandler]。
全局异常处理者就如同
[`Thread.defaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) 
一样，在没有更多的指定的异常处理者被注册的时候被使用。
在 Android 中，`uncaughtExceptionPreHandler` 被设置在全局协程异常处理者中。

`CoroutineExceptionHandler` 仅在**未捕获**的异常上调用 &mdash; 没有以其他任何方式处理的异常。
特别是，所有*子*协程（在另一个 [Job] 上下文中创建的协程）委托<!--
它们的父协程处理它们的异常，然后它们也委托给其父协程，以此类推直到根协程，
因此永远不会使用在其上下文中设置的 `CoroutineExceptionHandler`。
除此之外，[async] 构建器始终会捕获所有异常并将其表示在结果 [Deferred] 对象中，
因此它的 `CoroutineExceptionHandler` 也无效。

> 在监督作用域内运行的协程不会将异常传播到其父协程，并且<!--
-->会从此规则中排除。本文档的另一个小节——[监督](#监督)提供了更多细节。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    val job = GlobalScope.launch(handler) { // 根协程，运行在 GlobalScope 中
        throw AssertionError()
    }
    val deferred = GlobalScope.async(handler) { // 同样是根协程，但使用 async 代替了 launch
        throw ArithmeticException() // 没有打印任何东西，依赖用户去调用 deferred.await()
    }
    joinAll(job, deferred)
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-02.kt)获取完整代码。

这段代码的输出如下：

```text
CoroutineExceptionHandler got java.lang.AssertionError
```

<!--- TEST-->

### 取消与异常

取消与异常紧密相关。协程内部使用 `CancellationException` 来进行取消，这个<!--
-->异常会被所有的处理者忽略，所以那些可以被 `catch` 代码块捕获的异常<!--
-->仅仅应该被用来作为额外调试信息的资源。
当一个协程使用 [Job.cancel] 取消的时候，它会被终止，但是它不会取消它的父协程。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        val child = launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                println("Child is cancelled")
            }
        }
        yield()
        println("Cancelling child")
        child.cancel()
        child.join()
        yield()
        println("Parent is not cancelled")
    }
    job.join()
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-03.kt)获取完整代码。

这段代码的输出如下：

```text
Cancelling child
Child is cancelled
Parent is not cancelled
```

<!--- TEST-->

如果一个协程遇到了 `CancellationException` 以外的异常，它将使用该异常取消它的父协程。
这个行为无法被覆盖，并且用于为<!--
-->[结构化的并发（structured concurrency）](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/composing-suspending-functions.md#structured-concurrency-with-async) 提供稳定的协程层级结构。
[CoroutineExceptionHandler] 的实现并不是用于子协程。

> 在本例中，[CoroutineExceptionHandler] 总是被设置在由 [GlobalScope]
启动的协程中。将异常处理者设置在
[runBlocking] 主作用域内启动的协程中是没有意义的，尽管子协程已经设置了异常处理者，
但是主协程也总是会被取消的。

当父协程的所有子协程都结束后，原始的异常才会被父协程处理，
见下面这个例子。

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    val job = GlobalScope.launch(handler) {
        launch { // 第一个子协程
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        launch { // 第二个子协程
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    job.join()
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-04.kt)获取完整代码。

这段代码的输出如下：

```text
Second child throws an exception
Children are cancelled, but exception is not handled until all children terminate
The first child finished its non cancellable block
CoroutineExceptionHandler got java.lang.ArithmeticException
```
<!--- TEST-->

### 异常聚合

当协程的多个子协程因异常而失败时，
一般规则是“取第一个异常”，因此将处理第一个异常。
在第一个异常之后发生的所有其他异常都作为被抑制的异常绑定至第一个异常。

<!--- INCLUDE
import kotlinx.coroutines.exceptions.*
-->

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE) // 当另一个同级的协程因 IOException  失败时，它将被取消
            } finally {
                throw ArithmeticException() // 第二个异常
            }
        }
        launch {
            delay(100)
            throw IOException() // 首个异常
        }
        delay(Long.MAX_VALUE)
    }
    job.join()  
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-05.kt)获取完整代码。

> 注意：上面的代码将只在 JDK7 以上支持 `suppressed` 异常的环境中才能正确工作。

这段代码的输出如下：

```text
CoroutineExceptionHandler got java.io.IOException with suppressed [java.lang.ArithmeticException]
```

<!--- TEST-->

> 注意，这个机制当前只能在 Java 1.7 以上的版本中使用。
在 JS 和原生环境下暂时会受到限制，但将来会取消。

取消异常是透明的，默认情况下是未包装的：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception")
    }
    val job = GlobalScope.launch(handler) {
        val inner = launch { // 该栈内的协程都将被取消
            launch {
                launch {
                    throw IOException() // 原始异常
                }
            }
        }
        try {
            inner.join()
        } catch (e: CancellationException) {
            println("Rethrowing CancellationException with original cause")
            throw e // 取消异常被重新抛出，但原始 IOException 得到了处理
        }
    }
    job.join()
//sampleEnd    
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-exceptions-06.kt)获取完整代码。

这段代码的输出如下：

```text
Rethrowing CancellationException with original cause
CoroutineExceptionHandler got java.io.IOException
```
<!--- TEST-->

### 监督

正如我们之前研究的那样，取消是在协程的整个层次结构中传播的双<!--
-->向关系。让我们看一下需要单向取消的情况。

此类需求的一个良好示例是在其作用域内定义作业的 UI 组件。如果任何一个 UI 的子作业<!--
-->执行失败了，它并不总是有必要取消（有效地杀死）整个 UI 组件，
但是如果 UI 组件被销毁了（并且它的作业也被取消了），由于它的结果不再被需要了，它有必要使所有的子作业执行失败。

另一个例子是服务进程孵化了一些子作业并且需要 _监督_
它们的执行，追踪它们的故障并在这些子作业执行失败的时候重启。

#### 监督作业

[SupervisorJob][SupervisorJob()] 可以用于这些目的。
它类似于常规的 [Job][Job()]，唯一的不同是：<!--
-->SupervisorJob 的取消只会向下传播。这是很容易用以下示例演示：

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // 启动第一个子作业——这个示例将会忽略它的异常（不要在实践中这么做！）
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("The first child is failing")
            throw AssertionError("The first child is cancelled")
        }
        // 启动第二个子作业
        val secondChild = launch {
            firstChild.join()
            // 取消了第一个子作业且没有传播给第二个子作业
            println("The first child is cancelled: ${firstChild.isCancelled}, but the second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // 但是取消了监督的传播
                println("The second child is cancelled because the supervisor was cancelled")
            }
        }
        // 等待直到第一个子作业失败且执行完成
        firstChild.join()
        println("Cancelling the supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-01.kt)获取完整代码。

这段代码的输出如下：

```text
The first child is failing
The first child is cancelled: true, but the second one is still active
Cancelling the supervisor
The second child is cancelled because the supervisor was cancelled
```
<!--- TEST-->


#### 监督作用域

对于*作用域*的并发，可以用 [supervisorScope][_supervisorScope] 来替代 [coroutineScope][_coroutineScope] 来实现相同的目的。它只会单向的传播<!--
-->并且当作业自身执行失败的时候将所有子作业全部取消。作业自身也会在所有的子作业执行结束前等待，
就像 [coroutineScope][_coroutineScope] 所做的那样。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("The child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("The child is cancelled")
                }
            }
            // 使用 yield 来给我们的子作业一个机会来执行打印
            yield()
            println("Throwing an exception from the scope")
            throw AssertionError()
        }
    } catch(e: AssertionError) {
        println("Caught an assertion error")
    }
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-02.kt)获取完整代码。

这段代码的输出如下：

```text
The child is sleeping
Throwing an exception from the scope
The child is cancelled
Caught an assertion error
```
<!--- TEST-->

#### 监督协程中的异常

常规的作业和监督作业之间的另一个重要区别是异常处理。
监督协程中的每一个子作业应该通过异常处理机制处理自身的异常。
这种差异来自于子作业的执行失败不会传播给它的父作业的事实。
这意味着在 [supervisorScope][_supervisorScope] 内部直接启动的协程*确实*使用了<!--
-->设置在它们作用域内的 [CoroutineExceptionHandler]，与父协程的方式相同
（参见 [CoroutineExceptionHandler](#coroutineexceptionhandler) 小节以获知更多细节）。

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("CoroutineExceptionHandler got $exception") 
    }
    supervisorScope {
        val child = launch(handler) {
            println("The child throws an exception")
            throw AssertionError()
        }
        println("The scope is completing")
    }
    println("The scope is completed")
}
```

</div>

> 可以在[这里](../kotlinx-coroutines-core/jvm/test/guide/example-supervision-03.kt)获取完整代码。

这段代码的输出如下：

```text
The scope is completing
The child throws an exception
CoroutineExceptionHandler got java.lang.AssertionError
The scope is completed
```
<!--- TEST-->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[CoroutineExceptionHandler]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Deferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[SupervisorJob()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html
[Job()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html
[_coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[_supervisorScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
<!--- END -->
