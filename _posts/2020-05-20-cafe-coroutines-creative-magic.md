---
layout: post
title: "协程咖啡厅 - 构造魔法"
description: "探索 Kotlin 协程实现原理"
cover_url: /assets/covers/cafe-coroutines.webp
cover_meta: illustration by [Kayamori](https://www.pixiv.net/artworks/63116446)
tags: 
  - Develop
  - Kotlin
  - Café Coroutines
---

协程为 Kotlin 提供了更优雅的异步编程方式，但由于 Jvm 本身并没有提供协程支持，因此 Kotlin/Jvm 中的协程仿佛如魔法一般的存在。

这一次我们将对协程的实现原理进行解析。

## 挂起函数

### 函数声明转换

Kotlin 从语言层面提供了 `suspend` 关键字，用于表示该函数为挂起函数，但在 Jvm 中并没有可挂起的概念。

以下面的一个挂起函数声明为例：

``` kotlin
suspend fun suspendingFunction(a: Int, vararg b: String)
```

如果通过 IDEA 将其编译的 Class 文件进行反编译，可以得到一个与其 Jvm 实现等价的 Java 方法声明：

``` java
public static final Object suspendingFunction(int a, @NotNull String[] b, @NotNull Continuation $completion)
```

挂起函数的 `suspend` 关键字转换成了一个额外的参数，其类型为 `Continuation`。这一操作是 Kotlin 编译器在编译期执行的，称为 **CPS（Continuation-Passing Style，续体传递风格）转换**，而在下一步分析协程实现原理之前，需要先对 “续体” 这一概念进行说明。

### 续体

采用回调的方法获取耗时操作的结果一般会类似如下例子中的形式：

``` kotlin
request(arguments) { result ->
    runTask(result)
}
```

在上述例子中，Lambda 作为这一代码片段中 `request()` 函数执行完毕的后续操作，这一部分被称为 **续体（Continuation）**。

协程中同样拥有续体的概念，Kotlin 中的定义为 **挂起点之后的剩余应执行的代码**，如在以下例子中，`runTask()` 便作为挂起函数 `request()` 这一挂起点的续体。

``` kotlin
val result = request(arguments)
runTask(result)
```

`Continuation` 便是续体在 Kotlin 协程中的表现形式，它是一个用于回调的接口。而 CPS 转换，在命名上听起来非常高深，但本质上即将协程代码转换为等价的回调形式，如前文的 `suspendingFunction()` 函数声明会被编译器转换为以下形式：

``` kotlin
fun suspendingFunction(a: Int, vararg b: String, $completion: Continuation<Unit>): Any?
```

就这？

仿佛一切魔法都被打破一般，协程的神奇之处仅此而已？但是，如果协程仅被单纯地转换为回调的形式，会有一个严重的问题：

**如果每个续体都要创建一个类，那该如何解决类加载的大量资源损耗？**

### 状态机

我们用上文提及的方法，看看下面这段代码的 Jvm 实现是什么：

``` kotlin
suspend fun suspendingFunction(argument: String): Boolean {
    val result = mockRequest(argument)
    return result == 0
}

suspend fun mockRequest(argument: String): Int {
    delay(1000)
    return argument.length;
}
```

这里以 `suspendingFunction()` 为例进行分析，转换所得的 Java 代码如下：

``` java
@Nullable
public static final Object suspendingFunction(@NotNull String argument, @NotNull Continuation $completion) {
    Object $continuation;
    label25: {
        if ($completion instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)$completion;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
                ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
                break label25;
            }
        }

        $continuation = new ContinuationImpl($completion) {
            // $FF: synthetic field
            Object result;
            int label;
            Object L$0;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
                this.result = $result;
                this.label |= Integer.MIN_VALUE;
                return SampleKt.suspendingFunction((String)null, this);
            }
        };
    }

    Object $result = ((<undefinedtype>)$continuation).result;
    Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
    Object var10000;
    switch(((<undefinedtype>)$continuation).label) {
        case 0:
            ResultKt.throwOnFailure($result);
            ((<undefinedtype>)$continuation).L$0 = argument;
            ((<undefinedtype>)$continuation).label = 1;
            var10000 = mockRequest(argument, (Continuation)$continuation);
            if (var10000 == var5) {
                return var5;
            }
            break;
        case 1:
            argument = (String)((<undefinedtype>)$continuation).L$0;
            ResultKt.throwOnFailure($result);
            var10000 = $result;
            break;
        default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }

    int result = ((Number)var10000).intValue();
    return Boxing.boxBoolean(result == 0);
}
```

这段 Java 代码还存在一些转换问题，因为 Kotlin 虽然生成的字节码符合 Jvm 规范，但不一定可以用 Java 生成同样的字节码。在结合字节码实际内容，并将生成代码的命名替换为可读性更强的形式之后，可以得出与 Jvm 实现等价的 Kotlin 代码。

``` kotlin
fun suspendingFunction(argument: String, completion: Continuation): Any? {

    val continuation: Continuation =
        if (completion is SuspendingFunctionStateMachine) completion.apply {
            if ((label or Int.MIN_VALUE) != 0) label -= Int.MIN_VALUE
        } else SuspendingFunctionStateMachine(completion)

    val result = continuation.result
    val resultTemp: Any? = null
    
    when (continuation.label) {
        0 -> run {
            throwOnFailure(result)
            continuation.argumentState = argument
            continuation.label = 1
            resultTemp = mockRequest(argument, continuation)
            if (resultTemp == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        1 -> run {
            argument = continuation.argumentState
            throwOnFailure(result)
            resultTemp = result
        }
        else -> throw IllegalStateException("call to 'resume' before 'invoke' with coroutine")
    }

    return (resultTemp as Int) == 0
}

class SuspendingFunctionStateMachine(
    completion: Continuation
) : ContinuationImpl(completion) {

    var result: Any? = null
    var label: Int = 0
    var argumentState: Any? = null

    override fun invokeSuspend(result: Any): Any? {
        this.result = result
        this.label = (this.label or Int.MIN_VALUE)
        return suspendingFunction(null, this)
    }
}
```

可能你会惊讶仅两行代码的挂起函数转换之后包含这么多内容，不必担心，整段代码可以拆分成两个部分进行解析，先来看看第一部分。

``` kotlin
fun suspendingFunction(argument: String, completion: Continuation): Any? {

    val continuation: Continuation =
        if (completion is SuspendingFunctionStateMachine) completion.apply {
            if ((label or Int.MIN_VALUE) != 0) label -= Int.MIN_VALUE
        } else SuspendingFunctionStateMachine(completion)

    // ..
}

class SuspendingFunctionStateMachine(
    completion: Continuation
) : ContinuationImpl(completion) {

    var result: Any? = null
    var label: Int = 0
    var argumentState: Any? = null

    override fun invokeSuspend(result: Any): Any? {
        this.result = result
        this.label = (this.label or Int.MIN_VALUE)
        return suspendingFunction(null, this)
    }
}
```

这里与一般的回调相比会略显奇特，回调部分会调用函数自身，在形式上更类似于递归。这一部分的代码便是续体的构建过程：如果该函数被上层函数所调用，则构建一个持有上层函数传入续体的新续体（这里是一个重点，但在此先不深入了解 `ContinuationImpl` 的实现）；如果是被自己的续体所调用，则直接在续体上进行修改操作即可。

接下来便是第二部分，也是挂起函数实现的重点。

``` kotlin
fun suspendingFunction(argument: String, completion: Continuation): Any? {

    // ..

    val result = continuation.result
    val resultTemp: Any? = null
    
    when (continuation.label) {
        0 -> run {
            throwOnFailure(result)
            continuation.argumentState = argument
            continuation.label = 1
            resultTemp = mockRequest(argument, continuation)
            if (resultTemp == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        1 -> run {
            argument = continuation.argumentState
            throwOnFailure(result)
            resultTemp = result
        }
        else -> throw IllegalStateException("call to 'resume' before 'invoke' with coroutine")
    }

    return (resultTemp as Int) == 0
}
```

函数中原本的多个原子操作被拆分成了 `when` 语句的多个分支，通过续体中的标签对分支进行选择，同时续体对局部变量的值进行了保存与恢复（在一定程度上类似于操作系统中进程调度时对进程状态的暂存）。可见，函数与续体构成了一个有限状态机（FSM，即 Finite-State Machine），至此，协程精妙地解决了创建回调类过多的问题。

经过 CPS 转换后，挂起函数的返回值发生了变化，在状态机中可以看到，返回值新增了一个类型 `COROUTINE_SUSPENDED`，用于表示函数当前处于挂起状态，同时这个状态也会层层上传，直到最顶层的挂起函数。

> 上述的 Kotlin 代码应当作伪代码，因为在续体内调用函数的过程中传入了一个 null，而该参数是非空的，这是因为 Kotlin/Jvm 中的非空实现是 `Intrinsics.checkParameterIsNotNull()` 函数进行了断言，而 CPS 的过程中并没有这一步操作，续体中对函数的调用仅仅是为了运行下一个状态，不需要原本的参数。

## 创建协程

上一部分完成对 Jvm 中 “不存在的事物” 的解构，所以协程最接近魔法的一层已经不复存在，但现在疑问反而更多了

- 挂起函数只能在挂起函数中调用，因为挂起函数会获取到上层传入的续体，并在自己的续体中持有，**但最上层的续体是从哪来的**？

- 谁调用了续体？

不过，如果进行一定的思考，做出一些合理的假设，之后的解析再进行证明或推翻，可以帮助理清思路。通过 CPS 转换的结果，可以大致猜测：

- `ContinuationImpl.invokeSuspend()` 的返回值如果不是 `COROUTINE_SUSPENDED`，即挂起函数执行完毕不再挂起，则续体将调用自己持有的上层续体，并将 `ContinuationImpl.invokeSuspend()` 的返回结果传给上层，从而做到下层挂起函数执行完成后唤醒上层挂起函数。

- 因为 `withContext()` 等线程切换相关的函数存在，续体中会持有关于线程调度相关的元素，比如协程调度器 Dispatcher。

打起精神，解析上述问题将会进入协程解析中最核心的部分。

### 回看续体

在此之前先来看看 `Continuation`。

``` kotlin
/**
 * Interface representing a continuation after a suspension point that returns a value of type `T`.
 */
@SinceKotlin("1.3")
public interface Continuation<in T> {
    /**
     * The context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing a successful or failed [result] as the
     * return value of the last suspension point.
     */
    public fun resumeWith(result: Result<T>)
}
```

`context` 便是协程上下文，`CoroutineContext` 是一个接口，根据我们的假设可以大致猜测与线程调度相关。

而 `resumeWith()` 便是续体的回调函数了，与我们所假设的 `invokeSuspend()` 有些出入，可以猜测在 `resumeWith()` 会调用到 `invokeSuspend()`。继续查看可以发现这个猜测是正确的。

``` kotlin
@SinceKotlin("1.3")
internal abstract class BaseContinuationImpl(
    // This is `public val` so that it is private on JVM and cannot be modified by untrusted code, yet
    // it has a public getter (since even untrusted code is allowed to inspect its call stack).
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable {
    // This implementation is final. This fact is used to unroll resumeWith recursion.
    public final override fun resumeWith(result: Result<Any?>) {
        // This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume
        var current = this
        var param = result
        while (true) {
            // Invoke "resume" debug probe on every resumed continuation, so that a debugging library infrastructure
            // can precisely track what part of suspended callstack was already resumed
            probeCoroutineResumed(current)
            with(current) {
                val completion = completion!! // fail fast when trying to resume continuation without completion
                val outcome: Result<Any?> =
                    try {
                        val outcome = invokeSuspend(param)
                        if (outcome === COROUTINE_SUSPENDED) return
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
                releaseIntercepted() // this state machine instance is terminating
                if (completion is BaseContinuationImpl) {
                    // unrolling recursion via loop
                    current = completion
                    param = outcome
                } else {
                    // top-level completion reached -- invoke and return
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }

    protected abstract fun invokeSuspend(result: Result<Any?>): Any?

    protected open fun releaseIntercepted() {
        // does nothing here, overridden in ContinuationImpl
    }

    // ..
}

@SinceKotlin("1.3")
// State machines for named suspend functions extend from this class
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {
    constructor(completion: Continuation<Any?>?) : this(completion, completion?.context)

    public override val context: CoroutineContext
        get() = _context!!

    @Transient
    private var intercepted: Continuation<Any?>? = null

    public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }

    protected override fun releaseIntercepted() {
        val intercepted = intercepted
        if (intercepted != null && intercepted !== this) {
            context[ContinuationInterceptor]!!.releaseInterceptedContinuation(intercepted)
        }
        this.intercepted = CompletedContinuation // just in case
    }
}
```

在 `BaseContinuationImpl.resumeWith()` 中可见，当 `invokeSuspend()` 返回的结果不再是 `COROUTINE_SUSPENDED` 时，**如果上层续体是由挂起函数生成的，则其 `invokeSuspend()` 会被调用**。

注意，是 `invokeSuspend()` 被调用，而不是 `resumeWith()` 被调用。

代码中存在一个 `while (true)` 循环，循环获取上层续体并执行其 `invokeSuspend()`，直至出现挂起。之所以不直接采用类似递归的调用形式，是为了避免和递归类似的问题 —— 调用栈过深。

这一问题在代码注释中也提及到。

> This loop unrolls recursion in current.resumeWith(param) to make saner and shorter stack traces on resume

在代码中还出现了 `intercepted()` 与 `releaseIntercepted()` 两个函数，`releaseIntercepted()` 的调用时机在 `invokeSuspend()` 返回真正的结果，可作出假设，存在一个续体拦截机制，这两个函数用于在该机制下，用于对续体作进一步处理。

续体拦截机制暂且不进行展开，先来看看协程是如何被启动的。

### 启动协程

通过 `CoroutineScope.launch()` 函数可以启动一个协程，并在协程对应的调度器中执行一个可挂起的扩展函数

``` kotlin
/**
 * Launches a new coroutine without blocking the current thread and returns a reference to the coroutine as a [Job].
 * The coroutine is cancelled when the resulting job is [cancelled][Job.cancel].
 *
 * The coroutine context is inherited from a [CoroutineScope]. Additional context elements can be specified with [context] argument.
 * If the context does not have any dispatcher nor any other [ContinuationInterceptor], then [Dispatchers.Default] is used.
 * The parent job is inherited from a [CoroutineScope] as well, but it can also be overridden
 * with a corresponding [context] element.
 *
 * By default, the coroutine is immediately scheduled for execution.
 * Other start options can be specified via `start` parameter. See [CoroutineStart] for details.
 * An optional [start] parameter can be set to [CoroutineStart.LAZY] to start coroutine _lazily_. In this case,
 * the coroutine [Job] is created in _new_ state. It can be explicitly started with [start][Job.start] function
 * and will be started implicitly on the first invocation of [join][Job.join].
 *
 * Uncaught exceptions in this coroutine cancel the parent job in the context by default
 * (unless [CoroutineExceptionHandler] is explicitly specified), which means that when `launch` is used with
 * the context of another coroutine, then any uncaught exception leads to the cancellation of the parent coroutine.
 *
 * See [newCoroutineContext] for a description of debugging facilities that are available for a newly created coroutine.
 *
 * @param context additional to [CoroutineScope.coroutineContext] context of the coroutine.
 * @param start coroutine start option. The default value is [CoroutineStart.DEFAULT].
 * @param block the coroutine code which will be invoked in the context of the provided scope.
 **/
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

如果暂且不考虑更具体的实现，我们可大致得出，函数创建了一个协程，启动之后将其返回，而且协程实现了 `Job` 接口，查看协程的代码实现，可以惊喜地发现：最顶层的续体实现就是协程自身

``` kotlin
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, active) {
    // ..
}
```

``` kotlin
/**
 * Abstract base class for implementation of coroutines in coroutine builders.
 *
 * This class implements completion [Continuation], [Job], and [CoroutineScope] interfaces.
 * It stores the result of continuation in the state of the job.
 * This coroutine waits for children coroutines to finish before completing and
 * fails through an intermediate _failing_ state.
 *
 * The following methods are available for override:
 *
 * * [onStart] is invoked when the coroutine was created in non-active state and is being [started][Job.start].
 * * [onCancelling] is invoked as soon as the coroutine starts being cancelled for any reason (or completes).
 * * [onCompleted] is invoked when the coroutine completes with a value.
 * * [onCancelled] in invoked when the coroutine completes with an exception (cancelled).
 *
 * @param parentContext the context of the parent coroutine.
 * @param active when `true` (by default), the coroutine is created in the _active_ state, otherwise it is created in the _new_ state.
 *               See [Job] for details.
 *
 * @suppress **This an internal API and should not be used from general code.**
 */
