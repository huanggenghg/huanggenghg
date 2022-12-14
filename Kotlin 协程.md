## Kotlin 协程

### 一、基础使用

优点：轻量、内存泄漏更少、内置取消支持、Jetpack 集成

```kotlin
fun main() {
	GlobalScope.launch(context = Dispatchers.IO) {
		delay(1000)
		log("launch")
	}
	Thread.sleep(2000)
	log("end")
}

public suspend fun delay(timeMillis: Long) {

}
```

#### 四个基础概念：

- suspend function
- CoroutineScope
- ConroutineContext
- CoroutineBuilder：协程构建器，协程在 CoroutineScope 的上下文中通过 launch、async 等协程构建器来进行声明并启动。launch、async 均被声明为 CoroutineScope 的扩展方法。

#### suspend 挂起与恢复

`suspend`、`resume`

Kotlin 使用**堆栈帧**来管理运行哪个函数和及所有局部变量。

Android 平台上协程主要解决两个问题：
1. 处理耗时任务，避免阻塞主线程
2. 保证主线程安全

#### CoroutineScope 协程作用域

跟踪管理协程，Kotlin 不允许在没有 CoroutineScope 情况下启动协程。

大体有三种：
- GlobalScope。全局作用域。在这个范围内启动的协程可以一直运行直到应用停止运行。GlobalScope 本身不会阻塞当前线程，且**启动的协程相当于守护线程，不会阻止 JVM 结束运行**。
- runBlocking。只有当内部**相同作用域**的所有协程都运行结束后，声明在 runBlocking 之后的代码才能执行，即 runBlocking 会阻塞其所在线程。
- supervisorScope。抛出异常不会连锁取消同级协程和父协程。
- `CoroutineScope()` 或 `MainScope()`

```kotlin
private val mainScope = MainScope()

fun onDestroy() {
	mainScope.cancel()
}

// 委托模式
class Activity : CoroutineScope by CoroutineScope(Dispatchers.Default) {
	fun main = runBlocking {
		//...
	}
	fun onDestroy() {
		cancel()
	}
}
```

#### CoroutineBuilder

launch：

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT, // 协程启动方式
    block: suspend CoroutineScope.() -> Unit
): Job
```

> 协程启动方式：
> - CoroutineStart.DEFAULT：立即进入等待调度
> - CoroutineStart.LAZY：懒启动，延迟启动
> - CoroutineStart.ATOMIC：原子性，启动前无法取消
> - CoroutineStart.DEFAULT：立即进入等待调度

Job：协程句柄。使用 launch 或 async 创建的每个协程都会返回一个 Job 实例，该实例唯一标识协程并管理其生命周期。

async：可返回结果

```kotlin
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T>
```
```kotlin
val deferreds = listOf(
		async { fetchDoc(1) },
		async { fetchDoc(2) }
	)
