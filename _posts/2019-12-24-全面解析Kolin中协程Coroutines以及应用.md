---
layout: post
title: "全面解析Kolin中协程Coroutines以及应用"
date: 2019-12-24
description: "Kotlin协程"
tag: Kotlin 
--- 
#### 作为Kotlin语言来说，协程完全是一个可以化腐朽为神奇的东西，本篇全盘分析了Kotlin协程Coroutines

## 理解异步回调本质

### 1. 什么是异步

```
我记得小学二年级碰到过一个让我受益终身的数学题：
烧开水需要15分钟，洗碗需要5分钟，扫地需要5分钟，请问做完这三件事，总共需要几分钟？从此我做什么事，都事先想想先后顺序，看看可不可以一并去做。

长大后才知道这就是异步的用法，它其实已经渗透到你的生活中。
```
上面这段话节选自：余叶《代码里的世界观——通往架构师之路》，这段话中揭示了异步的本质。异步意味着同时进行一个以上彼此目的不同的任务。
如果上面三个任务一个一个按部就班地去做的话，你可能总共需要25分钟。但是烧开水的时候并不需要你在一旁一直等待着，如果你利用烧开水的时间去完成洗碗和扫地的任务，你只需要15分钟就可以完成以上三个任务。
接着试着对着Android（或者其他任何UI开发框架）的线程模型类比一下：

你是主线程。烧开水是个耗时操作（比如网络请求），洗碗和扫地是与视图相关的非耗时操作。洗碗和扫地必须由你本人亲自完成（视图相关的工作只能交给主线程），烧开水可以交给电磁炉完成，你只需要按下电磁炉的开关（可以类比成网络请求的发起）。

没有使用异步也就意味着你在烧开水的时候一直在旁边等待着，无法完成其他工作，这也就意味着Android在等待网络请求的时候主线程阻塞，视图卡顿无法交互，这在Android中当然是不允许的。

所以必须使用异步的方式，你（主线程）在按下电磁炉开关（发起网络请求）之后就继续完成洗碗扫地（视图交互）等其他任务了。

所以异步很好理解，就是同时进行着不同的任务就叫做异步。

### 2. 为什么需要回调

当你按下电磁炉的按钮，并地利用这段时间完成了扫地的任务，你感到很有成就感。心里想：异步机制真好，原来25分钟的工作如今只需要15分钟就能完成，以后我任何的工作都要异步地完成。

不过你并没有高兴太久，在洗碗时你遇到了麻烦。碗和盘子上沾满了油污，单靠自来水和洗洁精根本搞不定，这时你想到了别人教你的方法，常温的水洗不掉的油污用热水就可以轻松洗掉。

但是你发现暖水瓶是空的，而放在电磁炉上的水刚烧了5分钟，还不够热，也就是说你必须等待水烧开了才能开始洗碗。

这时你不禁陷入了思考：异步机制真的是万能的吗？对于有前后依赖关系的任务，异步该如何处理呢？这段等待烧水的时间我可以去做其他工作吗？我怎么确定水什么时候才能烧开呢？

这时，你眼前一亮：你发现了买水壶时赠送的一个配件，那是一个汽笛，它可以在水烧开的时候发出鸣叫声。听到了汽笛声你就可以知道水烧开了，接着就可以用刚烧开的热水来刷碗，并且烧水的过程中你仍然可以去完成其他工作（比如看技术博客），而不用担心烧水的问题。这个汽笛就可以看成异步中的回调机制。
![](/images/xiecheng/xiecheng_1.jpg)
同样地我们来类比一下Android开发中的场景：

洗碗（渲染某个视图）依赖于烧开水（网络请求）这个耗时操作的结果才能进行，所以你（主线程）在按下电磁炉开关（发起网络请求）的时候，为水壶装上了汽笛（为网络请求配置了回调），以便在水烧开（网络请求完成）的时候，汽笛发出鸣叫（回调函数被调用）你（主线程）就可以继续用烧开的水（网络请求的结果）洗碗（渲染某个视图）了，而等待水烧开（等待网络请求结果）的时候还可以去看技术博客（视图渲染与交互）。

这在Android开发过程中几乎是基础得不能再基础的应用场景了，可以说几乎所有的Android应用程序都有这样的一个过程。

所以理解为什么需要回调也很简单：因为不同的任务之间存在前后的依赖关系。