@InternalCoroutinesApi
public abstract class AbstractCoroutine<in T>(
    /**
     * The context of the parent coroutine.
     */
    @JvmField
    protected val parentContext: CoroutineContext,
    active: Boolean = true
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {

    // ..

    /**
     * Starts this coroutine with the given code [block] and [start] strategy.
     * This function shall be invoked at most once on this coroutine.
     *
     * First, this function initializes parent job from the `parentContext` of this coroutine that was passed to it
     * during construction. Second, it starts the coroutine based on [start] parameter:
     *
     * * [DEFAULT] uses [startCoroutineCancellable].
     * * [ATOMIC] uses [startCoroutine].
     * * [UNDISPATCHED] uses [startCoroutineUndispatched].
     * * [LAZY] does nothing.
     */
    public fun start(start: CoroutineStart, block: suspend () -> T) {
        initParentJob()
        start(block, this)
    }

    /**
     * Starts this coroutine with the given code [block] and [start] strategy.
     * This function shall be invoked at most once on this coroutine.
     *
     * First, this function initializes parent job from the `parentContext` of this coroutine that was passed to it
     * during construction. Second, it starts the coroutine based on [start] parameter:
     * 
     * * [DEFAULT] uses [startCoroutineCancellable].
     * * [ATOMIC] uses [startCoroutine].
     * * [UNDISPATCHED] uses [startCoroutineUndispatched].
     * * [LAZY] does nothing.
     */
    public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        initParentJob()
        start(block, receiver, this)
    }
}
```

> 源码中涉及了大量的扩展函数与操作符重载，`start()` 是 `CoroutineStart.invoke()` 操作符重载的形式。

为了之后的思路不至于乱成线球，在这里强调上文代码中推断所得的结论：

**在语言层面中，协程 `Coroutine` 是续体 `Continuation` 的一个实现，协程创建完毕之后将通过 `CoroutineStart.invoke()` 函数，让挂起函数运行，并将协程作为最顶层的续体进行传递**

以默认的 `CoroutineStart.DEFAULT` 为例，继续解析

``` kotlin
/**
 * Defines start options for coroutines builders.
 * It is used in `start` parameter of [launch][CoroutineScope.launch], [async][CoroutineScope.async], and other coroutine builder functions.
 *
 * The summary of coroutine start options is:
 * * [DEFAULT] -- immediately schedules coroutine for execution according to its context;
 * * [LAZY] -- starts coroutine lazily, only when it is needed;
 * * [ATOMIC] -- atomically (in a non-cancellable way) schedules coroutine for execution according to its context;
 * * [UNDISPATCHED] -- immediately executes coroutine until its first suspension point _in the current thread_.
 */