deferreds.awaitAll()
```

Defered 继承于 Job，扩展了`await()`

#### CoroutineContext

CoroutineContext 使用以下元素集定义协程的行为：
- Job：控制协程的生命周期
- CoroutineDispatcher：将任务指派给适当的线程
- CoroutineName：协程的名称，可用于调试
- CoroutineExceptionHandler：处理未捕获的异常

协程中的 Job 是其上下文 CoroutineContext 中的一部分，可以通过`coroutineContext[Job]`表达式从上下文中获取到，我们可以通过控制 Job 来控制 CoroutineScope 的生命周期。

CoroutineContext 包含一个 CoroutineDispatcher（协程调度器）用于指定执行协程的目标载体，即 运行于哪个线程。

withContext：不引入回调的情况下控制任何代码的执行线程池。

CoroutineName：用于为协程指定一个名字，方便调试和定位问题。

CoroutineExceptionHandler：异常处理

组合上下文元素：`+`运算符，可同时指定协程的 Dispatcher 和 CoroutineName

#### 取消协程

`cancel()`：取消协程，但协程不一定就是已经停止运行了。
`join()`：阻塞等待协程运行结束。
`cancelAndJoin()`：取消并阻塞等待协程结束。

协程的取消是协作完成的。协程库中的所有挂起函数都是可取消的，它们在运行前检查协程是否被取消了，并在取消时抛出 CancellationException 从而结束整个任务。而如果协程在执行计算任务前没有判断自身是否已被取消的话，此时就无法取消协程。

NonCancellable：无法取消的协程作用域

传播取消操作：一般情况下，协程的取消操作会通过协程的层次结构来进行传播：如果取消父协程或者父协程抛出异常，那么子协程都会被取消；而如果子协程被取消，则不会影响同级协程和父协程，但如果子协程抛出异常则也会导致同级协程和父协程被取消。

#### 异常处理

CoroutineExceptionHandler：异常捕获处理
SupervisorJob：向下传播

摘录自[一文快速入门 Kotlin 协程](https://juejin.cn/post/6908271959381901325)

### 二、原理

优秀[Kotlin 协程硬核解读](https://blog.csdn.net/xmxkf/category_11003996.html)，以下是摘录。

kotlin 协程**解决了异步任务非阻塞同步返回数据的问题，达到简化代码逻辑、所见即所得的目的**。

协程作用域是用于构建协程的，它维护的协程上下文数据将作为初始上下文对象传递给被创建的协程对象，提供的扩展函数cancel()可用于取消协程从而控制协程生命周期。

协程对象中的上下文 = 初始上下文(作用域的上下文or父协程上下文) + 构建器参数上下文 + 续体拦截器(调度器) + Job(协程对象本身) （上下文可以通过+组合成新对象）。

**两个协程之所以能形成父子关系的关键原因是子协程继承了父协程的上下文对象中的Job 上下文元素，从而传播可取消性。**

```kotlin
GlobalScope.launch {
	// 非子协程，父协程上下文的 job 对象被顶替
	CoroutineScope(coroutineContext + Job()).launch { ... }
}
```

协程代码块中必须调用挂起函数才能使cancel()函数取消成功。对父级Job的cancel()调用会导致其所有子Job递归地立即取消，调用子Job的cancel()不会影响父Job和它的兄弟Job。

#### suspend 挂起函数，挂起和恢复的实现原理

不阻塞当前线程就是**挂起**，它指的是当协程中调用挂起函数时会记录当前的状态并挂起(暂停)协程的执行（释放当前线程），以达到非阻塞等待异步计算结果的目的。说白了就是不阻塞协程代码块所在的线程但暂停挂起点之后的代码执行，当挂起函数执行完毕再恢复挂起点后的代码执行。（**异步**）

`suspend`

 - 在编码阶段：表示一个挂起函数，它只能在协程或其他挂起函数中被调用。
 - 在编译阶段：函数会增加一个`Continuation`（续体）类型的参数。

 不完全挂起函数（组合挂起、内部挂起）：
 ```kotlin
suspend fun getUser2():User{
    delay(1000)  // 真正的挂起点，这部分执行的耗时是在协程所在线程中的，这部分的耗时计算不是在线程被挂起的状态下。（可理解为这里才真正去执行挂起动作）
    println("假的挂起函数${Thread.currentThread()}")
    Thread.sleep(1000)
    return User("openXu")
}
 ```

真正的、完全的挂起函数
```kotlin
//调用suspendCancellableCoroutine()，将函数体作为参数传入
suspend fun getUser(): User = suspendCancellableCoroutine {
	//被拦截后的可取消续体对象
    cancellableContinuation ->
    println("挂起函数执行线程${Thread.currentThread()}") //Thread[main,5,main]
    Thread.sleep(3000)
    cancellableContinuation.resume(User("openXu"))
    cancellableContinuation.cancel()
}
```

```kotlin
public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T =
    suspendCoroutineOrReturn { c: Continuation<T> ->
        val safe = SafeContinuation(c)
        block(safe)
        safe.getResult()
    }