### 3. 回调的缺点
以上的应用场景相对简单，回调处理起来也游刃有余，可以描述为以下代码：

```
//烧2000mL热水来洗碗
boilWater(2000) { water ->
     washDishes(water)           
}

```
但函数回调也有缺陷，就是代码结构过分耦合，遇到多重函数回调的嵌套，代码难以维护。

比如客户端顺序进行多次网络异步请求:

```
//客户端顺序进行三次网络异步请求，并用最终结果更新UI
request1(parameter) { value1 ->
    request2(value1) { value2 ->
        request3(value2) { value3 ->
            updateUI(value3)            
        } 
    }              
}
```
这种结构的代码无论是阅读起来还是维护起来都是极其糟糕的。对多个回调组成的嵌套耦合，我亲切地称为“回调地狱（Callback Hell）”。

解决回调地狱的方案有很多，其中比较常见的有：链式调用结构。例如：

```
request1(parameter)
    .map { value1 ->
         request2(value1)
       }.map { value2 ->
        request3(value2)
       }.subscribe { value3 ->
        updateUI(value3)
       }

```
上面的代码看起来就舒服多了，这就是链式调用结构的魅力。实现链式调用结构的常见方式就是使用RxJava，RxJava是一个强大的工具，它是反应函数式编程在Java中的实现，我们可以通过RxJava中的“流”来构建链式调用结构。

虽然RxJava足够强大，但是它也足够复杂，RxJava中“流”的创建、转化与消费都需要使用到它提供的各种类和丰富的操作符，所以要想对RxJava运用自如就需要对这些类和操作符非常熟悉，这也加大了RxJava的学习成本了。

我们可以链式调用结构中获得一些启发，虽然回调嵌套和链式调用在代码结构上完全不一样，但是其表达的东西完全一致。也就是说回调嵌套和链式调用者两种结构表达的都是同一种逻辑，这不禁让我们想对于回调的本质做一些深入思考，究竟回调的背后是什么东西呢？

### 4. 深入理解异步回调（重点）

在接触多线程编程之前，我们天真地认为代码就是从上到下一行一行地执行的。代码中的基本结构只有三种：顺序、分支、循环，顺序结构中写在上面的代码就是比写在下面的代码先执行，写在下面的代码就是要等到上面的代码执行完了才能得到执行。

但接触到了多线程和并发之后，我们之前在脑袋里建立的秩序的世界完全崩塌了，取而代之的是一个混沌的世界。代码的执行顺序好像完全失控了，可能有多处的代码一起执行，写在下面的代码也可能先于上面的执行。

举个简单的例子：

```
//1.简单秩序的串行世界：
print("Hello ")
print("World!")
//结果为：Hello World!

//2.复杂混沌的并行世界：
Thread { 
    Thread.sleep(2000)
    print("Hello ") 
}.start()
print("World!")
```

那么我们思考一下在串行的世界里，由回调组织起来的代码结构属于顺序、分支、循环哪种呢？应该不难发现：烧完水再洗碗，网络请求成功再更新UI，这些看似复杂的回调结构其实表达的就是一种代码的顺序执行的方式。

回过头来看看之前提到的回调嵌套的例子，如果放在简单的串行世界里代码其实完全可以写成这样：

```
val value1 = request1(parameter)
val value2 = request2(value1)
val value3 = request2(value2)
updateUI(value3)
```

上面代码的执行顺序与下面的回调方式组织代码的执行顺序完全相同：


```
request1(parameter) { value1 ->
    request2(value1) { value2 ->
        request3(value2) { value3 ->
            updateUI(value3)            
        } 
    }              
}
```

既然代码执行顺序完全一致为什么我们还要使用回调这么麻烦的方式来顺序执行代码呢？原因就在于我们的世界不是简单的串行世界，实际的程序也不是只有一个线程那么简单的。

顺序代码结构是阻塞式的，每一行代码的执行都会使线程阻塞在那里，但是主线程的阻塞会导致很严重的问题，所以也就决定了所有的耗时操作不能在主线程中执行，所以就需要多线程来执行。

对于上面的例子，虽然代码执行顺序是：request1 -> request2 -> request3 -> updateUI。

![](/images/xiecheng/xiecheng_2.jpg)

但是他们是有可能工作在不同的线程上的，比如：request1(work thread) -> request2(work thread) -> request3(work thread) -> updateUI(main thread)。也就是说虽然代码确实是顺序执行的，但其实是在不同的线程上顺序执行的。