public enum class CoroutineStart {
    
    // ..

    /**
     * Starts the corresponding block as a coroutine with this coroutine's start strategy.
     *
     * * [DEFAULT] uses [startCoroutineCancellable].
     * * [ATOMIC] uses [startCoroutine].
     * * [UNDISPATCHED] uses [startCoroutineUndispatched].
     * * [LAZY] does nothing.
     *
     * @suppress **This an internal API and should not be used from general code.**
     */
    @InternalCoroutinesApi
    public operator fun <T> invoke(block: suspend () -> T, completion: Continuation<T>) =
        when (this) {
            CoroutineStart.DEFAULT -> block.startCoroutineCancellable(completion)
            // ..
        }

    /**
     * Starts the corresponding block with receiver as a coroutine with this coroutine start strategy.
     *
     * * [DEFAULT] uses [startCoroutineCancellable].
     * * [ATOMIC] uses [startCoroutine].
     * * [UNDISPATCHED] uses [startCoroutineUndispatched].
     * * [LAZY] does nothing.
     *
     * @suppress **This an internal API and should not be used from general code.**
     */
    @InternalCoroutinesApi
    public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>) =
        when (this) {
            CoroutineStart.DEFAULT -> block.startCoroutineCancellable(receiver, completion)
            // ..
        }

    /**
     * Returns `true` when [LAZY].
     *
     * @suppress **This an internal API and should not be used from general code.**
     */
    @InternalCoroutinesApi
    public val isLazy: Boolean get() = this === LAZY
}
```

再看 `startCoroutineCancellable()`。

``` kotlin
/**
 * Use this function to start coroutine in a cancellable way, so that it can be cancelled
 * while waiting to be dispatched.
 */
internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(receiver: R, completion: Continuation<T>) =
    runSafely(completion) {
        createCoroutineUnintercepted(receiver, completion).intercepted().resumeCancellableWith(Result.success(Unit))
    }
```

在 `createCoroutineUnintercepted()` 中，挂起函数以强制转换的形式变成了最原本的形式，最后到达最终的目的地 `createCoroutineFromSuspendFunction()`，用一个新的续体对在 `launch()` 中传入的可挂起 Lambda 进行包装。

``` kotlin
/**
 * Creates unintercepted coroutine without receiver and with result type [T].
 * This function creates a new, fresh instance of suspendable computation every time it is invoked.
 *
 * To start executing the created coroutine, invoke `resume(Unit)` on the returned [Continuation] instance.
 * The [completion] continuation is invoked when coroutine completes with result or exception.
 *
 * This function returns unintercepted continuation.
 * Invocation of `resume(Unit)` starts coroutine immediately in the invoker's call stack without going through the
 * [ContinuationInterceptor] that might be present in the completion's [CoroutineContext].
 * It is the invoker's responsibility to ensure that a proper invocation context is established.
 * Note that [completion] of this function may get invoked in an arbitrary context.
 *
 * [Continuation.intercepted] can be used to acquire the intercepted continuation.
 * Invocation of `resume(Unit)` on intercepted continuation guarantees that execution of
 * both the coroutine and [completion] happens in the invocation context established by
 * [ContinuationInterceptor].
 *
 * Repeated invocation of any resume function on the resulting continuation corrupts the
 * state machine of the coroutine and may result in arbitrary behaviour or exception.
 */