public suspend inline fun <T> suspendCancellableCoroutine(
    crossinline block: (CancellableContinuation<T>) -> Unit
): T =
    suspendCoroutineUninterceptedOrReturn { uCont ->
        val cancellable = CancellableContinuationImpl(uCont.intercepted(), resumeMode = MODE_CANCELLABLE)
        // NOTE: Before version 1.1.0 the following invocation was inlined here, so invocation of this
        // method indicates that the code was compiled by kotlinx.coroutines < 1.1.0
        // cancellable.initCancellability()
        block(cancellable)
        cancellable.getResult()
    }
```
> **挂起函数不一定是在子线程执行的。**

```kotlin
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
	...
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
		...
    }
}
```
当函数体已经具备异步能力的就选suspendCancellableCoroutine()，而不具备异步能力(需要手动切线程)的就选withContext()。

需要注意的是，如果使用withContext()则直接通过return返回计算结果或者通过throw抛异常来返回错误从而恢复协程执行；而用suspendCancellableCoroutine()的时候则通过续体continuation的resume系列方法来实现协程恢复。

![Continuation](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%8D%8F%E7%A8%8B%E7%BB%AD%E4%BD%93.drawio.png)

关于续体，**续体是对挂起点之后代码块的封装，表示挂起函数执行完后需要恢复继续执行的代码。同时可以将它当作一个CallBack回调，因为可以调用其resume方法返回挂起点函数的计算结果（成功or失败）**。

续体传递风格 CPS（Continuation-Passing-Style）：每个挂起函数在编译后都会附加一个`Continuation`类型的参数，在调用时隐式传入续体对象（在挂起函数中可通过续体对象的`resume`系列函数返回成功值或者异常来恢复协程的执行）。

![续体传递风格 CPS](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/whyGetResultAny.drawio.png)

**SuspendLambada**

```md
Continuation 续体：代表协程下一步应该执行的代码
  |
  |-- BaseContinuationImpl 续体基本实现：主要实现了resumeWith(Result)函数，以控制续体状态机的执行流程，定义了抽象方法invokeSuspend(Result)
        |	
        |-- ContinuationImpl 续体实现：增加了拦截器intercepted()功能，实现线程调度等等
               |
               |-- SuspendLambda 挂起lambda表达式：是对协程代码块中代码的封装
                     |
                     |-- kotlin编译为协程生成的匿名子类：将协程代码块中的代码分为n个状态后作为invokeSuspend(Result)的函数体从而实现该函数
```

**kotlin 在编译时会为**每个协程**生成一个`SuspendLambda`的匿名子类，并创建其对象。**
- 对创建协程时传入的挂起Lambda表达式中代码块的封装，将代码块中的代码以调用挂起函数为分割点分为多个部分后填充到invokeSuspend()函数中从而实现SuspendLambda。
- 实现了Continuation，本身就是一个续体对象，resumeWith()具有恢复协程执行的能力。

**状态机**

考虑到性能，kotlin 协程在实现的时候，并不是调用了多少个挂起函数（多少个挂起点），就需要多少个续体对象。（一个协程对应一个匿名 SuspendLambda 子类，一个其（也就是续体）对象）。通过状态机实现，维护一个 Int 类型的状态值，笔记续体执行到哪个步骤。

invokeSuspend()函数就是对协程代码块中代码的封装，只是加入了状态机将代码分为多个部分，当协程开始执行时会首先调用一次invokeSuspend()函数触发协程代码块初始状态的代码执行，当调用到挂起函数时，挂起函数会返回一个COROUTINE_SUSPENDED标志，导致invokeSuspend()的renturn停止剩下代码的执行，这就是协程的非阻塞挂起。当挂起函数执行完成会调用续体的resumeWith()函数以返回函数结果或者异常，而resumeWith()中又调用了invokeSuspend()根据状态机的状态值恢复执行协程代码块下一个状态的代码，这就是协程的恢复。所以**协程的执行实际上就是n+1次（n==协程代码块中调用的挂起函数数量）调用invokeSuspend()，整个流程是通过resumeWith()和invokeSuspend()中的状态机switch来控制的**。

```java
fun main() {
    GlobalScope.launch {
        //--------------------初始状态0----------------------
        println("状态0")
        delay(1000)         //挂起点1
        //--------------------状态1----------------------
        println("状态1")
        var user= getUser()  //挂起点2
        //--------------------状态2----------------------
        println("状态2 $user")
        delay(1000)          //挂起点3
        //--------------------状态3----------------------
        println("状态3")
    }
    Thread.sleep(5000)
}