![](/images/xiecheng/xiecheng_3.jpg)


通常线程切换的工作是由异步函数内部完成的，通过回调的方式异步调用外界注入的代码。也就是说：异步回调其实就是代码的多线程顺序执行。

那么能不能既按照顺序的方式编写代码，又可以让代码在不同的线程顺序执行呢？有没有一个东西可以帮助我自动地完成线程的切换工作呢？答案当然是肯定的，接下来就轮到Kotlin协程大显身手的时候了。

##  Coroutines初体验

### 1. 添加依赖 
Kotlin协程不属于Kotlin语言本身，使用之前必须手动引入。在Android平台上使用可以添加Gradle依赖：


```
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.1.1'

```
### 2. 启动协程

首先看下如下代码：


```
GlobalScope.launch {
    delay(1000L)    
    println("Hello,World!")
}
```
上述代码使用launch方法启动了一个协程，launch后面的花括号就是协程，花括号内的代码就是运行在协程内的代码。

接着来深入了解一下launch方法的声明：


```
public fun CoroutineScope.launch(    
    context: CoroutineContext = EmptyCoroutineContext,    
    start: CoroutineStart = CoroutineStart.DEFAULT,    
    block: suspend CoroutineScope.() -> Unit): Job {...}
```
可以看到launch方法是CoroutineScope的拓展方法，也就是说我们启动协程要在一个指定的CoroutineScope上来启动。

CoroutineScope翻译过来就是“协程范围”，指的是协程内的代码运行的时间周期范围，如果超出了指定的协程范围，协程会被取消执行，上面第一段代码中的GlobalScope指的是与应用进程相同的协程范围，也就是在进程没有结束之前协程内的代码都可以运行。

除此之外为了方便我们的使用，在Google的Jetpack中也提供了一些生命周期感知型协程范围。实际开发中我们可以方便地选择适当的协程范围来为耗时操作（网络请求等）指定自动取消执行的时机，详情见:https://developer.android.google.cn/topic/libraries/architecture/coroutines

接着可以看下launch方法的其他参数：
1. context：协程上下文，可以指定协程运行的线程。默认与指定的CoroutineScope中的coroutineContext保持一致，比如GlobalScope默认运行在一个后台工作线程内。也可以通过显示指定参数来更改协程运行的线程，Dispatchers提供了几个值可以指定：Dispatchers.Default、Dispatchers.Main、Dispatchers.IO、Dispatchers.Unconfined。
2. start：协程的启动模式。默认的（也是最常用的）CoroutineStart.DEFAULT是指协程立即执行，除此之外还有CoroutineStart.LAZY、CoroutineStart.ATOMIC、CoroutineStart.UNDISPATCHED。
3. block：协程主体。也就是要在协程内部运行的代码，可以通过lamda表达式的方式方便的编写协程内运行的代码。
4. CoroutineExceptionHandler：除此之外还可以指定CoroutineExceptionHandler来处理协程内部的异常。

返回值Job：对当前创建的协程的引用。可以通过Job的start、cancel、join等方法来控制协程的启动和取消。

启动协程不是只有launch一个方法的，还有async等其他方法可以启动协程，不过launch是最常用的一种方法，其他的方法大家可以去自行了解。

### 3. 调用挂起函数
回到上面的代码：


```
println("Start")
GlobalScope.launch(Dispatchers.Main) {
    delay(1000L)
    println("Hello World")
}
println("End")
```
首先通过GlobalScope.launch启动了一个协程，这里指定协程运行的线程为主线程，接着协程内只有两行代码，协程启动之后就立即执行。首先直接输出了"Start"和"End"，接着1秒钟后又输出了"Hello World"。这结果看起来看似顺理成章，因为我们使用非常相似的Thread相关的代码也完全可以实现以上代码的效果：


```
println("Start")
Thread {
    Thread.sleep(1000L)
    println("Hello World")
}.start()
println("End")
```
两段代码看起来长得几乎一模一样，运行结果也完全一致。那究竟协程的神奇之处在哪里呢？顺序编写异步代码有体现在什么地方呢？

我们在上面两段代码的所有输出的位置上全部加上输出当前线程名的操作：


```
/协程代码
println("Start ${Thread.currentThread().name}")
GlobalScope.launch(Dispatchers.Main) {
    delay(1000L)
    println("Hello World ${Thread.currentThread().name}")
}
println("End ${Thread.currentThread().name}")
```