@SinceKotlin("1.3")
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (/* .. */)
        // ..
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}
```

``` kotlin
/**
 * This function is used when [createCoroutineUnintercepted] encounters suspending lambda that does not extend BaseContinuationImpl.
 *
 * It happens in two cases:
 *   1. Callable reference to suspending function,
 *   2. Suspending function reference implemented by Java code.
 *
 * We must wrap it into an instance that extends [BaseContinuationImpl], because that is an expectation of all coroutines machinery.
 * As an optimization we use lighter-weight [RestrictedContinuationImpl] base class (it has less fields) if the context is
 * [EmptyCoroutineContext], and a full-blown [ContinuationImpl] class otherwise.
 *
 * The instance of [BaseContinuationImpl] is passed to the [block] so that it can be passed to the corresponding invocation.
 */
@SinceKotlin("1.3")
private inline fun <T> createCoroutineFromSuspendFunction(
    completion: Continuation<T>,
    crossinline block: (Continuation<T>) -> Any?
): Continuation<Unit> {
    val context = completion.context
    // label == 0 when coroutine is not started yet (initially) or label == 1 when it was
    return if (context === EmptyCoroutineContext)
        object : RestrictedContinuationImpl(completion as Continuation<Any?>) {
            private var label = 0

            override fun invokeSuspend(result: Result<Any?>): Any? =
                when (label) {
                    0 -> {
                        label = 1
                        result.getOrThrow() // Rethrow exception if trying to start with exception (will be caught by BaseContinuationImpl.resumeWith
                        block(this) // run the block, may return or suspend
                    }
                    1 -> {
                        label = 2
                        result.getOrThrow() // this is the result if the block had suspended
                    }
                    else -> error("This coroutine had already completed")
                }
        }
    else
        object : ContinuationImpl(completion as Continuation<Any?>, context) {
            private var label = 0

            override fun invokeSuspend(result: Result<Any?>): Any? =
                when (label) {
                    0 -> {
                        label = 1
                        result.getOrThrow() // Rethrow exception if trying to start with exception (will be caught by BaseContinuationImpl.resumeWith
                        block(this) // run the block, may return or suspend
                    }
                    1 -> {
                        label = 2
                        result.getOrThrow() // this is the result if the block had suspended
                    }
                    else -> error("This coroutine had already completed")
                }
        }
}
```

### 协程上下文

上文实际上还留着一个坑，只不过为了保证思路不偏离并没有进行展开。

在解析 `Continuation` 接口时，我们猜测协程调度器实现的 `CoroutineContext` 是进行线程调度的主体，但无论是 `ContinuationImpl` 中的 `intercepted()` 与 `releaseIntercepted()`，还是 `startCoroutineCancellable` 中的 `intercepted()`，最终都将我们引向了另一个接口：`ContinuationInterceptor`。

更有意思的是，`ContinuationInterceptor` 的实现是通过 `ConroutineContext` 的实现获取的。

好吧，在看这一部分代码之前，我认为先讲结论会更合适一点。

**协程上下文 `CoroutineContext` 用于记录当前协程所持有的信息，是协程运行中一个重要的数据对象。**

- `CoroutineContext` 是一个 `CoroutineContext.Element` 的索引集，可通过 `CoroutineContext.Key` 获取对应的 `CoroutineContext.Element`。

- `CoroutineContext.Element` 继承 `CoroutineContext`。

- `ContinuationInterceptor` 继承 `CoroutineContext.Element`。

- `Job`、`CoroutineName`、`CoroutineId` 均继承 `CoroutineContext.Element`。

接下来展示的是协程之中这一秀到令人头皮发麻的设计。

``` kotlin
/**
 * Persistent context for the coroutine. It is an indexed set of [Element] instances.
 * An indexed set is a mix between a set and a map.
 * Every element in this set has a unique [Key].
 */