final class runable/SuspendKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 {
	...
	int label = 0;   //★ 状态码
    //调用续体的resumeWith()会触发invokeSuspend()，所以这里将invokeSuspend()直接换成resumeWith()
    public final void resumeWith(@NotNull Object result) {
        //每次调用都会根据状态值跳转到不同的代码块，从而执行不同的状态代码
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        if (label == 3) goto L3
        else throw IllegalStateException()
        L0: {  //--------------------初始状态0----------------------
            println("状态0")
            label = 1
            result = delay(1000, this)  //挂起点1，this作为续体传递
            //挂起协程，通过return跳出方法停止执行来实现
            if (result == COROUTINE_SUSPENDED) return
        }
        L1: {  //--------------------状态1----------------------
            println("状态1")
            label = 2
            result = getUser(this)    //挂起点2，this作为续体传递
            if (result == COROUTINE_SUSPENDED) return
        }
        L2: {  //--------------------状态2----------------------
            Object user = result   //getUser()返回的User对象
            println("状态2 $user")
            label = 3
            result = delay(1000, this)   //挂起点3，this作为续体传递
            if (result == COROUTINE_SUSPENDED) return
        }
        L3: {  //--------------------状态3----------------------
            println("状态3")
            return   //lambda表达式执行完成（续体执行完毕），直接return
        }
    }
	...
}
```
```java
fun main() {
    runBlocking{
        //--------------------父 初始状态0----------------------
        println("父协程 $this")
        //调用挂起函数withContext()开启子协程
        val result = withContext(Dispatchers.IO){
            //--------------------子 初始状态0----------------------
            println("子协程 $this")
            delay(1000)
            //--------------------子 状态1----------------------
            val str = "ABC"
            println("挂起函数返回值$str")
            str
        }
        //--------------------父 状态1----------------------
        println(result)
    }
}

/* 
  两个协程就对应两个SuspendLambda的子类，因为resumeWith()的目的是调用invokeSuspend()方法，这里直接换成了resumeWith()更加直观 
*/
//1 父协程的SuspendLambda封装
final class runable/SuspendKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 {
	...
	int label = 0;   //★ 父状态码
    public final void resumeWith(@NotNull Object result) {  //main()所在的主线程执行
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        L0: {  //--------------------父 初始状态0----------------------
            println("父协程 $this")
            label = 1
            result = withContext(Dispatchers.IO, this){}  //挂起点1
            //挂起协程，return
            if (result == COROUTINE_SUSPENDED) return
        }
        L1: {  //--------------------状态1----------------------
           println(result)
           return   //lambda表达式执行完成（续体执行完毕），直接return
        }
    }
	...
}