```
//线程代码
println("Start ${Thread.currentThread().name}")
Thread {
    Thread.sleep(1000L)
    println("Hello World ${Thread.currentThread().name}")
}.start()
println("End ${Thread.currentThread().name}")
```
线程代码输出为：“Start main”->“End main”->“Hello World Thread-2”。这个结果也很好理解，首先在主线程里输出"Start"，接着创建了一个新的线程并启动后阻塞一秒，这时主线程继续向下执行输出"End"，这时启动的线程阻塞时间结束，在当前创建的线程输出"Hello World"。

协程代码输出为：“Start main”->“End main”->“Hello World main”。前两个输出很好理解与上面一致，但是等待一秒之后协程里面的输出结果却显示当前输出的线程为主线程！

这是个很神奇的事情，输出"Start"之后就立即输出了"End"说明了我们的主线程并没有被阻塞，等待的那一秒钟被阻塞的一定是其他线程。

但是阻塞结束后的输出却发生在主线程中，这说明了一件事：协程中的代码自动地切换到其他线程之后又自动地切换回了主线程！这不正是我们一直想要的效果吗？

![](/images/xiecheng/xiecheng_4.jpg)


还记得上一章中说到的吗？这个例子中delay和println两行代码紧密地写在协程之中，他们的执行也严格按照从上到下一行一行地顺序执行，但是这两行的代码却运行在完全不同的两个线程中，这就是我们想要的“既按照顺序的方式编写代码，又可以让代码在不同的线程顺序执行”的“顺序编写异步代码的效果”。顺序编写保证了逻辑上的直观性，协程的自动线程切换又保证了代码的非阻塞性。

![](/images/xiecheng/xiecheng_5.jpg)
那为什么协程中的delay函数没有在主线程中执行呢？而且执行完毕为什么还会自动地切回主线程呢？这是怎么做到的呢？我们可以来看一下delay函数的定义：

```
public suspend fun delay(timeMillis: Long) {...}
```
可以发现这个函数与正常的函数相比前面多了一个suspend关键字，这个关键字翻译过来就是“挂起”的意思，suspend关键字修饰的函数也就叫“挂起函数”。

关于挂起函数有个规定：挂起函数必须在协程或者其他挂起函数中被调用，换句话说就是挂起函数必须直接或者间接地在协程中执行。

关于挂起的概念大家不要理解错了，挂起的不是线程而是协程。遇到了挂起函数，协程所在的线程不会挂起也不会阻塞，但是协程被挂起了，就是说协程被挂起时当前协程与它所运行在的线程脱钩了。

线程继续执行其他代码去了，而协程被挂起等待着，等待着将来线程继续回来执行自己的代码。也就是协程中的代码对线程来说是非阻塞的，但是对协程自己本身来说是阻塞的。换句话说，协程的挂起阻塞的不是线程而是协程。

![](/images/xiecheng/xiecheng_6.jpg)

所以说，协程的挂起可以理解为协程中的代码离开协程所在线程的过程，协程的恢复可以理解为协程中的重新代码进入协程所在线程的过程。协程就是通过的这个挂起恢复机制进行线程的切换。

### 4. 线程切换
既然协程执行到了挂起函数会被挂起，那么是suspend关键字进行的线程切换吗？怎么指定切换到哪个线程呢？对此我们可以做一个简单的试验：

```
GlobalScope.launch(Dispatchers.Main) {
    println("Hello ${Thread.currentThread().name}")    
    test()
    println("End ${Thread.currentThread().name}")
}

suspend fun test(){
    println("World ${Thread.currentThread().name}")
}
```
执行结果为：Hello main -> World main -> End main，也就是说这个suspend函数仍然运行在主线程中，suspend并没有切换线程的作用。

实际上我们可以withContext方法来在suspend函数中进行线程的切换：

```
GlobalScope.launch(Dispatchers.Main) {
    println("Hello ${Thread.currentThread().name}")    
    test()
    println("End ${Thread.currentThread().name}")
}

suspend fun test(){
   withContext(Dispatchers.IO){
        println("World ${Thread.currentThread().name}")
   }
}
```
执行的结果为：Hello main -> World DefaultDispatcher-worker-1 -> End main，这说明我们的suspend函数的确运行在不同的线程之中了。就是说实际是上withContext方法进行的线程切换的工作，那么suspend关键字有什么用处呢？

