---
layout: post
title: "协程咖啡厅 - 初遇"
description: "探索 Kotlin 协程及在 Android 中的应用"
cover_url: /assets/covers/cafe-coroutines.webp
cover_meta: illustration by [Kayamori](https://www.pixiv.net/artworks/63116446)
tags: 
  - Develop
  - Kotlin
  - Café Coroutines
---

<p style="font-size:14px;margin-bottom:8px;">本文不是一篇协程教程，而是一篇协程介绍，其编写目的是用较易理解的方式让读者了解协程的基础知识、使用方式与优点，因此本文对描写协程的笔墨并不是很多。</p>

<p style="font-size:14px;margin-bottom:8px;">若阅读本文后读者希望对协程有更多的了解，可浏览文末的参考链接，其中包括 Kotlin 中文站（对 Kotlin 官方网站进行翻译）协程文档与其它平台的技术博文，有助于更深入地了解协程。</p>

## 前置任务

在了解协程前先看看 Kotlin 的 Lambda 语法。

Kotlin 中的函数是**头等函数（First-Class Function）**，这意味着函数可以存储在变量或数据结构中，也可以作为其它函数的参数或返回值。既然函数可作为变量存储，那么需要了解函数的类型定义，举个例子，如以下函数：

``` kotlin
fun function(valueA: Int, valueB: Boolean): String { /* .. */ }
```

对应的变量类型为：

``` kotlin
(Int, Boolean) -> String
```

可以通过声明变量存储函数。

``` kotlin
val value: (Int, Boolean) -> String = ::function
```

> 不返回任何数据的函数，其返回类型为 Unit，通常在函数声明时可以省略，如以下两个函数的声明是一致的：
> 
> ``` kotlin
> fun helloWorld(): Unit { /* .. */ }
> 
> fun helloWorld() { /* .. */ }
> ```
> 
> 但对于函数的类型定义，返回类型不可省略，上述两个函数的函数类型均为：
> 
> ``` kotlin
> () -> Unit
> ```

函数的实例化可通过 Lambda 实现，如以下函数：

``` kotlin
fun add(a: Int, b: Int): Int { return a + 1 }
```

与其实现等价的 Lambda 为：

``` kotlin
{ a: Int, b: Int -> a + 1 }
```

> Lambda 最后一行表达式的值即为 Lambda 的返回值。

在可通过前文推断类型的情况下，参数类型可以省略。

``` kotlin
{ a, b -> a + 1 }
```

若 Lambda 中不需要某参数，可以通过下划线省略。

``` kotlin
{ a, _ -> a + 1 }
```

特别的，若 Lambda 仅有一个参数，该参数可以省略而采用 it 代替，如以下两个 Lambda 等价：

``` kotlin
{ msg -> println(msg) }

{ println(it) }
```

当 Lambda 作为函数的最后一个参数时，该 Lambda 可以提到括号之后，如以下两个函数调用等价：

``` kotlin
networkRequest(arguments, { result ->
    showResult(result)    
})

networkRequest(arguments) { result ->
    showResult(result)
}
```

当 Lambda 为函数的唯一参数时，括号可以省略。

``` kotlin
button.setOnClickListener { view -> /* .. */ }
```

## 故事要从一间咖啡厅说起

你开了一间咖啡厅。

目前你的咖啡厅只有一名店员，在空闲期可以完美处理所有工作。

> 单线程应用。

但客人渐渐多了起来，仅一名店员无法应付工作，于是你决定聘用临时店员。

> 开启新的线程。

虽然临时店员解决了店内的压力，但是频繁聘用临时店员的手续实在过于复杂，聘用多名正式店员显然是更棒的主意。

> 创建可复用线程的线程池。

多名正式店员的加入让咖啡厅有能力接待更多的客人，为了让咖啡厅的工作更加高效，咖啡制作与客人接待的工作分离，每一名店员有各自的职能，并彼此协作。

> 线程调度。

现在，咖啡厅的工作有序地进行着，咖啡厅里充满了快活的空气。

## 让咖啡厅的工作更加高效

虽然目前咖啡厅的工作安排已经可以很好地减弱了接待的压力，但是存在一个影响效率的问题：**一旦店员开始了一项工作，在未完成之前无法停止**。

这种一条路走到黑的方式实在不怎么灵活：客人取消了订单，没有做完的咖啡却仍然需要完成；如果店长在打烊时未能整理完上个月的账单，店员们只能坐下细品咖啡，晚点下班。

在程序中也存在着同样的情况，我们把不可分割、在执行完毕之前无法被中断的操作称为**原子操作（Atomic Operation）**，正如咖啡厅不会让服务生进行较为漫长的料理工作一般，程序设计中我们会更倾向于将大量占用资源的操作交由后台线程处理。问题在于，如果一个耗费资源的原子操作执行，即便中途确认这个原子操作不再具有意义（比如用户希望查看一张网络图片，但图片在加载过程中用户不想继续等待，因此取消了图片的查看，那么图片加载便不需要继续执行），操作也无法被终止，以致资源的浪费。

如何让店员们工作进行一半的时候能够腾出手来，或者放弃不必要的任务？

## 漫长的等待是另一个坑

一位客人点了一杯玛奇朵，这杯玛奇朵的咖啡萃取由店员 A 进行，拉花由店员 B 进行。因为拉花必须要在咖啡萃取完之后，因此如何安排两位店员的任务就成了一个问题，最简单的方式莫过于 B 等待 A 萃取完咖啡后再进行拉花，但等待 A 的这段时间里 B 无法进行任何工作，否则等 A 萃取完咖啡时 B 早已将拉花的工作搁置在一旁了。

为了确保 B 不会忘记拉花的工作，我们可以让 A 在萃取完咖啡后顺便提醒 B，于是 “提醒 B 拉花” 这件事便被添加在 A 的任务之内。

同样地，在程序设计上，如果我们将部分代码进行包装，以在另一个任务完成后执行，可以解决阻塞（等待资源不工作的过程）问题，最常见的方式即回调。但将拉花提醒放在萃取咖啡的步骤里实在显得奇怪，而且，如果再加上到前文的原子操作问题，两位店员的任务便被绑定在一起。若客人因急事需要取消订单，不仅 A 无法提前结束萃取咖啡的操作，B 无意义的拉花工作也需要继续进行。

## 让工作可以暂停

在 Kotlin 原有的模型中，每一个函数都是一个原子操作，即每个函数被调用之后便无法停止，而协程引入了一种新的函数类型：**挂起函数（Suspending Function）**。挂起函数的声明与普通函数类似，仅需要添加一个 suspend 关键字：

``` kotlin
suspend fun suspendingFunction() { /* .. */ }

suspend fun anotherSuspendingFuction(argument: Int): String { /* .. */ }
```

相比不可中断的普通函数，挂起函数可以在执行的过程中被挂起（暂停），挂起恢复后还可切换到其它线程运行，需要注意的是，挂起函数只能在挂起函数内部调用，需要通过一些协程相关函数使其可在普通函数内调用。

> 为什么挂起函数只能在挂起函数内部调用？
>
> 前文提及，挂起函数可以在执行的过程中被挂起，而普通函数是无法做到的，因此在一个作为原子操作的函数内部调用可挂起的函数，其可挂起的特性并没有任何意义。反过来，挂起函数内却可以调用普通函数，实际上之所以挂起函数可以被挂起，是由于挂起函数内部是由多个原子操作构成的，每一个原子操作之间可以作为挂起点，当然，倘若一个挂起函数内部仅包含一个原子操作，那么这个挂起函数实际上并不具备挂起功能。

一个最简单的挂起函数在普通函数内调用的方法：使用 runBlocking() 函数。这个函数接收一个可挂起的扩展函数（该扩展函数的类型为 *suspend CoroutineScope.() -> ???*），用于将该挂起扩展函数转换为阻塞的普通函数调用，采用 Lambda 作为挂起扩展函数的匿名实现是最常见的做法。

``` kotlin
runBlocking {
    suspendingFunction()
    anotherSuspendingFunction()
}
```

runBlocking 的返回值等于挂起扩展函数的返回值。

## 梦开始的地方

挂起函数的特性为协程提供了支持。

在 Kotlin/Jvm 中，协程本质上是轻量级的线程，用于运行可挂起的代码，且可在代码执行过程中进行线程切换。

launch() 函数可启动一个新的协程，一个简单的例子如下：

``` kotlin
GlobalScope.launch {
    suspendingFunction()
}
```

在实现效果上类似于开启一个新的线程：

``` kotlin
Thread {
    function()
}.start()
```

两者的区别在于，协程是可被取消的，launch() 函数可接收一个可挂起的扩展函数（该函数的类型为 *suspend CoroutineScope.() -> Unit*），从而可以直接调用挂起函数，而开启线程方式中的 Runnable 属于普通函数，无法直接调用挂起函数。

取消一个协程也非常简单，launch() 函数返回一个 Job 类型的变量，可用于对协程进行控制，取消协程可通过调用 Job.cancel() 实现。

``` kotlin
val job = GlobalScope.launch {
    suspendingFunction()
}

job.cancel()
```

也可通过调用 Job.join() 阻塞代码，该函数调用后的代码将在对应协程执行完毕后才会继续执行。

``` kotlin
val job = GlobalScope.launch {
    runningTask()
}

job.join()
println("This message will be displayed until task is completed.")
```

launch 函数可通过指定**调度器（Dispatcher）**来确定协程将启动在哪个线程上。

``` kotlin
GlobalScope.launch(Dispatchers.IO) {
    taskRunningOnThreadIO()
}
```

调度器的概念实际并不复杂，它实现了协程上下文的接口（CoroutineContext），用于确定协程在哪个线程运行，一个调度器对应一个线程或线程池，可通过 newSingleThreadContext() 函数实现一个自定义调度器，函数将通过传入的线程名称创建一个新的线程，由于专有线程是较为昂贵的资源，当不再需要后通过 close() 函数手动关闭。

``` kotlin
val customDispatcher = newSingleThreadContext("custom thread name")

val job = GlobalScope.launch(customDispatcher) {
    runningTask()
}

job.join()
customDispatcher.close()
```

协程内也可以创建新的协程，协程内所创建的协程称为原有协程的子协程。当一个协程被取消时，其所有的子协程也会被取消。

``` kotlin
scope.launch {
    println("this message is displayed on coroutine")
    scope.launch {
        println("this message is displayed on child of coroutine")
    }
}
```

> 注意到这部分代码中，协程的启动不再是调用 GlobalScope 的 launch() 函数，是因为通过 GlobalScope.launch() 启动的协程为独立运作的，不具有父协程。
>
> 有关 GlobalScope 的概念将在后文的协程作用域部分提及。

## 准点下班

咖啡厅没有加班的传统，不错 (´▽｀)。

如果要庆祝一位店员的生日，大家决定在打烊后开场派对，但如果因为一些无关紧要的工作耽搁，那就非常扫兴了。

虽然店员们已经可以随时停下自己手头的工作了，但每逢打烊时一个一个店员通知难免有些麻烦，最好的方式莫过于定好闹钟，打烊时间一到，大家都能知晓。

在程序设计中，一些对象具备生命周期，如果可以在对象指定生命周期结束时自动取消所有未完成的协程，可以有效的避免资源浪费，也能减少如内存泄漏等问题的隐患。

**协程作用域（Coroutine Scope）**用于管理协程的生命周期，前文中采用的 GlobalScope 即为实现协程作用域的接口（CoroutineScope），而通过 launch() 函数启动的协程便隶属该作用域。下面用最简单的方法构造一个协程作用域，协程作用域可指定默认的调度器，当作用域启动的协程不指定调度器时采用：

``` kotlin
val customScope = CoroutineScope(customDispatcher)
```

采用 cancel() 函数取消作用域内未完成的协程，通常在所绑定的生命周期结束回调时调用。

``` kotlin
customScope.launch {
    runningTask()
}

customScope.cancel()
```

在 Android 中，如 lifecycleScope 等协程作用域的实现已经默认帮我们绑定了指定的生命周期对象，无需我们手动对作用域进行管理。

``` kotlin
lifecycleScope.launchWhenStarted {
    Log.v(TAG, "coroutine will be started when onStart() function is called")
    Log.v(TAG, "and will be cancelled when onStop() function is called")
    runningTask()
}
```

## 参考链接

- [Kotlin 中文站 - 异步程序设计](https://www.kotlincn.net/docs/tutorials/coroutines/async-programming.html)

- [Kotlin 中文站 - 协程指南](https://www.kotlincn.net/docs/reference/coroutines/coroutines-guide.html)