//2 子协程的SuspendLambda封装
final class runable/SuspendKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 {
	...
	int label = 0;   //★ 子状态码
    public final void resumeWith(@NotNull Object result) {  //Dispatchers.IO子线程中执行
        if (label == 0) goto L0
        if (label == 1) goto L1
        else throw IllegalStateException()
        L0: {  //--------------------子 初始状态0----------------------
            println("子协程 $this")
            label = 1
            result = delay(1000, this) //挂起点1
            if (result == COROUTINE_SUSPENDED) return  //挂起协程，return
        }
        L1: {  //--------------------状态1----------------------
           val str = "ABC"
           println("挂起函数返回值$str")
           return str       //返回结果值
        }
    }
	...
}
```

总结：**协程的return式非阻塞挂起和resumeWith()恢复的交替保证了协程代码块中代码的顺序执行，所以在协程代码块中可以将异步逻辑编写为同步的上下顺序结构**。

![SuspendLambda 内部状态机](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/109ceba032ae4363b5bd144b3f1c6e8d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

#### 协程的创建和启动流程分析

启动方式：`runBlocking{}`、`launch{}`、`async{}`、`withContext()`等等。

启动协程有多种方式，通过runBlocking {}或者CoroutineScope作用域的扩展launch{}创建最外层协程，或者通过扩展async {}创建并发子协程、通过withContext()创建子协程等等，不管是在非挂起作用域创建外层协程还是创建子协程，其实步骤都是差不多的。

**创建外层协程时根据作用域的上下文对象构建一个协程对象，然后根据启动模式启动协程；而创建子协程时则是将父协程的实例当作作用域，然后重复这个步骤。**

- 使用协程作用域CoroutineScope的扩展函数launch()创建一个协程对象coroutine(AbstractCoroutine的子类对象)，并将作用域的上下文+参数上下文+调度器(拦截器)组合成一个新的上下文对象传递给协程构造函数，绑定作用域上下文中的Job和协程对象的父子关系以传递取消功能，最后调用协程的start()启动协程。
- AbstractCoroutine.start()函数将启动协程的任务交给了启动模式CoroutineStart的调用操作符重载函数invoke()，根据不同的启动模式启动协程，默认启动模式DEFAULT会立即执行启动。
- 将协程对象传递给SuspendLambda匿名子类的create()函数创建原始续体对象，这是对协程对象的包装。
- 然后从上下文对象中获取拦截器(通常是一个协程调度器对象)并调用其interceptContinuation()拦截原始续体对象将其包装成DispatchedContinuation类型的续体，这个代理续体维护了调度器dispatcher和原始续体continuation对象。
- 将代理续体对象当作Runnable扔进调度器dispatcher维护的线程池中完成线程切换，run()方法中调用原始续体的resumeWith()从而触发SuspendLambda的invokeSuspend()开始执行协程代码块。

![协程启动流程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%8D%8F%E7%A8%8B%E7%9A%84%E5%90%AF%E5%8A%A8%E3%80%81%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.drawio.png)

总结：
`AbstractCoroutine`表示协程的抽象类，其子类对象就是协程对象；`SuspendLambda`更倾向将其看作协程，因为`SuspendLambda`的子类对象控制了协程代码块和其执行顺序。他们都可看做协程，但都不是完整的协程，协程其实有 3 层包装：

- 第一层就是`launch`和`async`返回的`Job`、`Deferred`，里面封装了协程状态，提供了取消协程接口，而它们的实例都是继承自`AbstractCoroutine`，`AbstractCoroutine`就是协程的第一层包装；
- 第二层包装是编译器生成的`SuspendLambda`的子类，封装了协程执行的代码和执行逻辑，`SuspendLambda`又继承自`BaseContinuationImpl`，其`completion`属性就是协程的第一层包装；
- 第三层包装是协程的线程调度器`DispatchedContinuation`，封装了线程调度逻辑，并持有协程的第二层包装对象。

三层包装都实现了`Continuation`接口，通过装饰模式组合在一起，每层负责不同的功能，其实这里也可以说是代理模式，因为装饰模式和代理模式的共同点是外层持有内层的引用。但是说是装饰模式更加合适，因为不同的层负责不同的功能，可以看作是对原始协程对象的功能增强。

#### 协程异常传播取消和异常处理机制

`resumeWithException()`：
- 续体对象会被包装为`DispatchedContinuatoin`类型的调度器续体对象，它的`resume`系列函数将成功结果或异常封装为一个 Result 对象，然后`result`通过调度器切换到协程所在的线程（主线程）传递给`SuspendLambda.invokeSuspend(result)`函数
- `SuspendLambda.invokeSuspend()`中检测`result`是如果是异常类型则通过`throw`关键字将异常抛出。

协程代码块中之所以能直接捕获在异步挂起函数中"抛出"的异常，是因为**协程中通过调度器将子线程的异常对象切换到了协程所在线程，然后在协程代码块中throw抛出。如果我们在挂起函数的子线程中直接通过throw抛出异常，协程代码块中的try并不能捕获到，协程隐藏了线程切换的实现细节，让我们直接可以在协程代码块中捕获挂起函数中通过resume恢复的异常，从而将异步异常"变为"同步异常。**

在编写挂起函数时永远不要使用throw(只能在协程所调度的线程捕获到)，当需要抛出异常时必须调用续体的resumeWithException(e)，否则会导致协程被永远的挂起。

当一个协程抛出未捕获的异常时，会取消自己及其所有子协程。**如果子协程抛出了未捕获的非`CancellationException`类型的异常（`CancellationException`类型的异常会被父协程忽略），这个异常对象会被传递到根协程处理，并会取消协程及所有其他子级**

**协程异常传递**

`BaseContinuationImpl.resumeWith()中try住异常对象exception -> AbstractCoroutine.resumeWith(Result.failure(exception)) -> JobSupport.makeCompletingOnce(result.toState()) ->JobSupport.tryMakeCompleting(state, proposedUpdate)-> JobSupport.tryMakeCompletingSlowPath(state, proposedUpdate)`：

```kotlin
//kotlinx.coroutines.JobSupport
private fun tryMakeCompletingSlowPath(state: Incomplete, proposedUpdate: Any?): Any? {
    ...
    //根据state获取到异常对象，这个异常对象就是最初BaseContinuationImpl.resumeWith()中传递过来的
    var notifyRootCause: Throwable? = ...
    //★ 1. 当异常对象不为空时，触发协程cancel取消，但并不会立马将当前协程状态置为canceled，需要等待该协程下所有子协程都取消完毕后才会修改当前协程状态为已取消
    notifyRootCause?.let { notifyCancelling(list, it) }
    val child = firstChild(state)
    // ★★★ 6. 等待所有子协程完成或者取消
    if (child != null && tryWaitForChild(finishing, child, proposedUpdate))
    return COMPLETING_WAITING_CHILDREN
    return finalizeFinishingState(finishing, proposedUpdate)
}
```

当异常对象被传递给协程后，会触发`notifyCancelling()`函数取消当前协程，而`notifyCancelling()`中做了两件事。首先调用`notifyHandlers()`遍历子协程节点取消子协程，然后调用`cancelParent()`取消父协程。

![异常取消传递情况](https://img-blog.csdnimg.cn/20210523203428597.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAxNjM0NDI=,size_16,color_FFFFFF,t_70#pic_center)

```kotlin
//等待所有子协程完成或者取消 
private tailrec fun tryWaitForChild(state: Finishing, child: ChildHandleNode, proposedUpdate: Any?): Boolean {
     val handle = child.childJob.invokeOnCompletion(
         invokeImmediately = false,
         //★ 7. 调用ChildCompletion()传入this表示父协程，child表示父协程的某一个子协程，这个方法确保child的所有子协程都完成或者取消，意思就是确保child这个子树分支都已经完成或者取消
         handler = ChildCompletion(this, state, child, proposedUpdate).asHandler
     )
     //如果这个子协程树未完成或者未取消，就返回（等待其完成）
     if (handle !== NonDisposableHandle) return true 
     //★ 8. 如果这个子协程树已经完成或者取消，则获取获取下一个子协程(遍历所有子协程)，递归调用tryWaitForChild()，直到发现有一个子协程还没有完成或者取消
     val nextChild = child.nextChild() ?: return false
     return tryWaitForChild(state, nextChild, proposedUpdate)
 }