其实，忽略原理只从使用上来讲，suspend关键字只起到了标志这个函数是一个耗时操作，必须放在协程中执行的作用。关于线程切换其实还有其他方法，但是withContext是最常用的一个，其他的如感兴趣可以自行了解。

### 5. 顺序执行与并发执行
这是上一章中演示回调地狱的代码：

```
//客户端顺序进行三次网络异步请求，并用最终结果更新UI
request1(parameter) { value1 ->
    request2(value1) { value2 ->
        request3(value2) { value3 ->
            updateUI(value3)            
        } 
    }              
}
```
![](/images/xiecheng/xiecheng_7.jpg)
我们试着用刚刚学到的协程的方式来改进这个代码：

```
//用协程改造回调代码
GlobalScope.launch(Dispatchers.Main) {
    //三次请求顺序执行
    val value1 = request1(parameter)
    val value2 = request2(value1)
    val value3 = request2(value2)
    //用最终结果更新UI
    updateUI(value3)
}

//requestAPI适配了Kotlin协程
suspend fun request1(parameter : Parameter){...}
suspend fun request2(parameter : Parameter){...}
suspend fun request3(parameter : Parameter){...}
```
前提是request相关的API已经改造成了适应协程的方式，并在内部进行了线程切换。这样代码看起来是不是整洁多了？没有了烦人的嵌套，所有的逻辑都体现在了代码的先后顺序上了，是不是一目了然呢？

### 5.2 并发执行

那么接下来实现一些有挑战性的东西：如果三次网络请求并不存在前后的依赖关系，也就是说三次请求要并发进行，但是最终更新UI要将三次请求的结果汇总才可以。这样的需求如果没有RxJava或Kotlin协程这种强大的工具支持，单靠自己编码实现的确是一个痛苦的过程。
![](/images/xiecheng/xiecheng_8.jpg)
不过Kotlin协程提供了一种简单的方案：async await方法。


```
//并发请求
GlobalScope.launch(Dispatchers.Main) {
    //三次请求并发进行
    val value1 = async { request1(parameter1) }
    val value2 = async { request2(parameter2) }
    val value3 = async { request3(parameter3) }
    //所有结果全部返回后更新UI
    updateUI(value1.await(), value2.await(), value3.await())
}

//requestAPI适配了Kotlin协程
suspend fun request1(parameter : Parameter){...}
suspend fun request2(parameter : Parameter){...}
suspend fun request3(parameter : Parameter){...}
```
上面的代码中我们用async方法包裹执行了suspend方法，接着在用到结果的时候使用了await方法来获取请求结果，这样三次请求就是并发进行的，而且三次请求的结果都返回之后就会切回主线程来更新UI。
### 5.3 复杂业务逻辑

实际开发遇到了串行与并行混合的复杂业务逻辑，那么我们当然也可以混合使用上面介绍的方法来编写对应的代码。比如这样的业务逻辑：request2和request3都依赖于request1的请求结果才能进行，request2和request3要并发进行，更新UI依赖request2和request3的请求结果。
![](/images/xiecheng/xiecheng_9.jpg)
这样的复杂业务逻辑，如果自己实现是不是感觉要被逼疯？来看看Kotlin协程给出的方案：


```
//复杂业务逻辑的Kotlin协程实现
GlobalScope.launch(Dispatchers.Main) {
    //首先拿到request1的请求结果
    val value1 = request1(parameter1)
    //将request1的请求结果用于request2和request3两个请求的并发进行
    val value2 = async { request2(value1) }
    val value3 = async { request2(value1) }
    //用request2和request3两个请求结果更新UI
    updateUI(value2.await(), value3.await())
}

//requestAPI适配了Kotlin协程
suspend fun request1(parameter : Parameter){...}
suspend fun request2(parameter : Parameter){...}
suspend fun request3(parameter : Parameter){...}
```
怎么样？发现没有，无论怎样的复杂业务逻辑，用Kotlin协程表达出来始终是从上到下整齐排列的四行代码，无任何耦合嵌套，有没有从中感受到Kotlin协程的这股化腐朽为神奇的神秘力量。

了解了Kotlin协程的用法之后，是不是迫不及待地想要在实际Android项目中使用它了？接下来我们来在项目中使用Kotlin协程的最佳实践。

