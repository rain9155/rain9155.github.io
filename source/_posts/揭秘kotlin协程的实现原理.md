---
title: 揭秘kotlin协程的实现原理
date: 2022-05-26 02:56:26
tags: 协程
categories: kotlin
---

## 前言

- 上一篇文章：[揭秘kotlin协程中的CoroutineContext](https://juejin.cn/post/6926695962354122765#heading-0)

上一篇文章中介绍了kotlin协程的CoroutineContext的主要组成以及它的结构，kotlin协程的CoroutineContext它是一个K-V数据结构，保存了跟协程相关联的运行上下文例如协程的线程调度策略、异常处理逻辑、日志记录、运行标识、名字等，本篇文章是作为上一篇文章的补充，在使用kotlin协程一年多之后，对kotlin协程的实现有了新的认识，本文会深入介绍kotlin协程的实现原理，例如Continuation和CPS，suspend方法的含义以及背后的原理，协程是如何被创建、启动、调度，同时使用kotlin-stdlib提供的intrinsics原语实现一个简化版的协程，从而帮助我们更好地理解kotlin协程的整个设计思想，kotlin协程的源码被放在了两个库中，一部分是在kotlin标准库[kotlin-stdlib](https://github.com/JetBrains/kotlin/tree/1.4.0/libraries/stdlib/src/kotlin/coroutines)中，一部分是在kotlin协程官方实现库[kotlinx-coroutines](https://github.com/Kotlin/kotlinx.coroutines/tree/native-mt-1.4.20/kotlinx-coroutines-core)中，其中kotlinx-coroutines是基于kotlin-stdlib的，kotlin-stdlib库提供了实现协程所需的基本原语。

> 本文涉及到的源码都是基于kotlin1.4版本

## Continuation和CPS

在讲解协程的原理之前，我们先来了解一下Continuation和CPS，理解了这两个术语，那么后面对于协程的理解就非常容易了：

**Continuation**

[Continuation](https://en.wikipedia.org/wiki/Continuation)延续在计算机中表示**程序剩余的部分**，它保存了程序从某一点开始的执行状态，并能够在稍后的时间让程序回到这一点恢复执行，所以它是一种能够保存程序执行状态的数据结构，像break、continue这类控制流操作符一样可以暴露给用户使用，用户通过操作Continuation来控制程序的执行顺序，Continuation的概念在上个世纪五、六十年代就被提出来，首次实现Continuation的编程语言是上个世纪70年代的[Scheme](https://en.wikipedia.org/wiki/Scheme_(programming_language))语言，在Scheme语言中它引入了**call/cc**关键字 - [call-with-current-continuation](https://en.wikipedia.org/wiki/Call-with-current-continuation)，通过call/cc关键字我们可以捕获程序当前剩余的执行状态保存到Continuation中，并在之后适当的时候执行Continuation以恢复到捕获Continuation时所在的上下文继续执行，由于我不熟悉Schema语言，这里我用kotlin来模拟这个关键字，**假设**kotlin有call/cc关键字，它是这样使用：

```kotlin
fun main(){
    //pause
    val result = call/cc { continuation ->
        //do something
        continuation("world")
        //ignore
        return "world!"
    }
    //resume
    println("hello $result")
}

//运行输出：
//hello world
```

call/cc接收一个带有一个参数的函数，这个参数就是current-continuation表示程序剩余的部分，当程序运行到call/cc时，它会暂停程序后续的执行并捕获程序当前剩余的部分作为参数传进call/cc接收的函数中，然后执行这个函数，然后在适当的时候我们可以调用continuation，continuation接收一个参数作为call/cc的返回值，一旦我们调用continuation后，函数后面的部分就不会继续执行而是返回到call/cc调用处继续执行，而如果我们不调用continuation，那么函数就会正常执行完毕返回，这时call/cc的返回值就是函数的返回值，由于我们这里调用了continuation，所以这里程序恢复后输出了"hello world"，这就是使用call/cc应用Continuation的一个简单例子，通过Continuation我们还可以实现更为复杂的场景例如[异常处理](https://en.wikipedia.org/wiki/Exception_handling)，这里继续通过call/cc实现一个异常处理try catch能力：

```kotlin
fun main(){
    tryCatch({
         //do something,
         throwException(IllegalAccessException("something error"))
    }, { e: Exception ->
        //continue
        println("catch $e")
    })
  	println("finish")
}

val continuationStack = Stack<Continuation>()

fun tryCatch(tryBlock: () -> Unit, catchBlock: (e: Exception) -> Unit){
    val result = call/cc { continuation ->
        funStack.add(continuation)
        return tryBlock()
    }
    funStack.pop()
    if(result is Exception) {
        catchBlock(result)
    }
}

fun throwException(e: Exception) {
    if(continuationStack.size > 0) {
        val continuation = continuationStack.peek()
        continuation(e)
    }
}

//运行输出：
//catch IllegalAccessException: something error
//finish
```

tryCatch方法接收两个函数：一个是正常代码执行主体tryBlock，一个是异常处理执行主体catchBlock，每次在执行tryBlock前都会把当前捕获的延续Continuation压入栈中，然后每次调用throwException方法抛出异常时都会弹出最近的Continuation传入异常恢复外部执行，如果tryBlock正常返回即没有调用throwException方法，这时call/cc的返回值是一个Unit类型，如果tryBlock出现异常即调用了throwException方法抛出异常，那么这时call/cc的返回值是一个Exception类型，这时就调用catchBlock处理异常，这样就通过Continuation实现了一个简单的try catch能力，这里我们也可以看到Continuation的作用，可以让我们灵活的控制程序的执行，除了异常处理，Continuation也可以被运用来实现[协程](https://en.wikipedia.org/wiki/Coroutine#Mutual_recursion)、[生成器](https://en.wikipedia.org/wiki/Generator_(computer_programming))等，后面我们就会看到kotlin协程的实现原理。

**CPS**

介绍完Continuation，继续来了解一下[CPS](https://en.wikipedia.org/wiki/Continuation-passing_style)即Continuation-passing Style延续传递风格，它是Continuation在[函数式编程](https://en.wikipedia.org/wiki/Functional_programming)中的应用，像一些支持函数式编程的编程语言例如scheme、kotlin、js、py、C#等都可以把它们的函数转化为CPS风格，CPS风格的函数有以下特点：

1、函数没有return语句；

2、函数都有一个额外的Continuation参数；

3、函数内对于Continuation的传递调用都是[尾调用](https://en.wikipedia.org/wiki/Tail_call)。

先看一个普通的函数的调用：

```kotlin
fun main(){
    val result = add(1, 1)
    println("$result")
}

fun add(a: Int, b: Int): Int {
    return a + b
}
```

上面定义了一个add方法，调用它返回两个参数相加的结果，接下把它翻译成CPS函数：

```kotlin
fun main(){
    add(1, 1) { result ->  
        println("$result")
    }
}

fun add(a: Int, b: Int, continuation: (result: Int) -> Unit) {
    continuation(a + b)
}
```

可以看到CPS风格的add方法与普通的add方法多了一个continuation参数，用来表示外部的控制流，当方法需要返回时，就调用传进来的continuation代替return语句，当调用传进来的continuation后，外部代码的逻辑就继续执行。

下面再看一个嵌套的函数调用：

```kotlin
fun main(){
    val result = squareAdd(1, 1)
    println("$result")
}

fun squareAdd(a: Int, b: Int): Int {
    return add(square(a), square(b))
}

fun add(a: Int, b: Int): Int {
    return a + b
}

fun square(c: Int): Int {
    return c * c
}
```

上面定义了一个squareAdd方法，调用它返回两个平方的相加结果，把它翻译成CPS函数：

```kotlin
fun main(){
    squareAdd(1, 1) { result ->  
        println("$result")
    }
}

fun squareAdd(a: Int, b: Int,  continuation: (result: Int) -> Unit) {
    square(a) { aSquareResult ->
        square(b) { bSquareResult ->
            add(aSquareResult, bSquareResult) { abAddResult ->
                continuation(abAddResult)
            }
        }
    }
}

fun add(a: Int, b: Int, continuation: (result: Int) -> Unit) {
    continuation(a + b)
}

fun square(c: Int, continuation: (result: Int) -> Unit) {
    continuation(c * c)
}
```

可以看到CPS风格的squareAdd方法里面不断的嵌套调用其他方法，并且调用其他方法传递Continuation时都是尾调用，尾调用就是在函数的末尾调用了另外一个函数而没有做其他操作，相应地，如果在函数的末尾调用地是函数本身，那么这就叫做尾递归，每个CPS风格的方法就是这样不断地在尾部调用其他方法并把自己当前的延续Continuation传递给调用的方法，这就是**Continuation-passing延续传递**名字的由来，从本质讲，CPS方法就是一个回调函数，Continuation相当于一个回调，每个CPS方法只能通过Continuation回调来恢复程序的后续逻辑执行，随着代码的复杂度提升，方法的调用数变多，CPS方法的嵌套深度也会越来越深，代码的可读性也会越来越差，出现回调地狱callback-hell现象，同时如果编译器不支持尾调用优化，那么CPS方法很容易就出现栈溢出错误。

> 如果编译器支持，尾调用和尾递归都可以进行优化，尾调用由于不需要依赖调用方，所以调用方函数的栈帧可以直接被尾调用函数的栈帧代替，如果所有函数都是都是尾调用，那么调用方就可以直接goto到最深处调用的函数，减少调用栈帧从而避免了栈溢出，同时减少了栈帧的内存消耗，这就是尾调用优化，而尾递归除了可以应用尾调用优化外，它还有自己特属的优化方法，由于尾递归的特殊性，我们可以把一个尾递归函数展开为一个循环调用，这样也减少了调用栈帧和内存消耗，这就是尾递归优化，kotlin中可以通过**tailrec**修饰符让编译器对一个尾递归函数进行优化，不管是尾调用优化和还是尾递归优化，它们都改变了原本函数的调用栈帧，所以会让debug变得困难，这也是为什么支持尾调用和尾递归优化的编译器不默认打开这个选项的原因。

那么CPS存在的意义是什么？其实CPS方法主要是作为高级语言的一种中间表示IR，把高级语言的方法逻辑编译成CPS风格，可以大大地减少编译器的实现复杂度，当程序被编译成CPS时，方法会被划分成不可再分割的最小粒度例如基本的运算、基本的方法调用等，例如 1 + 2 + 3 * 4 的计算翻译成CPS风格：

```kotlin
fun main(){
   calculate {
       println(it)
   }
}

fun calculate(continuation: (result: Int) -> Unit) {
    { cont1: (Int) -> Unit ->
        cont1(3 * 4)
    }({ mul: Int ->
        { cont2: (Int) -> Unit ->
            cont2(2 + mul)
        }({ add: Int ->
            { cont3: (Int) -> Unit ->
                cont3(1 + add)
            }({ result: Int ->
                continuation(result)
            })
        })
    })
}
```

可以看到每一条基本的计算语句(+、*)都会被包含在一个函数中，每个函数只负责基本的运算，然后原函数剩余的部分被包装在Continuation中，这对于用户来说可能比较难以阅读，但对于编译器来说这会让程序的语法分析更加简单，同时CPS所有的控制流例如if else、try catch等都会通过Continuation显式表示出来，这时编译器可以直接进行控制流分析，同时在CPS的基础上还可以进行尾调用优化等手段，如果对CPS这些编译优化感兴趣的可以阅读下面链接：

[Compiling CPS](https://www.gnu.org/software/guile/manual/html_node/Compiling-CPS.html)

[What optimizations CPS transformations enables / disables](https://www.reddit.com/r/haskell/comments/d5tutb/what_optimizations_cps_transformations_enables/)

CPS除了应用在编译器中，还可以应用在异步编程中，异步编程就是我们以不阻塞当前线程的方式来获取一个耗时操作的执行结果，例如网络请求、IO读取等，在Android中一般通过callback实现异步编程，但是通过callback进行异步编程是很困难，因为程序的逻辑被分散到各个callback，程序的连续性被打破，同时当每个callback相互依赖时就会出现callback-hell，让代码可读性降低，我们还需要额外去维护每一个callback，前面讲过CPS方法本质上是一个callback方法，所以通过CPS方法也可以处理异步编程的场景，由于CPS方法遵循一定的规则，所有编程语言就很容易替我们完成CPS转换和Continuation管理，不用我们编写复杂的CPS代码，例如js、c#中的async/await、kotlin中的suspend关键字等，这些都是语法糖，通过这些关键字修饰的一些方法都会有CPS转换的过程，可以让我们像编写同步代码那样编写异步代码，可以在一定的范围内保持程序的连续性，例如下面login和fetchData都是异步方法，fetchData方法依赖login方法，displayUI方法依赖fetchData方法：

```kotlin
//js
async function display() {
  var user = await login(); //async方法
  var data = await fetchData(user); //async方法
  displayUI(data);
}

//c#
async void display() {
  var user = await login(); //async方法
  var data = await fetchData(user); //async方法
  displayUI(data);
}


//kotlin
suspend fun display() {
   val user = login() //suspend方法
   val userData = fetchData(user) //suspend方法
   displayUI(userData)
}
```

即使login和fetchData方法是异步的，但是上面的整个运行过程都是线性的，每一个方法都会等前一个方法返回后再继续执行，这就是Continuation和CPS在异步编程中的应用，对方法进行CPS转换时，首先要进行call/cc处理即捕获当前延续Continuation，然后还要处理不同Continuation之间的流转，实现暂停和恢复，每个编程语言对于这些实现是不一样，主流的有**生成器**和**状态机**两种实现方式，js中通过[生成器](https://dev.to/yelouafi/algebraic-effects-in-javascript-part-1---continuations-and-control-transfer-3g88)实现，而c#和kotlin则是通过[状态机](https://en.wikipedia.org/wiki/Finite-state_machine)的方式实现，得益于编程语言的良好封装，我们通过这些语法糖编写异步代码时不用再去维护每一个callback，不用再考虑这些复杂的处理。

## suspend方法的实现

通过前面的介绍，相信大家已经猜到kotlin suspend方法的实现原理，suspend就是一个语法糖，当我们用suspend修饰命名方法或者匿名、lambda方法时，kotlin编译器会替我们把suspend方法进行CPS转换，转化后的方法会多一个额外的名为completion的Continuation类型参数，原本的返回值类型会移动到Continuation的类型参数中，并且把返回值用Any类型表示，例如命名方法：

```kotlin
suspend fun login(): String
```

转化为：

```kotlin
fun login(completion: Continuation<String>): Any?
```

再例如lambda方法：

```kotlin
suspend () -> String
```

转化为：

```kotlin
(completion: Continuation<String>) -> Any?
```

其实kotlin中的每个lambda方法都会对应一个[Function](https://github.com/JetBrains/kotlin/blob/1.4.0/libraries/stdlib/jvm/runtime/kotlin/jvm/functions/Functions.kt)类型, 如果lambda方法没有参数就对应Function0类型，如果lambda方法有一个参数就对应Function1类型，以此类推，每个Function类都有一个invoke方法，invoke方法的参数就对应lambda方法的参数，invoke方法的返回值就对应lambda方法的返回值，调用Function类实例的invoke方法就相当于调用对应的lambda方法，所以上面CPS转化后的lambda方法在kotlin中实际表示为：

```kotlin
//XXX就是lambda方法对应类的名称，会根据所在类、所在方法用$符号拼接而成
//这里是简化版，实际情况还会实现一个SuspendLambda抽象类，SuspendLambda继承自ContinuationImpl，后面会讲到
class XXX : Function1<Continuation<String>, Any?> {

    override fun invoke(completion: Continuation<String>): Any? {
        //...
    }
}
```

suspend方法CPS后的方法都返回值用一个Any类型表示，它是**T | COROUTINE_SUSPENDED**的组合类型，T表示suspend方法同步执行返回时的类型，例如这里返回为String类型，当suspend方法不需要挂起时，suspend方法就正常返回对应的值或者抛出异常，COROUTINE_SUSPENDED表示suspend方法需要挂起时返回的一个枚举类型，当suspend方法需要挂起时，suspend方法就返回COROUTINE_SUSPENDED表示这个suspend方法被挂起，最终真正执行挂起动作返回COROUTINE_SUSPENDED的地方是kotlin intrinsics提供的**suspendCoroutineUninterceptedOrReturn**方法，这个方法可以捕获传递过来的Continuation，然后决定是否挂起，kotlin协程库提供的一些封装好的挂起方法如withContext、delay、await等最终都是调用这个方法捕获Continuation和执行挂起动作，我们编写suspend方法时也可以直接使用这个方法，实现我们自己的挂起逻辑，后面在intrinsics中会介绍这个方法。

当一个suspend方法被挂起，说明这个suspend方法不能马上同步返回对应的结果，而是在稍后准备好时再通过调用Continuation的**resumeWith**方法从挂起点恢复返回结果，Continuation在kotlin中是一个接口：

```kotlin
//T为suspend方法的返回值类型
public interface Continuation<in T> {
  
    //当前延续的上下文
    public val context: CoroutineContext

    //当需要从挂起点恢复时调用这个方法，result可以表示正常恢复还是异常恢复
    public fun resumeWith(result: Result<T>)
}
```

挂起点suspend point就是调用suspend方法的地方，当我们在suspend方法中调用suspend方法时，每一个suspend方法的调用处就是一个挂起点，整个suspend方法被挂起点分割成多个部分，每一个部分都对应一个Continuation，例如：

```kotlin
suspend fun display() {     
    val user = login() //suspend方法，挂起点1     
    val data = fetchData(user) //suspend方法，挂起点2   
    displayUI(data) //普通方法                         
}

suspend fun login(): String {
  	delay(200) //suspend方法，延迟200ms后返回
  	return "user"
}

suspend fun fetchData(user: String): String {
    return "$user data"
}

fun displayUI(data: String) {
    println("displayUI: $data")
}
```

根据挂起点划分，上面display方法有三个延续Continuation：

1、初始Continuation，整个display方法就是一个Continuation；

2、子Continuation1，挂起点1到display方法结尾；

3、子Continuation2，挂起点2到display方法结尾。

类似地，login方法有两个延续，kotlin并不会像传统的CPS处理那样为每一个Continuation创建对应的实例，kotlin只会为整个suspend方法创建一个初始Continuation实例，然后在这个Continuation实例内部通过**状态机**进行流转，每个挂起点对应状态机中的一个状态，通过状态机就可以复用一个Continuation实例就能达到在多个挂起点之间进行挂起和恢复的效果，减少了Continuation实例的创建数量，下面是display方法的对应实现，是经过简化后的版本：

```kotlin
fun display(completion: Continuation<Unit>): Any? {

    //display方法的状态机，继承自ContinuationImpl
    class DisplayStateMachine(
        //completion是调用display方法时传递进来的，当display执行完毕时，通过completion恢复外部执行
        completion: Continuation<Unit>
    ) : ContinuationImpl(completion) {

        //保存每个挂起点恢复后的结果
        var result: Result<Any?> = null

        //当前display方法的状态
        var label: Int = 0

        //当invokeSuspend被调用时，再次调用display方法，这时result会是前一个状态的结果，而label也已处于将要执行的状态
        override fun invokeSuspend(result: Result<Any?>): Any? {
            this.result = result
            return display(this)
        }
    }

    //如果是第一次调用display方法，就创建状态机实例，如果不是第一次调用，继续执行状态
    val continuation = completion as? DisplayStateMachine ?: DisplayStateMachine(completion)

    //COROUTINE_SUSPENDED标记，用于判断是否挂起
    val val0 = COROUTINE_SUSPENDED

    when(continuation.label) {
        0 -> {
            //错误检查
            continuation.result?.getOrThrow()
            //下次display方法被调用时, 它应当直接去到状态1
            continuation.label = 1
            //调用login方法，传入continuation
            val val1 = login(continuation)
            //判断是否挂起
            if(val1 == val0) {
                //如果挂起，直接return，后面通过传进login方法的continuation恢复当前状态机执行
                return val0
            }
            //如果没有挂起，继续执行，下面流程跟label1类似
            val user = val1 as String
            continuation.label = 2
            val val2 = fetchData(user, continuation)
            if(val2 == val0) {
                return val0
            }
            val data = val2 as String
            continuation.label = -1
            displayUI(data)
            return Unit
        }
        1 -> {
            //错误检查, 并获取前一个状态的结果
            val user = continuation.result?.getOrThrow() as String
          	//下次display方法被调用时, 它应当直接去到状态2
            continuation.label = 2
            //调用fetchData方法，传入continuation
            val val2 = fetchData(user, continuation)
            //判断是否挂起
            if(val2 == val0) {
                //如果挂起，直接return，后面通过传进fetchData方法的continuation恢复当前状态机执行
                return val0
            }
            //如果没有挂起，继续执行，下面流程跟label2类似
            val data = val2 as String
            continuation.label = -1
            displayUI(data)
            return Unit
        }
        2 -> {
          	//错误检查, 并获取前一个状态的结果
            val data = continuation.result?.getOrThrow() as String
            //display方法执行完毕，把label置为非法状态
            continuation.label = -1
            displayUI(data)
            return Unit
        }
        else -> {
            throw IllegalStateException("call to 'resume' before 'invoke' with coroutine")
        }
    }
}

fun login(completion: Continuation<String>): Any? {
    //login方法的状态机, 跟display方法的类似
    class LoginStateMachine(
        //completion是调用login方法时传递进来的，当login执行完毕时，通过completion恢复外部执行
        completion: Continuation<Unit>
    ) : ContinuationImpl(completion) {
      
        var result: Result<Any?> = null
        var label: Int = 0

        override fun invokeSuspend(result: Result<Any?>): Any? {
            this.result = result
            return login(this)
        }
    }
    val continuation = completion as? LoginStateMachine ?: LoginStateMachine(completion)
    val val0 = COROUTINE_SUSPENDED
    when(continuation.label) {
        0 -> {
            continuation.result?.getOrThrow()
            continuation.label = 1
            val val1 = delay(continuation)
            if(val1 == val0) {
                return val0
            }
            return "user"
        }
        1 -> {
            continuation.result?.getOrThrow()
            continuation.label = -1
            return "user"
        }
        else -> {
            throw IllegalStateException("call to 'resume' before 'invoke' with coroutine")
        }
    }
}

fun fetchData(user: String, completion: Continuation<String>): Any? {
    return "$user data"
}

fun displayUI(data: String) {
    println("displayUI: $data")
}
```

可以看到display和login方法都创建了对应的状态机，每个延续都对应状态机的一个状态，每个状态机都继承自**ContinuationImpl**，ContinuationImpl的父类是**BaseContinuationImpl**，它实现了**Continuation**的resumeWith方法并且含有一个invokeSuspend抽象方法，所以每个状态机都会实现这个invokeSuspend方法，并且每个状态机都会持有一个外部的完成延续Continuation，用来在当前状态机运行结束时恢复外部的Continuation，关于Continuation的恢复后面会讲。

当suspend方法第一次被调用时就会创建一个状态机，这时状态机是初始状态，对应执行初始延续的逻辑，每执行完一个状态，都会把状态机的状态提前置为下一个状态，当要执行下一个状态时，只需要再次调用suspend方法就行，而这个再次调用就由**invokeSuspend**方法来完成，invokeSuspend方法中会调用suspend方法进行**状态流转**，而invokeSuspend方法会被**resumeWith方**法调用，而Continuation的resumeWith方法什么时候调用，就是由我们自己决定的，因为最终suspend方法的状态机Continuation会被传递到kotlin intrinsics提供的**suspendCoroutineUninterceptedOrReturn**方法中，在这个方法中我们可以捕获到这个Continuation，并决定什么时候调用这个Continuation到resumeWith方法。

上面的display和login方法都是命名suspend方法，对于suspend lambda方法，kotlin编译器也会对它进行CPS转换并且创建状态机，不同的是suspend lambda方法的状态机是继承自**SuspendLambda**类，而SuspendLambda是ContinuationImpl的子类，例如我们通过launch方法传递**block块**启动协程：

```kotlin
fun main() {
   GlobalScope.launch {//suspend lambda方法
      	val user = login() //suspend方法，挂起点1     
    		val data = fetchData(user) //suspend方法，挂起点2   
    		displayUI(data) //普通方法    
    }
}

public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    //...
}
```

这个block块就是一个suspend lambda方法，所以kotlin编译器会为这个suspend lambda方法创建一个实现了Function接口、继承自SuspendLambda的类，并且这个类同时也实现了状态机，如下：

```kotlin
fun main(){
    GlobalScope.launch(EmptyCoroutineContext, CoroutineStart.DEFAULT, SuspendLambdaStateMachine(null))
}

class SuspendLambdaStateMachine(completion: Continuation<Unit>) : SuspendLambda(2, completion), Function2<CoroutineScope, Continuation<Unit>, Any?> {

   	//保存每个挂起点恢复后的结果
    var result: Result<Any?> = null
  	//当前lambda方法的状态
  	var label: Int = 0

    //invoke方法被调用相当于lambda方法被执行，在里面它会创建状态机，并执行状态机的invokeSuspend方法
    override fun invoke(p1: CoroutineScope, completion: Continuation<Unit>): Any? {
        return (create(completion) as SuspendLambdaStateMachine).invokeSuspend(Result.success(Unit))
    }
    
    //调用create方法可以传入一个完成延续创建suspend lambda方法的状态机
    override fun create(completion: Continuation<*>): Continuation<Unit> {
        return SuspendLambdaStateMachine(completion)
    }

	  //调用invokeSuspend方法进行状态流转，这时result会是前一个状态的结果，而label也已处于将要执行的状态
    override fun invokeSuspend(result: Result<Any?>): Any? {
        this.result = result
        when(label) {
            //...跟前面display方法类似, 这个
        }
    }
}
```

suspend lambda方法的状态机会比suspend命名方法多实现一个**create**方法，这个create方法也来自于父类BaseContinuationImpl中，这个create方法的目的就是创建一个suspend lambda方法的状态机实例并传入它的完成延续，这个方法最终会被kotlin intrinsice方法的**createCoroutineUnintercepted**方法调用。

通过前面介绍的suspend命名方法和suspend lambda方法实现可以看出，kotlin编译器为每个suspend方法做了以下几件事：

- 1、为含有挂起点、且挂起点不是尾调用的suspend方法创建一个私有的状态机；

- 2、状态机中通过变量保存了suspend方法将要执行的状态和上一个状态的结果，每一次执行状态前，为了防止挂起函数运行失败都会进行状态检查，并且调用挂起函数前，状态机的状态都会提前置为下一个状态；

- 3、调用其他挂起函数时，都会把当前状态机实例作为Continuation传递过去，而被调用的挂起函数满足1条件时也会被创建一个状态机，当被调用挂起函数的状态机运行结束时，可以利用传递过去的Continuation恢复当前状态机执行。

从第1点可以看出，kotlin编译器并不总是为suspend方法创建状态机，例如fetchData方法，虽然它是suspend方法，但是它里面没有调用其他的suspend方法，并不需要处理状态，所以kotlin编译器不会为它创建状态机，还有一种情况kotlin编译器也不会创建状态机，就是如果suspend方法中只有一个suspend方法调用并且这个suspend方法调用是尾调用，那么kotlin编译器不会为它创建状态机，只会简单地把suspend方法的Continuation实例继续传递给尾调用的suspend方法，因为在尾调用中，调用方不需要保存状态，所以总结起来就是kotlin编译器只会为**含有非尾部suspend方法调用**的suspend方法创建状态机，这是kotlin编译器的一个优化，避免创建多余的状态机实例。

> kotlin官方对于suspend方法还提出了另外一个优化，就是只有当首次挂起时才进行状态机的创建，即状态机懒创建，因为在首次挂起前，suspend方法很有可能因为其他原因提前退出了，这时提前创建的状态机就是多余的。

从第2点可以看出，suspend方法的挂起和恢复是通过状态机的状态切换来实现的，每个状态对应suspend方法的每个延续，状态机保存了每个延续恢复后的结果，从第3点可以看出，suspend方法的状态机实例会作为Continuation在suspend方法之间传递，而最终这个状态机实例Continuation会传到**suspendCoroutineUninterceptedOrReturn**方法中暴露给我们使用，在这个方法里我们可以控制这个Continuation例如包装它在恢复前作出一些我们的自定义行为，并决定什么时候进行恢复，当我们决定恢复时，就调用resumeWith方法就行。

## intrinsics方法

kotlin intrinsics方法是用来实现协程的基本原语，前面已经讲过协程的实现原理是Continuation，Continuation的最主要的好处就是可以暴露给用户用于控制程序的执行，而kotlin intrinsics作用就是可以让我们调用它提供的基本方法获取Continuation，kotlin intrinsics在[kotlin-stdlib](https://github.com/JetBrains/kotlin/blob/1.4.0/libraries/stdlib/jvm/src/kotlin/coroutines/intrinsics/IntrinsicsJvm.kt)和[kotlinx-coroutines](https://github.com/Kotlin/kotlinx.coroutines/tree/native-mt-1.4.20/kotlinx-coroutines-core/common/src/intrinsics)都有相应的intrinsics包，而kotlinx-coroutines的intrinsics包是基于kotlin-stdlib的intrinsics包的安全实现，增加了一些try catch、启动时可取消、拦截的能力，kotlin-stdlib的intrinsics包是不推荐给用户使用的，因为使用它必须要注意一些问题，所以kotlin在IDE中隐藏了kotlin-stdlib的intrinsics包的智能提示，我们无法自动导入这个包只能手动导入，并且里面的方法在引用时也没有提示，只能手动编写，这里我主要讲kotlin-stdlib的intrinsics包的方法，因为它才是最基本的实现，如果平时开发使用，还是推荐使用kotlinx-coroutines的intrinsics包。

首先我们要手动导入kotlin-stdlib的intrinsics包：

```kotlin
import kotlin.coroutines.intrinsics.*
```

intrinsics包主要有两部分，一部分是基于suspend lambda方法创建Continuation，一部分是捕获suspend方法的Continuation，先看Continuation的创建：

```kotlin
public actual fun <T> (suspend () -> T).createCoroutineUnintercepted(completion: Continuation<T>)

public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

**createCoroutineUnintercepted**方法用来创建一个返回值为T类型的初始Continuation实例，创建Continuation实例需要传入一个完成延续，当需要执行这个Continuation时，就调用它的resumeWith方法，当这个Continuation执行完毕时，完成延续completion的resumeWith方法就会回调，suspend lambda方法是普通方法和suspend方法之间的**桥梁**，因为suspend方法只能在suspend方法中调用，所以为了调用suspend方法，我们只能在普通方法声明一个suspend lambda类型的参数，然后在调用普通方法时传入suspend lambda方法块，并在传入suspend lambda方法块中调用其他suspend方法，例如kotlin协程通过launch方法启动时，都是要传一个**block**块，这个block块就是一个suspend lambda方法，我们传进去的block最终会被调用它的createCoroutineUnintercepted方法创建初始协程的初始Continuation实例，以CoroutineStart.DEFAULT启动模式为例，调用链如下：

```kotlin
CoroutineScope.launch(context, start, block)
-> AbstractCoroutine.start(start, coroutine, block)
-> CoroutineStart.invoke(block, receiver, completion)
-> block.startCoroutineCancellable(receiver, completion)
-> createCoroutineUnintercepted(receiver, completion).intercepted().resumeCancellable(Result.success(Unit))
```

而前面讲过suspend lambda方法会被CPS成一个状态机实现，这个状态机继承自BaseContinuationImpl并且实现了**create**方法，而createCoroutineUnintercepted方法会调用这个create方法创建状态机实例作为协程的初始Continuation。

从上面launch的调用链可以看到调用了createCoroutineUnintercepted方法后会马上调用**intercepted**方法，intercepted方法是Continuation的扩展方法，它也属于intrinsics方法，方法签名如下：

```kotlin
fun Continuation<T>.intercepted(): Continuation<T>
```

这个方法的作用是在Continuation的上下文CoroutineContext中查找拦截器ContinuationInterceptor，并返回拦截器对Continuation的拦截延续，它包装了原始的Continuation，在Continuation恢复前做出一些其他操作，目前在协程实现中intercepted方法返回的是一个**DispatchedContinuation**，它的作用是在Continuation恢复前把它分发到对应上下文的Dispatcher中恢复，这样原始的Continuation就会被切换到对应的Dispatcher中执行，由于拦截在协程的执行过程会经常用到，所以kotlin就建议在调用createCoroutineUnintercepted方法创建了初始Continuation后和调用suspendCoroutineUninterceptedOrReturn方法捕获Continuation后马上调用它的intercepted方法，因为intercepted方法中会返回的拦截延续进行缓存，这样后续调用intercepted方法时就能马上返回。

前面多次讲到了**suspendCoroutineUninterceptedOrReturn**方法，它也属于intrinsics方法，suspendCoroutineUninterceptedOrReturn方法的作用是捕获suspend连续传递过来的Continuation，方法签名如下：

```kotlin
public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(crossinline block: (Continuation<T>) -> Any?): T 
```

suspendCoroutineUninterceptedOrReturn方法的返回值类型即为Continuation的结果类型，在block块中我们可以拿到Continuation实例，注意到block块的返回值为**Any**类型，还记得前面讲suspend方法CPS后的返回值也为Any类型，这个Any类型就是**T | COROUTINE_SUSPENDED**的组合类型，如果block块中返回了COROUTINE_SUSPENDED，则表示suspend方法需要挂起并且不会立即返回结果，在这种情况下，要在将来的某个时刻调用Continuation的resumeWith来恢复suspend方法的执行，如果block块中返回了T类型的值或者抛出了异常，这表示执行没有被挂起，suspend方法可以直接同步返回结果，suspendCoroutineUninterceptedOrReturn方法是一个非常实用且常用的intrinsics方法，通过它我们可以对普通回调方法进行包装，把它与suspend方法进行结合，如下：

```kotlin
fun main() {
  GlobalScope.launch {
        try {
            val data = fetchData()
            println(data)
        }catch (e: NullPointerException) {
            e.printStackTrace()
        }
    }
}

suspend fun fetchData(): String = suspendCoroutineUninterceptedOrReturn { continuation ->
    fetchDataAsync {
      	//恢复
        if(it.isNullOrEmpty()) {
            continuation.resumeWithException(NullPointerException())
        }else {
            continuation.resume(it)
        }
    }
    //挂起                                                                
    COROUTINE_SUSPENDED
}

fun fetchDataAsync(callback: (String) -> Unit) {
    Executors.newCachedThreadPool().execute {
        callback.invoke("result")
    }
}
```

可以看到suspendCoroutineUninterceptedOrReturn方法可以把回调方法结果在suspend方法中以同步的形式返回，除了这种应用，我还可以对捕获到的Continuation进行包装，像DispatchedContinuation那样在Continuation恢复前后自定义我们自己的逻辑，像协程提供的delay、await、withContext等方法都是利用suspendCoroutineUninterceptedOrReturn方法实现它们的逻辑，例如delay方法可以让捕获的Continuation延迟指定时间后恢复，await方法可以让捕获的Continuation等到协程完成得到结果后才恢复，但是我们在日常开发中一般不会直接使用suspendCoroutineUninterceptedOrReturn方法，因为使用不当会让线程出现**栈溢出**错误，suspendCoroutineUninterceptedOrReturn方法已经在注释中明确提示：

```kotlin
/**
 * Note that it is not recommended to call either Continuation.resume nor Continuation.resumeWithException   	* functions synchronously in the same stackframe where suspension function is run. Use suspendCoroutine as a 	* safer way to obtain current continuation instance.
 */
```

不推荐在运行suspend方法的同一堆栈帧中同步调用Continuation的resumeWith方法，例如：

```kotlin
suspend fun fetchData(): String = suspendCoroutineUninterceptedOrReturn { continuation ->
   //错误做法，不推荐在当前线程栈帧调用Continuation的resumeWith方法                                                    		continuation.resume("result")
}
```

因为如果我们直接在当前线程上同步调用resumeWith方法，就相当于**递归调用**上一个suspend方法，这样当当前线程长时间运行时，就会很容易出现栈溢出错误，注释提到推荐使用**suspendCoroutine**方法代替suspendCoroutineUninterceptedOrReturn方法获取当前Continuation，该方法定义在kotlin.coroutines中：

```kotlin
public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T {
    return suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
        val safe = SafeContinuation(c.intercepted())
        block(safe)
        safe.getOrThrow()
    }
}
```

通过suspendCoroutine获取到的Continuation是一个**SafeContinuation**，它是对suspendCoroutineUninterceptedOrReturn捕获的Continuation又一层包装，它可以让我们同步地、安全地调用Continuation的resumeWith方法，而不用考虑任何限制，如下：

```kotlin
suspend fun fetchData(): String = suspendCoroutine { continuation ->
   //没问题，SafeContinuation可以同步调用
   continuation.resume("result")
}
```

SafeContinuation的原理就是它重写了resumeWith方法，在同步调用的情况下，不调用真正的Continuation的resumeWith方法，而是先保存结果，然后把保存的结果在调用getOrThrow方法时直接return给调用方法，这样就避免了同步调用Continuation的resumeWith方法出现的问题，同时SafeContinuation还替我们封装好了返回COROUTINE_SUSPENDED的逻辑，我们使用suspendCoroutine需要挂起时不用再显式地返回COROUTINE_SUSPENDED，除了suspendCoroutine方法，我们还可以使用**suspendCancellableCoroutine**方法代替，它定义在kotlinx.coroutines中：

```kotlin
public suspend inline fun <T> suspendCancellableCoroutine( crossinline block: (CancellableContinuation<T>) -> Unit): T = suspendCoroutineUninterceptedOrReturn { uCont ->
      val cancellable = CancellableContinuationImpl(uCont.intercepted(), resumeMode = MODE_CANCELLABLE)
      cancellable.initCancellability()
      block(cancellable)
      cancellable.getResult()
 }
```

通过suspendCancellableCoroutine获取到的Continuation是一个**CancellableContinuation**，它也是对suspendCoroutineUninterceptedOrReturn捕获的Continuation又一层包装，它除了可以让我们同步地、安全地调用Continuation的resumeWith方法外，还可以取消Continuation同时响应协程的取消，例如：

```kotlin
suspend fun fetchData(): String = suspendCancellableCoroutine{ continuation ->
    val fileIs = FileInputStream(File("test"))
    continuation.invokeOnCancellation {
        //Continuation被取消时执行一些资源释放工
        fileIs.close()
    }                                                             
    //...do something
    //没问题，CancellableContinuation可以同步调用
    continuation.resume("result")                                      
}
```

kotlin intrinsics方法中还有一个**startCoroutineUninterceptedOrReturn**方法，当你调用它之后，他会创建一个Continuation并立即执行它，直到遇到第一个挂起点，而createCoroutineUnintercepted方法创建的Continuation需要你显式调用resumeWith方法才会执行，它的方法签名如下：

```kotlin
public inline fun <T> (suspend () -> T).startCoroutineUninterceptedOrReturn(completion: Continuation<T>): Any?

public inline fun <R, T> (suspend R.() -> T).startCoroutineUninterceptedOrReturn(receiver: R, completion: Continuation<T>): Any?
```

这个返回的调用同样需要传入一个Continuation作为该方法创建的Continuation的完成延续，当该方法创建的Continuation执行完毕后，完成延续completion的resumeWith方法就会被调用，与createCoroutineUnintercepted方法不同的是它的返回值是一个Any类型，这个Any类型的含义和前面讲的suspendCoroutineUninterceptedOrReturn方法中block块的返回值含义一样，这个方法的主要是和suspendCoroutineUninterceptedOrReturn方法结合使用，在相同的上下文中使用不同的suspend lambda块创建执行新的Continuation，并在新的Continuation结束后恢复suspendCoroutineUninterceptedOrReturn方法捕获的Continuation，例如withContext方法就使用到了suspendCoroutineUninterceptedOrReturn方法，如下：

```kotlin
public suspend fun <T> withContext(
    context: CoroutineContext,
    block: suspend CoroutineScope.() -> T
): T {
    return suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
        val oldContext = uCont.context
        val newContext = oldContext + context
        newContext.checkCompletion()
        // FAST PATH #1 -- newContext等于oldContext，不需要执行上下文切换
        if (newContext === oldContext) {
            val coroutine = ScopeCoroutine(newContext, uCont)
            //最终调用block.startCoroutineUninterceptedOrReturn方法
            return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
        }
        // FAST PATH #2 -- newContext的Dispatcher等于oldContext的oldContext，不需要切换Dispatcher
        if (newContext[ContinuationInterceptor] == oldContext[ContinuationInterceptor]) {
            val coroutine = UndispatchedCoroutine(newContext, uCont)
            withCoroutineContext(newContext, null) {
              	//最终调用block.startCoroutineUninterceptedOrReturn方法
                return@sc coroutine.startUndispatchedOrReturn(coroutine, block)
            }
        }
        // SLOW PATH -- newContext不等于oldContext
        val coroutine = DispatchedCoroutine(newContext, uCont)
        coroutine.initParentJob()
        //最终调用block.createCoroutineUnintercepted方法
        block.startCoroutineCancellable(coroutine, coroutine)
        coroutine.getResult()
    }
}
```

withContext方法的作用是把block块运行在新的上下文中，并返回block块的运行结果，同时返回时会切换到原来的上下文中，withContext方法在不需要进行Dispatcher切换的情况中会直接使用block.startCoroutineUninterceptedOrReturn方法，这样会减少无谓的intercepted方法调用。

createCoroutineUnintercepted、startCoroutineUninterceptedOrReturn、suspendCoroutineUninterceptedOrReturn三个方法就是kotlin intrinsice中最常用到的方法，用来创建、启动和捕获Continuation，kotlin协程的本质就是启动、调度和管理Continuation，所以说intrinsics方法是kotlin协程实现的基石。

## Continuation的恢复

从前面可以看到，当我们需要从挂起点恢复被挂起的Continuation或者首次执行这个Continuation时，就要调用[Continuation](https://github.com/JetBrains/kotlin/blob/1.4.0/libraries/stdlib/src/kotlin/coroutines/Continuation.kt)接口的resumeWith方法，resumeWith方法方法根据Continuation的子类不同有不同的实现，在kotlin协程中，Continuation主要有**BaseContinuationImpl**、**DispatchedContinuation**、**SafeContinuation**、**CancellableContinuation**、**AbstractCoroutine**这几种实现，下面主要讲一下DispatchedContinuation、BaseContinuationImpl和AbstractCoroutine的resumeWith方法实现，它们之间的关系如下：

{% asset_img coroutine1.png coroutine %}

**DispatchedContinuation**

DispatchedContinuation就是把Continuation分发到对应上下文的Dispatcher中执行，当我们需要拦截Continuation时，就调用它的intercepted方法获取它的DispatchedContinuation，当一个Continuation被拦截后，后续它执行都在对应的Dispatcher中，DispatchedContinuation当resumeWith方法实现如下：

```kotlin
internal class DispatchedContinuation<in T>(
    @JvmField val dispatcher: CoroutineDispatcher,//被拦截Continuation的Dispatcher
    @JvmField val continuation: Continuation<T>//被拦截的Continuation
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation {
  
  //...
  
  override fun resumeWith(result: Result<T>) {
        val context = continuation.context
        val state = result.toState()
        if (dispatcher.isDispatchNeeded(context)) {//IO、DEFAULT、MAIN走这里逻辑
            _state = state
            resumeMode = MODE_ATOMIC
            //调用dispatch方法后，DispatchedTask的run方法会执行
            dispatcher.dispatch(context, this)
        } else {//Unconfined走这里的逻辑
            executeUnconfined(state, MODE_ATOMIC) {
                withCoroutineContext(this.context, countOrElement) {
                   //调用Continuation的resumeWith方法
                    continuation.resumeWith(result)
                }
            }
        }
  }
}

internal abstract class DispatchedTask<in T>(@JvmField public var resumeMode: Int) : SchedulerTask() {
  
  //...
  
  public final override fun run() {
    val taskContext = this.taskContext
    var fatalException: Throwable? = null
    try {
      val delegate = delegate as DispatchedContinuation<T>
      val continuation = delegate.continuation
      val context = continuation.context
      val state = takeState()
      withCoroutineContext(context, delegate.countOrElement) {
        val exception = getExceptionalResult(state)
        val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
        if (job != null && !job.isActive) {
          val cause = job.getCancellationException()
          cancelCompletedResult(state, cause)
          continuation.resumeWithStackTrace(cause)
        } else {
          //调用Continuation的resumeWith方法
          if (exception != null) {
            continuation.resumeWithException(exception)
          } else {
            continuation.resume(getSuccessfulResult(state))
          }
        }
      }
    } catch (e: Throwable) {
      fatalException = e
    } finally {
      val result = runCatching { taskContext.afterTask() }
      handleFatalException(fatalException, result.exceptionOrNull())
    }
  }
}
```

可以看到如果Dispatcher是Unconfined，那么就会在当前线程调用Continuation的resumeWith方法，如果Dispatcher是IO、DEFAULT、MAIN，就调用它们的dispatch方法提交DispatchedTask任务等待调度执行，而DispatchedContinuation同时又继承自DispatchedTask，所以它是一个DispatchedTask，等IO、DEFAULT、MAIN的Dispatcher调度时，run方法就会执行，这时就调用Continuation的resumeWith方法，这样Continuation就被分发到对应上下文的线程中恢复。

**BaseContinuationImpl**

[BaseContinuationImpl](https://github.com/JetBrains/kotlin/blob/1.4.0/libraries/stdlib/jvm/src/kotlin/coroutines/jvm/internal/ContinuationImpl.kt)是所有suspend方法状态机的共同父类，例如子类ContinuationImpl就表示suspend命名方法，子类SuspendLambda就表示suspend lambda方法，除了这些普通的suspend方法外，kotlin中还有一种受限suspend方法，它是一种带有限制的suspend方法作用域，在这种带限制的suspend方法中只能调用**@RestrictsSuspension**注解的类中定义的suspend方法，例如[sequence](https://kotlinlang.org/docs/sequences.html)方法的block块就是一个带有限制的suspend lambda方法：

```kotlin
fun main() {
    sequence<Int> {
      //display() 报错，不允许调用其他suspend方法
      yield(1) //只能调用被@RestrictsSuspension注解的SequenceScope类中定义的yield方法
  }
}

public fun <T> sequence(@BuilderInference block: suspend SequenceScope<T>.() -> Unit): Sequence<T> {
  //...
}

@RestrictsSuspension
public abstract class SequenceScope<in T> internal constructor() {
    
    public abstract suspend fun yield(value: T)
  
 	 //...
}
```

受限的suspend方法用RestrictedContinuationImpl表示，受限的suspend lambda方法用RestrictedSuspendLambda表，当我们调用BaseContinuationImpl的resumeWith方法时，就是在执行当前suspend方法的状态机，并且在状态机运行结束时恢复外部Continuation，我们可以看一下BaseContinuationImpl的resumeWith方法的实现：

```kotlin
internal abstract class BaseContinuationImpl(
    //每个BaseContinuationImpl实例都会引用一个完成Continuation，用来在当前状态机流转结束时恢复这个Continuation
    public val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable {
  
    //resumeWith方法中通过循环由里到外恢复Continuation
    public final override fun resumeWith(result: Result<Any?>) {
        var current = this
        var param = result
        while (true) {
            probeCoroutineResumed(current)
            with(current) {
                val completion = completion!!
                val outcome: Result<Any?> =
                    try {
                        //通过调用invokeSuspend方法执行当前suspend方法主体，进行状态流转
                        val outcome = invokeSuspend(param)
                        if (outcome === COROUTINE_SUSPENDED) return
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
                //当invokeSuspend方法没有返回COROUTINE_SUSPENDED，就表示当前状态机流转结束，即当前suspend方法执行完毕
                releaseIntercepted() 
              	//然后在这里判断是否还有suspend方法需要恢复
                if (completion is BaseContinuationImpl) { //completion是suspend方法，继续恢复
                    current = completion
                    param = outcome
                } else { //completion不是suspend方法，调用resumeWith方法恢复
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }

    //每个suspend方法状态机都要实现这个接口，用来调用suspend方法主体
    protected abstract fun invokeSuspend(result: Result<Any?>): Any?

    //...
}
```

每个BaseContinuationImpl实例就代表一个suspend方法状态机，当suspend方法状态机执行结束时，BaseContinuationImpl就会恢复引用的完成Continuation，如果完成Continuation是suspend方法，就调用它状态机的invokeSuspend方法，当遇到完成Continuation不是suspend方法时，就调用它的resumeWith方法执行对应的逻辑。

**AbstractCoroutine**

[AbstractCoroutine](https://github.com/Kotlin/kotlinx.coroutines/blob/native-mt-1.4.20/kotlinx-coroutines-core/common/src/AbstractCoroutine.kt)是kotlin协程的基类，AbstractCoroutine在kotlin协程实现中会作为**最后一个恢复的Continaution**，所以当所有suspend方法都执行完毕后，AbstractCoroutine的resumeWith方法就会被调用，这时它就可以进行协程的生命周期流转，例如判断子协程是否完成，如果子协程都完成了，那么就能置为完成状态，否则就置为完成中状态等待所有子协程完成，如下：

```kotlin
public abstract class AbstractCoroutine<in T>(protected val parentContext: CoroutineContext, active: Boolean = true) : JobSupport(active), Job, Continuation<T>, CoroutineScope {

  	//...
  
    public final override fun resumeWith(result: Result<T>) {
      //进行协程生命周期流转
      val state = makeCompletingOnce(result.toState())
      if (state === COMPLETING_WAITING_CHILDREN) return
      afterResume(state)
    }
}
```

## 自己实现Coroutine

当我们通过launch方法传入block块启动一个协程，本质是通过这个block块创建了一个Continuation，当我们在block块中调用其他suspend方法，并且suspend方法中再调用其他suspend方法，Continuation就会在这些suspend方法之间传递，最终我们可以捕获到连续传递的Continuation，当我们通过Continuation恢复时，本质是上一个suspend方法的递归调用进行状态流转，而kotlin协程只是在这些Continuation的基础上添加了生命周期管理、父子关系、异常处理、线程切换等逻辑。

通过intrinsics方法，我们自己也可以实现一个协程，这里我通过intrinsics方法仿照kotlin协程写了个简化版的协程，它这样使用：

```kotlin
fun main() {
    val simpleScope = SimpleCoroutineScope(Dispatchers.Default)
    val simpleJob = simpleScope.launch(CoroutineName("main"), CoroutineStart.DEFAULT) {
        val user = login()
        val userData = fetchData(user)
        displayUI(userData)
    }
    simpleJob.invokeOnCompletion(object : SimpleJob.CompletionHandler {
        override fun invoke(cause: Throwable?) {
            println("invokeOnCompletion: cause = $cause")
        }
    })
  
  	//进程保活
    Thread.sleep(1000)
}

private suspend fun SimpleCoroutineScope.login(): String {
    return async(CoroutineName("login")) {
        delay(200)
        return@async "user"
    }.await()
}

private suspend fun SimpleCoroutineScope.fetchData(user: String): String {
    return async(CoroutineName("fetch")) {
        delay(200)
        return@async "$user data"
    }.await()
}

private fun displayUI(data: String) {
    println("displayUI: $data")
}

//运行输出：
//displayUI: user data
//invokeOnCompletion: cause = null
```

invokeOnCompletion方法回调会在协程完成后被调用，如果协程正常完成那么，cause为null，如果协程异常完成，那么cause为对应的异常，上面协程正常完成，所有实现代码如下：

```kotlin
import kotlinx.coroutines.*
import kotlin.coroutines.*
import kotlin.coroutines.intrinsics.*
import java.util.concurrent.CopyOnWriteArraySet

/**
 * 简化版的[CoroutineScope]，提供协程运行作用域，它与[CoroutineScope]的区别是没有[CoroutineScope.cancel]、[CoroutineScope.ensureActive]等这些扩展方法
 */
private interface SimpleCoroutineScope {

    val coroutineContext: CoroutineContext
}

/**
 * [SimpleCoroutineScope]的实现
 */
private class SimpleCoroutineScopeImpl(override val coroutineContext: CoroutineContext) : SimpleCoroutineScope

/**
 * 构造[SimpleCoroutineScope]实例
 */
private fun SimpleCoroutineScope(context: CoroutineContext) = SimpleCoroutineScopeImpl(if(context[SimpleJob] != null) context else context + SimpleJob())

/**
 * 简化版的[Job]，用于管理协程的生命周期，它与[Job]的区别是它没有取消操作、异常传播、异常处理等功能，只有简单的状态流转：
 *
 *      start/await
 * NEW -------------> ACTIVE (isActive = true)
 *   \                /
 *    \  fail/finish /
 *     \            /
 *       COMPLETING
 *           |
 *           | wait children
 *           v
 *       COMPLETE (isComplete = true)
 */
private interface SimpleJob : CoroutineContext.Element {

    companion object Key : CoroutineContext.Key<SimpleJob>

    /**
     * 协程是否已启动
     */
    fun isActive(): Boolean

    /**
     * 协程是否已完成
     */
    fun isComplete(): Boolean

    /**
     * 启动协程
     */
    fun start()

    /**
     * 等待协程的结果返回
     */
    suspend fun <T> await(): T

    /**
     * 注册协程完成回调[completionHandler]，返回的[DisposableHandle]可以用来反注册回调
     */
    fun invokeOnCompletion(completionHandler: CompletionHandler, invokeImmediately: Boolean = true): DisposableHandle

    /**
     * 建立起与[childJob]子协程的父子关系
     */
    fun attachChild(childJob: SimpleJob)

    /**
     * 协程完成通知回调
     */
    interface CompletionHandler {

        /**
         * cause == null -> 成功结束
         * cause == other -> 异常结束
         */
        fun invoke(cause: Throwable?)
    }

    /**
     * 反注册句柄
     */
    interface DisposableHandle {

        /**
         * 调用[dispose]方法反注册
         */
        fun dispose()
    }
}

/**
 * [SimpleJob]的实现
 */
private open class SimpleJobImpl(active: Boolean) : SimpleJob {

    enum class State {
        NEW, ACTIVE, COMPLETING, COMPLETED
    }

    override val key: CoroutineContext.Key<*> get() = SimpleJob

    @Volatile
    private var state = if(active) State.ACTIVE else State.NEW
    @Volatile
    private var result: Any? = null
    private val children = CopyOnWriteArraySet<SimpleJob>()
    private val completionHandlers = CopyOnWriteArraySet<SimpleJob.CompletionHandler>()

    override fun isActive(): Boolean {
        return state == State.ACTIVE
    }

    override fun isComplete(): Boolean {
        return state == State.COMPLETED
    }

    override fun start() {
        if(state == State.NEW) {
            state = State.ACTIVE
            onStart()
        }
    }

    override suspend fun <T> await(): T {
        if(state == State.COMPLETED) {
            if(result is Throwable) {
                throw result as Throwable
            }else {
                return result as T
            }
        }
        if(state == State.NEW) {
            start()
        }
        return suspendCoroutineUninterceptedOrReturn {
            invokeOnCompletion(object : SimpleJob.CompletionHandler {
                override fun invoke(cause: Throwable?) {
                     if(cause != null) {
                         it.resumeWithException(cause)
                     }else {
                         it.resume(result as T)
                     }
                }
            })
            COROUTINE_SUSPENDED
        }
    }

    override fun invokeOnCompletion(completionHandler: SimpleJob.CompletionHandler, invokeImmediately: Boolean): SimpleJob.DisposableHandle {
        if(invokeImmediately && state == State.COMPLETED) {
            completionHandler.invoke(result as? Throwable)
        }
        completionHandlers.add(completionHandler)
        return CompletionHandlerDisposeHandle(completionHandler)
    }

    override fun attachChild(childJob: SimpleJob) {
        children.add(childJob)
    }

    protected fun initParentJob(parentJob: SimpleJob?) {
        parentJob?.start()
        parentJob?.attachChild(this)
    }

    protected fun tryMakeCompleted(value: Any?): Boolean {
        result = value ?: result
        val complete = children.find { !it.isComplete() } == null
        if(complete) {
            if(state == State.COMPLETED) {
                return true
            }
            state = State.COMPLETED
            val cause = if(result is Throwable) { result as Throwable } else { null }
            notifyCompleteHandlers(cause)
        }else {// 等待所有child完成
            if(state == State.COMPLETING) {
                return false
            }
            state = State.COMPLETING
            children.forEach {
                it.invokeOnCompletion(object : SimpleJob.CompletionHandler {
                    override fun invoke(cause: Throwable?) {
                        tryMakeCompleted(cause)
                    }
                }, invokeImmediately = true)
            }
        }
        return complete
    }

    private fun notifyCompleteHandlers(cause: Throwable?) {
        completionHandlers.forEach {
            it.invoke(cause)
        }
    }

    /**
     * 协程调用start/await方法从[State.NEW]转移到[State.ACTIVE]
     */
    protected open fun onStart() {}

    /**
     * 调用[dispose]方法解除注册的[completionHandler]
     */
    inner class CompletionHandlerDisposeHandle(private val completionHandler: SimpleJob.CompletionHandler) : SimpleJob.DisposableHandle {

        override fun dispose() {
            completionHandlers.remove(completionHandler)
        }
    }
}

/**
 * 构造[SimpleJob]实例
 */
private fun SimpleJob() = SimpleJobImpl(active = true)

/**
 * 简化版的协程，调用start方法启动协程
 * @param parentContext 协程的父Context，用于建立父子关系
 * @param active 为true时让协程处于active状态，否则处于new状态，处于new状态需要调用start/await方法才会启动协程
 */
private open class SimpleCoroutine<T>(private val parentContext: CoroutineContext, active: Boolean = true) : SimpleJobImpl(active), SimpleCoroutineScope, Continuation<T> {

    override val context: CoroutineContext = parentContext + this

    override val coroutineContext: CoroutineContext get() = context

    /**
     * 协程完成通知，这里处理结果，进行生命周期状态流转
     */
    override fun resumeWith(result: Result<T>) {
        println("resumeWith: result = $result, coroutineName = ${coroutineContext[CoroutineName]}")
        if(result.isSuccess) {//成功恢复
            tryMakeCompleted(result.getOrNull())
        }else {//错误恢复
            tryMakeCompleted(result.exceptionOrNull())
        }
    }

    /**
     * for [CoroutineStart.LAZY]
     */
    private var lazyContinuation: Continuation<Unit>? = null

    /**
     * for [CoroutineStart.LAZY]
     */
    override fun onStart() {
        lazyContinuation?.intercepted()?.resumeWith(Result.success(Unit))
    }

    /**
     * 建立协程的父子关系，使用[kotlin.coroutines.intrinsics]原语为[block]块创建协程的初始化[Continuation], 并根据[start]模式启动它
     */
    fun start(start: CoroutineStart, block: suspend SimpleCoroutineScope.() -> T) {
        if(coroutineContext[CoroutineExceptionHandler] != null) {
            throw IllegalAccessException("unsupport CoroutineExceptionHandler")
        }
        initParentJob(parentContext[SimpleJob])
        when(start) {
            /**
             * 立即启动协程，并把启动的协程运行在指定的Dispatcher上
             */
            CoroutineStart.DEFAULT -> {
                block.createCoroutineUnintercepted(this, this).intercepted().resumeWith(Result.success(Unit))
            }
            /**
             * 在当前线程立即启动协程, 但恢复时会把协程运行在指定的Dispatcher上，效果和指定[Dispatchers.Unconfined]类似
             */
            CoroutineStart.UNDISPATCHED -> {
                val result = try {
                    block.startCoroutineUninterceptedOrReturn(this, this)
                }catch (e: Throwable) {
                    e
                }
                if(result is Throwable) {
                    this.resumeWithException(result)
                }else if(result !== COROUTINE_SUSPENDED) {
                    this.resume(result as T)
                }else {
                    //COROUTINE_SUSPENDED, do noting
                }
            }
            /**
             * 不立即启动协程，当调用start/await方法时才启动协程，并把启动的协程运行在指定的Dispatcher上
             */
            CoroutineStart.LAZY -> {
                lazyContinuation = block.createCoroutineUnintercepted(this, this)
            }
            else -> {
                throw IllegalAccessException("unsupport $start")
            }
        }
    }
}

/**
 * 启动协程，没有结果返回
 */
private fun SimpleCoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend SimpleCoroutineScope.() -> Unit
): SimpleJob {
    val newContext = coroutineContext + context
    val coroutine = if(start == CoroutineStart.LAZY) {
        SimpleCoroutine<Unit>(newContext, active = false)
    }else {
        SimpleCoroutine<Unit>(newContext, active = true)
    }
    coroutine.start(start, block)
    return coroutine
}

/**
 * 启动协程，可以调用返回的SimpleJob的await方法等待结果
 */
private fun <T> SimpleCoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend SimpleCoroutineScope.() -> T
): SimpleJob {
    val newContext = coroutineContext + context
    val coroutine = if(start == CoroutineStart.LAZY) {
        SimpleCoroutine<T>(newContext, active = false)
    }else {
        SimpleCoroutine<T>(newContext, active = true)
    }
    coroutine.start(start, block)
    return coroutine
}
```

在这里我自定义了SimpleCoroutineScope、SimpleJob、SimpleCoroutine分别对应kotlin协程的CoroutineScope、Job、AbstractCoroutine角色，实现了协程的launch、async方法，支持DEFAULT、LAZY、UNDISPATCHED三种启动模式，在kotlin协程中CoroutineScope是用来控制协程的作用域，Job是用来管理协程的生命周期和父子关系，而AbstractCoroutine实现了Continuation同时继承自Job，它的作用在前面也讲过，就是当所有suspend方法都执行完毕后，AbstractCoroutine的resumeWith方法就会被调用，这时它就可以进行协程的生命周期流转，DEFAULT模式表示立即启动，所以它调用了createCoroutineUnintercepted方法创建初始Continuation后马上调用resumeWith方法执行它，LAZY模式表示延迟启动，所以它通过createCoroutineUnintercepted方法创建的初始Continuation的resumeWith方法会等到调用start方法时才调用，而UNDISPATCHED模式表示在当前线程立即启动，所以它通过startCoroutineUninterceptedOrReturn方法创建并执行Continuation，希望大家通过这个简化版的协程理解kotlin协程中角色的作用。

## 结语

本文介绍了kotlin协程的实现思想，Continuation、CPS和suspend方法的实现，不只是kotlin协程，其他语言的协程的实现思想也是类似的，同时还介绍了kotlin提供的intrinsics方法，它是用于给用户操纵这些Continuation，最后通过intrinsics方法实现了一个简化版的kotlin协程，所以kotlin协程也没有那么神秘，它只是Continuation的应用，它只是在这些Continuation的基础上添加了生命周期管理、父子关系、异常处理、线程切换等逻辑。

以上就是本文的所有内容，希望大家有所收获！

参考文档：

[Continuation-passing Style介绍及应用](https://blog.csdn.net/pzhang_9_25/article/details/7832072)

[KEEP-Kotlin Coroutines](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md#implementation-details)

[The suspend modifier](https://medium.com/androiddevelopers/the-suspend-modifier-under-the-hood-b7ce46af624f)