@SinceKotlin("1.3")
public interface CoroutineContext {
    /**
     * Returns the element with the given [key] from this context or `null`.
     */
    public operator fun <E : Element> get(key: Key<E>): E?

    // ..

    /**
     * Key for the elements of [CoroutineContext]. [E] is a type of element with this key.
     */
    public interface Key<E : Element>

    /**
     * An element of the [CoroutineContext]. An element of the coroutine context is a singleton context by itself.
     */
    public interface Element : CoroutineContext {
        /**
         * A key of this coroutine context element.
         */
        public val key: Key<*>

        public override operator fun <E : Element> get(key: Key<E>): E? =
            @Suppress("UNCHECKED_CAST")
            if (this.key == key) this as E else null

        // ..
    }
}
```

再看看 `ContinuationInterceptor` 是如何 “获取自身” 的：以伴生对象为 Key，通过重写 `get()` 函数将实例自身（可以是子类的实例）作为续体拦截器返回。

``` kotlin
/**
 * Marks coroutine context element that intercepts coroutine continuations.
 * The coroutines framework uses [ContinuationInterceptor.Key] to retrieve the interceptor and
 * intercepts all coroutine continuations with [interceptContinuation] invocations.
 *
 * [ContinuationInterceptor] behaves like a [polymorphic element][AbstractCoroutineContextKey], meaning that
 * its implementation delegates [get][CoroutineContext.Element.get] and [minusKey][CoroutineContext.Element.minusKey]
 * to [getPolymorphicElement] and [minusPolymorphicKey] respectively.
 * [ContinuationInterceptor] subtypes can be extracted from the coroutine context using either [ContinuationInterceptor.Key]
 * or subtype key if it extends [AbstractCoroutineContextKey].
 */