### Coroutines+Retrofit+ViewModel+LiveData实现MVVM客户端架构

我们在前两章中讲解了Kotlin协程的基本用法和所解决的关键性问题，接下来让我们来看看在实际项目中该怎么使用Kotlin协程这一利器呢。接下来一起来将Kotlin协程与Jetpack中的架构组件结合起来搭建个简单的项目吧
要实现的功能其实很简单，界面由一个按钮和三个图片组成。每次按下刷新按钮，就都会从网络上获取三张图片显示到界面上。从网络上获取图片的时候刷新按钮变为不可用状态，刷新完成后按钮恢复可用状态

### 1. 添加依赖


```
//添加Retrofit网络库和gsonConverter的依赖，注意一定要2.6.0版本以上
implementation 'com.squareup.retrofit2:retrofit:2.7.0'
implementation 'com.squareup.retrofit2:converter-gson:2.7.0'
//添加Jetpack中架构组件的依赖，注意viewmodel要添加viewmodel-ktx的依赖
implementation "androidx.lifecycle:lifecycle-livedata:2.1.0"
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.1.0'
implementation "androidx.lifecycle:lifecycle-extensions:2.1.0"
//添加Glide的依赖用于图片加载
implementation 'com.github.bumptech.glide:glide:4.10.0'
```
这里需要注意的是retrofit版本要求2.6.0以上，因为2.6.0以上的retrofit对于Kotlin协程提供了不错的支持，用起来也更方便。另外添加ViewModel的依赖一定要添加Kotlin版本的，因为这个版本为我们提供了viewModelScope这个协程范围的支持，这让我们可以方便地将生命周期管理和网络请求自动取消的任务交给它。

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:layout_marginTop="10dp"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/button"
        android:text="refresh"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

    <ImageView
        android:id="@+id/imageView1"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:scaleType="centerCrop"
        android:layout_marginTop="10dp"/>

    <ImageView
        android:id="@+id/imageView2"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:scaleType="centerCrop"
        android:layout_marginTop="10dp"/>

    <ImageView
        android:id="@+id/imageView3"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:scaleType="centerCrop"
        android:layout_marginTop="10dp"/>
</LinearLayout>
```
界面没什么可说的，从上到下垂直排列的一个按钮和三个图片，一个LinearLayout全部搞定。

### 3. 编写网络层接口
首先来看一下我们要使用到的搜狗美图的api接口：https://api.ooopn.com/image/sogou/api.php?type=json

此接口每次随机返回一张图片的url地址，返回数据格式为：

```
{
    "code": "200",
    "imgurl": "https://img02.sogoucdn.com/app/a/100520113/20140811192414"
}
```
数据格式很简单，我们可以很容易地创建出对应的实体类：

```
data class ImageDataResponseBody(
    val code: String,
    val imgurl: String
)
```
接着我们可以先创建个network包来存放网络层相关的代码：

![](/images/xiecheng/xiecheng_10.jpg)

ApiService为我们网络接口的访问单例类，NetworkService为定义的网络接口：

```
import com.njp.coroutinesdemo.bean.ImageDataResponseBody
import retrofit2.http.GET
import retrofit2.http.Query

//网络接口
interface ApiService {

    //声明为suspend方法
    @GET("image/sogou/api.php")
    suspend fun getImage(@Query("type") type: String = "json"): ImageDataResponseBody
}
```

```
import com.njp.coroutinesdemo.bean.ImageDataResponseBody
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import retrofit2.create
import java.util.concurrent.TimeUnit

//网络层访问统一入口
object NetworkService {

    //retorfit实例，在这里做一些统一网络配置，如添加转换器、设置超时时间等
    private val retrofit = Retrofit.Builder()
        .client(OkHttpClient.Builder().callTimeout(5, TimeUnit.SECONDS).build())
        .baseUrl("https://api.ooopn.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    //网络层访问服务
    val apiService = retrofit.create<ApiService>()

}
```
### 4. 编写ViewModel和View层代码 
首先由于我们的项目中要对网络加载的状态进行监听，以此来进行对刷新按钮是否可点击状态的设置和错误信息的显示。所以我们可以编写一个LoadState类来作为网络加载状态信息的承载：


```
sealed class LoadState(val msg: String) {
    class Loading(msg: String = "") : LoadState(msg)
    class Success(msg: String = "") : LoadState(msg)
    class Fail(msg: String) : LoadState(msg)
}
```

这里使用了sealed类，sealed类是一种特殊的父类，它只允许内部继承，所以在与when表达式合用来判断状态时很适合。其中Fail状态必须指定错误信息，其他的状态信息可为空。我们可以将其与ImageDataResponseBody一起放在新建的bean包下：

![](/images/xiecheng/xiecheng_11.jpg)
接着我们来创建我们的ViewModel：


```
import androidx.lifecycle.MutableLiveData
import androidx.lifecycle.ViewModel
import com.xxx.coroutinesdemo.bean.LoadState
import com.xxx.coroutinesdemo.network.NetworkService

class MainViewModel : ViewModel() {