```

以上代码是异常的取消传播：

- 获取当前协程的第一个子协程 child，调用`ChildCompletion()`确保这个子协程及其下的孙子协程都被取消；
- 然后通过`child.nextChild()`获取下一个子协程，递归调用`tryWaitForChild()`确保当前协程下的所有子协程树都被取消。

协程异常传递和处理机制：

```kotlin
private fun finalizeFinishingState(state: JobSupport.Finishing, proposedUpdate: Any?): Any? {
    ...
    assert { state.isCompleting } //★★★尝试：一致性检查--必须标记为已完成(canceled状态也是已完成)
    
    // proposedException就是未捕获的异常对象
    val proposedException = (proposedUpdate as? CompletedExceptionally)?.cause
    var wasCancelling = false
    // 当协程的多个子协程因异常而失败时， 一般规则是“取第一个异常”，因此将处理第一个异常。 在第一个异常之后发生的所有其他异常都作为被抑制的异常绑定至第一个异常
    val finalException = kotlinx.coroutines.internal.synchronized(state) { ... }
    ...
    // ★★★ 10. cancelParent(finalException) 取消父协程，这个过程上面已经分析过了，其实这里调用该方法并不是为了取消父协程，而是将异常对象交给父协程处理
    // 如果该方法返回true，则表示这个异常对象被父协程处理掉了，如果返回false表示父协程没有处理这个异常，则会调用handleJobException()让当前协程处理该异常
    if (finalException != null) {
        val handled = cancelParent(finalException) || handleJobException(finalException)
        if (handled) (finalState as CompletedExceptionally).makeHandled()
    }
    // 收尾工作设置协程状态为已完成、已取消 job.isCompleted == true  job.isCancelled == true
    if (!wasCancelling) onCancelling(finalException)
    onCompletionInternal(finalState)
    ...
    completeStateFinalization(state, finalState)
    return finalState
}
```
- 如果异常是CancellationException类型，cancelParent()会直接返回true，CancellationException类型的异常没有任何处理直接被忽略了；
- 如果是非CancellationException异常，如果父协程是SupervisorJob类型，cancelParent()返回false，将调用handleJobException()由当前协程处理异常；
- 如果是非是非CancellationException异常，并且协程树中不存在SupervisorJob类型，异常对象最终将由根协程处理。

> 默认的handleJobException()函数对异常不做任何处理，只在StandaloneCoroutine和ActorCoroutine两个协程类重写了该函数并调用了handleCoroutineException()进一步处理异常，launch{}构建的协程是StandaloneCoroutine类型的，actor{}构建的协程是ActorCoroutine类型的。

![协程异常传递和处理机制](https://img-blog.csdnimg.cn/20210523203436823.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAxNjM0NDI=,size_16,color_FFFFFF,t_70#pic_center)

总结：

当挂起函数resumeWithException(e)或者协程代码块{}中throw了一个异常，这个异常会被捕获处理，并将异常传递给对应的协程对象，协程对象收到异常后会根据情况在协程树中传递取消相关协程（子传递父，父取消子）。如果异常类型为CancellationException类型，当传递取消父协程时，父协程直接返回true表示忽略（不会取消父协程），当父协程的job为SupervisorJob 类型时，不管是什么异常都会终止向上传递取消。

异常对象最终会交给协程树中的某个协程处理：CancellationException类型的异常由抛出异常的协程自己处理；某一层协程对象Job为SupervisorJob 类型时由该层的下级子协程处理（抛异常的协程或者父协程，但是都是SupervisorJob层的子协程）；其他情况由根协程处理。

#### 协程调度器实现原理

协程的三层包装：

![协程的三层包装](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%8D%8F%E7%A8%8B%E7%9A%84%E4%B8%89%E5%B1%82%E5%8C%85%E8%A3%85.drawio.png)

在 kotlin 协程中，协程的执行涉及到 3 次线程切换：

- 切换到指定线程执行协程代码块中的代码；
- 当协程代码块调用异步挂起函数时，切换到指定线程执行挂起函数；
- 当异步挂起函数执行完毕，将函数执行结果 Result 对象切换到协程所在的线程，继续执行协程代码块中剩下的代码。

`ContinuationImpl$intercepted()`续体拦截器：将原始续体对象包装为另一种续体类型的对象，增强原续体的功能。

`Dispatchers`调度器：

![调度器相关类图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%BB%AD%E4%BD%93%E6%8B%A6%E6%88%AA%E5%99%A8.drawio.png)

第一次调度：

```kotlin
@ExperimentalCoroutinesApi
public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
    val combined = coroutineContext + context
    val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
    return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
        debug + Dispatchers.Default else debug // 保证上下文有调度器。
}
```
![第一次调度](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%8D%8F%E7%A8%8B%E5%88%87%E6%8D%A2%E7%BA%BF%E7%A8%8B1.drawio.png)

第二次调度：切换线程执行异步挂起函数

a. new Thread 等直接切换线程

b. `withContext(Dispatchers.IO){}`

```kotlin
public suspend fun <T> withContext(
    context: CoroutineContext,       //协程上下文，一般情况下传递一个调度器
    block: suspend CoroutineScope.() -> T   //子协程代码块
): T {
    ...
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
        val oldContext = uCont.context         //父协程的上下文
        val newContext = oldContext + context  //父协程上下文+新的调度器
        ...
        if (newContext === oldContext) {
            //新的上下文和父协程上下文地址相同，不需要切换线程，将创建一个ScopeCoroutine类型的子协程
            val coroutine = ScopeCoroutine(newContext, uCont)
            return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
        }
        // 如果传入的调度器和父协程调度器相同，不需要切换线程，将创建一个UndispatchedCoroutine类型（ScopeCoroutine子类）的子协程
        if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {
            val coroutine = UndispatchedCoroutine(newContext, uCont)
            withCoroutineContext(newContext, null) {
                return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
            }
        }
        //★上面两种不需要切换线程的就没必要跟踪了。当调度器不同时创建一个DispatchedCoroutine类型的子协程，切换到新线程执行子协程代码块（也就是函数体）
        val coroutine = DispatchedCoroutine(newContext, uCont)
        coroutine.initParentJob()
        //★★★ 接2.1的startCoroutineCancellable()，创建子SuspendLambda续体对象，然后包装为DispatchedContinuation类型，最后dispatcher.dispatch()切换到新线程开始执行函数体
        block.startCoroutineCancellable(coroutine, coroutine)   
        coroutine.getResult()
    }
}
```

第三次调度：异步挂起函数执行完毕后将函数执行结果切回协程所在线程并恢复执行。

a. 线程池或 Thread 的方式执行之后，会调用续体`continuation`的`resumeWith`，也就是`DispatchedContinuation`的`resumeWith()`，对应判断是否需要切换线程，需要则进行切换。

b. 协程代码块的代码会通过状态机被分为多个执行部分放入的SuspendLambda.invokeSuspend()函数中，invokeSuspend()又是在子协程的续体BaseContinuationImpl.resumeWith()中调用的，所以子协程代码块的返回值为最后一次调用invokeSuspend()函数的返回值，在BaseContinuationImpl.resumeWith()函数中这个返回值将被传递给子协程的AbstractCoroutine.resumeWith()函数作为子协程的返回值。
**withContext()创建的子协程DispatchedCoroutine会将父协程的续体的intercepted()函数再次包装为DispatchedContinuation类型，然后调用resumeCancellableWith()切换线程，将挂起函数的结果返回并恢复父协程执行**。

> kotlin 协程库 4 种调度器：
>
> - Dispatchers.Main：在带有UI模块的系统中，表示在**UI线程调度**；
> - Dispatchers.Default：通常情况下会采用CoroutineScheduler类型的线程池，这个线程池使用特殊的**任务抢占式**的Worker类型的线程，避免线程切换造成CPU资源消耗。但是如果设置了系统属性kotlinx.coroutines.scheduler的值为off时，将采用java的ForkJoinPool线程池，它是一种任务拆分式线程池，可以将一个大任务拆分为多个小任务后给多个线程并发执行，然后汇总结果，从而提高了CPU利用率；
> - Dispatchers.IO：它和Dispatchers.Default一样采用的是CoroutineScheduler任务抢夺式线程池，区别是在往线程池扔任务之前多了一个队列，用于控制最大并发任务数量；
> - Dispatchers.Unconfined：无限制的，**在任何情况下都不需要切换线程，直接在当前线程执行**，如果异步挂起函数执行完毕后恢复协程执行，协程将沿用挂起函数的线程上执行。