@SinceKotlin("1.3")
public interface ContinuationInterceptor : CoroutineContext.Element {
    /**
     * The key that defines *the* context interceptor.
     */
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>

    public override operator fun <E : CoroutineContext.Element> get(key: CoroutineContext.Key<E>): E? {
        // ..
        @Suppress("UNCHECKED_CAST")
        return if (ContinuationInterceptor === key) this as E else null
    }

    // ..
}
```

### 续体拦截机制

聊完了 `CoroutineContext` 与 `ContinuationInterceptor` 的关系便可以解析线程调度的实现了。回到我们前面所提到的续体拦截机制，所有代码最终都指向了 `ContinuationInterceptor`。

``` kotlin
/**
 * Marks coroutine context element that intercepts coroutine continuations.
 * The coroutines framework uses [ContinuationInterceptor.Key] to retrieve the interceptor and
 * intercepts all coroutine continuations with [interceptContinuation] invocations.
 *
 * [ContinuationInterceptor] behaves like a [polymorphic element][AbstractCoroutineContextKey], meaning that
 * its implementation delegates [get][CoroutineContext.Element.get] and [minusKey][CoroutineContext.Element.minusKey]
 * to [getPolymorphicElement] and [minusPolymorphicKey] respectively.
 * [ContinuationInterceptor] subtypes can be extracted from the coroutine context using either [ContinuationInterceptor.Key]
 * or subtype key if it extends [AbstractCoroutineContextKey].
 */
@SinceKotlin("1.3")
public interface ContinuationInterceptor : CoroutineContext.Element {

    // ..

    /**
     * Returns continuation that wraps the original [continuation], thus intercepting all resumptions.
     * This function is invoked by coroutines framework when needed and the resulting continuations are
     * cached internally per each instance of the original [continuation].
     *
     * This function may simply return original [continuation] if it does not want to intercept this particular continuation.
     *
     * When the original [continuation] completes, coroutine framework invokes [releaseInterceptedContinuation]
     * with the resulting continuation if it was intercepted, that is if `interceptContinuation` had previously
     * returned a different continuation instance.
     */
    public fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>

    /**
     * Invoked for the continuation instance returned by [interceptContinuation] when the original
     * continuation completes and will not be used anymore. This function is invoked only if [interceptContinuation]
     * had returned a different continuation instance from the one it was invoked with.
     *
     * Default implementation does nothing.
     *
     * @param continuation Continuation instance returned by this interceptor's [interceptContinuation] invocation.
     */
    public fun releaseInterceptedContinuation(continuation: Continuation<*>) {
        /* do nothing by default */
    }

    // ..
}
```

续体拦截器用于拦截一个续体（通常在 `launch()` 或 `withContext()` 等对可线程进行调度的函数进行）。最常见的续体拦截器即协程调度器，它们均继承自 `CoroutineDispatcher`。

``` kotlin
/**
 * Base class to be extended by all coroutine dispatcher implementations.
 *
 * The following standard implementations are provided by `kotlinx.coroutines` as properties on
 * the [Dispatchers] object:
 *
 * * [Dispatchers.Default] &mdash; is used by all standard builders if no dispatcher or any other [ContinuationInterceptor]
 *   is specified in their context. It uses a common pool of shared background threads.
 *   This is an appropriate choice for compute-intensive coroutines that consume CPU resources.
 * * [Dispatchers.IO] &mdash; uses a shared pool of on-demand created threads and is designed for offloading of IO-intensive _blocking_
 *   operations (like file I/O and blocking socket I/O).
 * * [Dispatchers.Unconfined] &mdash; starts coroutine execution in the current call-frame until the first suspension,
 *   whereupon the coroutine builder function returns.
 *   The coroutine will later resume in whatever thread used by the
 *   corresponding suspending function, without confining it to any specific thread or pool.
 *   **The `Unconfined` dispatcher should not normally be used in code**.
 * * Private thread pools can be created with [newSingleThreadContext] and [newFixedThreadPoolContext].
 * * An arbitrary [Executor][java.util.concurrent.Executor] can be converted to a dispatcher with the [asCoroutineDispatcher] extension function.
 *
 * This class ensures that debugging facilities in [newCoroutineContext] function work properly.
 */