    //存放三张图片的url数据
    val imageData = MutableLiveData<List<String>>()
    //存放网路加载状态信息
    val loadState = MutableLiveData<LoadState>()

    //从网络加载数据
    fun getData() {...}

}
```
在其中放了两个LiveData作为数据，第一个存放三张图片的url数据，第二个就是我们的网络加载的状态信息啦。

在具体实现我们的getData具体的方法体之前，我们先实现一下我们Activity中的View层代码：

```
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.widget.Toast
import androidx.lifecycle.Observer
import androidx.lifecycle.ViewModelProviders
import com.bumptech.glide.Glide
import com.xxx.coroutinesdemo.R
import com.xxx.coroutinesdemo.bean.LoadState
import kotlinx.android.synthetic.main.activity_main.*

class MainActivity : AppCompatActivity() {

    private lateinit var viewModel: MainViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        //获取ViewModel
        viewModel = ViewModelProviders.of(this).get(MainViewModel::class.java)

        //对加载状态进行动态观察
        viewModel.loadState.observe(this, Observer {
            when (it) {
                is LoadState.Success -> button.isEnabled = true
                is LoadState.Fail -> {
                    button.isEnabled = true
                    Toast.makeText(this, it.msg, Toast.LENGTH_SHORT).show()
                }
                is LoadState.Loading -> {
                    button.isEnabled = false
                }
            }

        })

        //对图片Url数据进行观察
        viewModel.imageData.observe(this, Observer {
            //用Glide加载三张图片
            Glide.with(this)
                .load(it[0])
                .into(imageView1)
            Glide.with(this)
                .load(it[1])
                .into(imageView2)
            Glide.with(this)
                .load(it[2])
                .into(imageView3)
        })

        //点击刷新按钮来网络加载
        button.setOnClickListener {
            viewModel.getData()
        }
    }
}
```
这里使用了Kotlin为我们提供的直接引用xml中控件id的方式，这样可以避免编写findViewById代码。首先我们用ViewModelProviders将我们的MainViewModel注入MainActivity中，接着分别对MainViewModel中的两组数据进行观察并更新我们的UI。

加载状态为LoadState.Loading的时候我们要设置刷新按钮为不可用状态，LoadState.Success和LoadState.Fail两种状态要将其设置为可用状态。此外失败状态还有将错误信息通过Toast显示出来。图片url数据更新时我们就使用Glide将三张图片加载到三个ImageView上即可。接着为刷新按钮设置点击事件，直接调用MainViewModel的getData方法即可。

我们可以将这两个类放在同一个包中（如果有其他新的页面的话需要二级分包）：
![](/images/xiecheng/xiecheng_12.jpg)

### 5. 实现getData方法
接下来我们就具体地来实现一下最核心的getData方法：


```
fun getData() {
    viewModelScope.launch(CoroutineExceptionHandler { _, e ->
            //加载失败的状态
            loadState.value = LoadState.Fail(e.message ?: "加载失败")
        }) {
            //更新加载状态
            loadState.value = LoadState.Loading()

            //并发请求三张图片的数据
            val data1 = async { NetworkService.apiService.getImage() }
            val data2 = async { NetworkService.apiService.getImage() }
            val data3 = async { NetworkService.apiService.getImage() }
            //通过为LiveData设置新的值来触发更新UI
            imageData.value = listOf(data1.await(), data2.await(), data3.await()).map {
                it.imgurl
            }

            //更新加载状态
            loadState.value = LoadState.Success()
        }
}
```
首先我们用Jetpack组件提供给我们的viewModelScope开启一个协程，我们可以稍微看下这个viewModelScope：

```
/**
 * ...
 * This scope is bound to [Dispatchers.Main]
 */