public abstract class CoroutineDispatcher :
    AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {

    /** @suppress */
    @ExperimentalStdlibApi
    public companion object Key : AbstractCoroutineContextKey<ContinuationInterceptor, CoroutineDispatcher>(
        ContinuationInterceptor,
        { it as? CoroutineDispatcher })

    /**
     * Returns `true` if the execution of the coroutine should be performed with [dispatch] method.
     * The default behavior for most dispatchers is to return `true`.
     *
     * If this method returns `false`, the coroutine is resumed immediately in the current thread,
     * potentially forming an event-loop to prevent stack overflows.
     * The event loop is an advanced topic and its implications can be found in [Dispatchers.Unconfined] documentation.
     *
     * A dispatcher can override this method to provide a performance optimization and avoid paying a cost of an unnecessary dispatch.
     * E.g. [MainCoroutineDispatcher.immediate] checks whether we are already in the required UI thread in this method and avoids
     * an additional dispatch when it is not required.
     *
     * While this approach can be more efficient, it is not chosen by default to provide a consistent dispatching behaviour
     * so that users won't observe unexpected and non-consistent order of events by default.
     *
     * Coroutine builders like [launch][CoroutineScope.launch] and [async][CoroutineScope.async] accept an optional [CoroutineStart]
     * parameter that allows one to optionally choose the [undispatched][CoroutineStart.UNDISPATCHED] behavior to start coroutine immediately,
     * but to be resumed only in the provided dispatcher.
     *
     * This method should generally be exception-safe. An exception thrown from this method
     * may leave the coroutines that use this dispatcher in the inconsistent and hard to debug state.
     */
    public open fun isDispatchNeeded(context: CoroutineContext): Boolean = true

    /**
     * Dispatches execution of a runnable [block] onto another thread in the given [context].
     * This method should guarantee that the given [block] will be eventually invoked,
     * otherwise the system may reach a deadlock state and never leave it.
     * Cancellation mechanism is transparent for [CoroutineDispatcher] and is managed by [block] internals.
     *
     * This method should generally be exception-safe. An exception thrown from this method
     * may leave the coroutines that use this dispatcher in the inconsistent and hard to debug state.
     *
     * This method must not immediately call [block]. Doing so would result in [StackOverflowError]
     * when [yield] is repeatedly called from a loop. However, an implementation that returns `false` from
     * [isDispatchNeeded] can delegate this function to `dispatch` method of [Dispatchers.Unconfined], which is
     * integrated with [yield] to avoid this problem.
     */
    public abstract fun dispatch(context: CoroutineContext, block: Runnable)

    // ..

    /**
     * Returns a continuation that wraps the provided [continuation], thus intercepting all resumptions.
     *
     * This method should generally be exception-safe. An exception thrown from this method
     * may leave the coroutines that use this dispatcher in the inconsistent and hard to debug state.
     */
    public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        DispatchedContinuation(this, continuation)

    @InternalCoroutinesApi
    public override fun releaseInterceptedContinuation(continuation: Continuation<*>) {
        (continuation as DispatchedContinuation<*>).reusableCancellableContinuation?.detachChild()
    }

    // ..
}
```

调度器将原有的续体包装为一个新类型的续体：`DispatchedContinuation`，并在其 `resumeWith()` 被调用时通过 `CoroutineDispatcher.dispatch()` 函数将代码以 `Runnable` 的形式推送到指定线程运行。

> 作为优化，`resumeWith()` 调用时会通过 `CoroutineDispatcher.isDispatchNeeded()` 判断是否需要进行调度。以 Android 中的 `Dispatchers.Main` 为例，如果在调度前的续体已经在主线程运行，则不必再进行调度工作。

``` kotlin
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_ATOMIC_DEFAULT), CoroutineStackFrame, Continuation<T> by continuation {
    
    // ..

    override fun resumeWith(result: Result<T>) {
        val context = continuation.context
        val state = result.toState()
        if (dispatcher.isDispatchNeeded(context)) {
            _state = state
            resumeMode = MODE_ATOMIC_DEFAULT
            dispatcher.dispatch(context, this)
        } else {
            executeUnconfined(state, MODE_ATOMIC_DEFAULT) {
                withCoroutineContext(this.context, countOrElement) {
                    continuation.resumeWith(result)
                }
            }
        }
    }

    // ..
}
```

## 参考链接

- [GitHub - Kotlin Coroutines](https://github.com/Kotlin/kotlinx.coroutines)

- [GitHub - 协程设计文档](https://github.com/Kotlin-zh/KEEP/blob/master/proposals/coroutines.md)

- [简书 - Kotlin/Jvm 协程实现原理](https://www.jianshu.com/p/d23c688feae7)