val ViewModel.viewModelScope: CoroutineScope
        get() {...}

```
可以看到viewModelScope是通过Kotlin的拓展属性的方式添加到ViewModel上的，并且其所处的线程是主线程，所以我们可以放心地在其中更新UI的操作。并且其与ViewModel的声明周期绑定，我们在这个协程范围内的耗时操作会在其生命周期结束时自动取消，不用担心内存泄漏之类的性能问题。

而且我们在开启协程的时候为其指定了CoroutineExceptionHandler，所有在协程中出现的错误都将回调这个方法。在加载数据时我们调用了apiService的suspend方法，并通过async方式来实现并发数据请求，最后通过为LiveData设置新值的方式触发UI的更新。

但是目前只有一个页面，只要一个ViewModel，所以这样的写法不会有什么问题。但是当页面数量和ViewModel的数量多起来的时候，每一次网络请求都要写一些模板代码总是有些不舒服，所以接下来我们来对网络请求的代码进行进一步的优化。

可能遇到重复代码的时候大家一般的想法是创建一个BaseViewModel类，重复的模板代码写在这个基类中，接着所有我们的ViewModel继承这个BaseViewModel。

这样的做法的确是可行的，但是我们只有一个很小的功能需要抽象出来，可能基类中也就只有这么一个方法，而且如果你的项目已经成型的时候，这种做法会严重破坏项目结构，你需要手动更改所有ViewModel的父类，接着更改所有的对应方法。

Kotlin为ViewModel添加viewModelScope的做法值得我们借鉴，我们可以为ViewModel添加一个拓展方法，而不需要更改其自身的继承结构：


```
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.CoroutineExceptionHandler
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.launch

fun ViewModel.launch(
    block: suspend CoroutineScope.() -> Unit,
    onError: (e: Throwable) -> Unit = {},
    onComplete: () -> Unit = {}
) {
    viewModelScope.launch(CoroutineExceptionHandler { _, e -> onError(e) }) {
        try {
            block.invoke(this)
        } finally {
            onComplete()
        }
    }
}

```
我们可以新建一个ViewModelExt.kt文件，在其中为ViewModel编写一个launch方法。我们为方法设置了三个参数：


```
1. block：协程主体；
2. onError：错误回调；
3. onComplete：完成回调。
```


接着我们在方法体中调用了viewModelScope.launch方法，并把我们的协程主体传入，并在其CoroutineExceptionHandler中调用了我们的onError，在viewModelScope.launch中我们通过一个try{...}finally{...}块包裹了方法体，但是我们没有catch任何错误，所以这在保证了onComplete一定得到执行的同时也保证了onError可以接受到所有的错误。

接着我们使用新的方法来重写我们的MainViewModel：

是不是感觉简洁了许多呢？

整体项目结构：

![](/images/xiecheng/xiecheng_13.jpg)
### 一些不足
其实这个演示项目中还留下了一些坑，因为我们的重点是讲Kotlin协程的实际应用，有些坑就没有处理，在这里我提一下。网络加载错误不一定只有网络连接和超时等这些明显的错误，对于业务上的错误我们没有做进一步的处理，相信实际项目中网络接口的结构都类似这种三段式的结构：

```
{
    "code": 200,
    "data": {...},
    "msg": "OK"
}
```
那么我们可以定义一个包装类ResonseBody：

```
data class ResponseBody<T>(
    val code: Int,
    val msg: String,
    val data: T
)
```
接着建立一个独立的网络访问层Repository：

```
object Repository {

    //数据脱壳与错误预处理
    fun <T> preprocessData(responseBody: ResponseBody<T>): T {
        return if (responseBody.code == 200) responseBody.data else throw Throwable(responseBody.msg)
    }

    suspend fun getImageData(paramter: Paramter1): ImageData {
        //调用ApiService定义的接口方法
        val responseBody = ApiService.getImage(paramter)
        //返回处理后的数据
        return preprocessData<ImageData>(responseBody)
    }

    suspend fun getOtherData(paramter: Paramter2): OtherData {...}

    ...
}
```
这样在网络层就将所有可能遇到的错误处理完毕了，ViewModel层直接拿到的就是脱壳后的正确数据，也不需要额外处理这些业务错误了，因为这里throw的错误最终都会由我们的onError回调接收到，我们编写的launch方法可以完美的对其进行处理。