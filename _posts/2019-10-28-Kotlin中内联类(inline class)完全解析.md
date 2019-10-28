---
layout: post
title: "Kotlin中内联类(inline class)完全解析"
date: 2019-10-28
description: "Kotlin"
tag: Kotlin 
--- 

## 概述
无论你是编写执行在云端的大规模数据流程程序还是低功耗手机运行的应用程序，大多数的开发者都希望他们的代码能够快速运行。现在，Kotlin的最新实验性的特性内联类允许创建我们想要的数据类型，并且还不会损失我们需要的性能!


### 强类型和普通值: 内联类的案例
星期一早上8点，在给自己倒了一杯新鲜的热气腾腾的咖啡之后，然后在项目管理系统中领到一份任务。上面写道：
#### 向新用户发送欢迎电子邮件 - 在注册后四天
因为已经编写好了邮件系统，您可以启动邮件调度程序的界面，正如你下面所看到的:

```
interface MailScheduler {
    fun sendEmail(email: Email, delay: Int)
}
```
看看这个函数，你知道你需要调用它…但是为了将电子邮件延迟4天，你会传递什么参数呢？

这个**delay**参数类型是**Int**. 所以我们仅仅知道这是一个Integer，但是我们并不知道它的单位是什么-你是应该传入4天呢? 或者它代表几个小时，如果是这样你传入的应该是96(24 * 4)。又或者它的单位是分钟、秒、毫秒…

我们应该如何去优化这个代码呢?

怎样才能让这个代码变得更好呢?

如果编译器能够强制指定正确的时间单位。例如，假设接收参数类型不是Int，让我们更新下interface中函数，让它接收一个强类型**Minutes**


```
val defaultDelay = Days(2)

fun send(email: Email) {
    mailScheduler.sendEmail(email, defaultDelay.toMinutes())
}
```

当我们可以充分利用类型系统时，我们提高了代码的健壮性。

但是开发者通常不会选择去为了单一普通值做个包装器类，而更多是通过传递Int、Float、Boolean这种基础类型。

为什么会这样呢?

通常，由于性能原因，我们反对创建这样的强类型。您可能还记得，JVM上的内存看起来像这样:



当我们创建一个基本类型的局部变量（即函数内定义的函数参数和变量）时 - 如Int、Float、Boolean - 这些值被存储在部分JVM 内存堆栈中。将这些基础类型的值存储在堆栈上所涉及的性能开销并不大。

在另一方面，每当我们实例化一个对象时，该对象实例就存储在JVM堆上了。我们在存储和使用对象实例时会有性能损失 - 堆分配和内存提取的性能代价很高。虽然看起来每个对象性能开销微不足道，但是累积起来，它对代码运行速度产生严重的影响。

如果我们能够在不受性能影响的情况下获得强类型系统的所有好处，那不是很好吗？

实际上，Kotlin新特性**inline class**就是为了解决这样的问题而设计的。

让我们一起来看看

### 内联类的介绍
内联类很容易去创建-仅仅需要在你定义的类前面加上inline关键字即可。

```
inline class Hours(val value: Int) {
    fun toMinutes() = Minutes(value * 60)
}
```
就是这样！这个类现在将作为您定义值的强类型，并且在许多情况下，它和常规非内联类相比性能成本几乎相同。

您可以像任何其他类一样实例化和使用内联类。您最终可能需要在代码中的某个位置引用里面包装的普通值 - 这个位置通常是在与另一个库或系统的边界处。 当然，在这一点上，您可以像通常使用任何其他类一样访问这个值。

### 您应该知道的关键术语
内联类包装基础类型的值。并且这个值也是有个类型的，我们把它称之为**基础类型**

### 为什么内联类可以高性能执行

那么，内联类为什么可以和普通类更好地执行呢?

你可以像这样去实例化一个内联类


```
val period = Hours(24)
```

…实际上该类并未在编译代码中实例化！事实上，就JVM而言，实际上相当于下面这样的代码…


```
int period = 24;
```
正如您所看到的，在此编译版本的代码中**没有Hours概念** - 它只是将基础值分配给int类型的变量！ 同样，当您使用内联类作为函数参数的类型时也是这样的:

```
fun wait(period: Hours) { /* ... */ }
```
…它可以有效地编译成如下这样…

```
void wait(int period) { /* ... */ }
```
因此，我们的代码中内联了基础类型和基础值。换句话说，编译后的代码只使用了int整数类型，因此我们避免了在堆内存上创建和访问对象的开销成本。

但是请等一下!

还记得Hours类有一个名为**toMinutes**（）的函数吗？因为编译后的代码使用的是int而不是Hours对象实例，因此想像一下调用toMinutes（）函数时会发生什么呢？

```
inline class Hours(val value: Int) {
    fun toMinutes() = Minutes(value * 60)
}
```
**Hours.toMinutes**()的编译代码如下所示：

```
public static final int toMinutes(int $this) {
	return $this * 60;
}
```
如果我们在Kotlin中调用**Hours(24).toMinutes()**，它可以有效地编译为**toMinutes(24)**.

没问题，确实可以像这样处理函数，但是类成员属性呢？如果我们希望**Hour**除了主要的基础值之外还包括其他一些数据，该怎么办？

一切事情都是有它的权衡的，那么这是其中之一 - **内联类除了基础值之外不能有任何其他成员属性**。让我们探讨其他的。

### 权衡和使用限制
现在我们知道内联类可以通过编译代码中的基础值来表示，我们已经准备好了解使用它们时应注意哪些使用限制。

首先，内联类必须包含一个基础值，这就意味它需要一个主构造器来接收
这个基础值，此外它必须是只读的(**val**)。你可以定义你想要的基础值变量名。

```
inline class Seconds()              // nope - needs to accept a value!
inline class Minutes(value: Int)    // nope - value needs to be a property
inline class Hours(var value: Int)  // nope - property needs to be read-only
inline class Days(val value: Int)   // yes!
inline class Months(val count: Int) // yes! - name it what you want
```
如果有需要，可以将该属性设为私有的，但构造函数必须是公有的。


```
inline class Years private constructor(val value: Int) // nope - constructor must be public
inline class Decades(private val value: Int)           // yes!
```
内联类中不能包含init block初始化块。我会在下一篇发表的文章中探讨内联类如何与Java进行互操作，这点将会彻底说明白。


```
inline class Centuries(val value: Int) {
	// nope - "Inline class cannot have an initializer block"
    init { 
        require(value >= 0)
    }
}
```
正如我们在上面发现的那样，除了一个基础值之外，我们的内联类主构造器不能包含其他任何成员属性。

```
// nope - "Inline class must have exactly one primary constructor parameter"
inline class Years(val count: Int, val startYear: Int)
```
但是呢，它的内部是可以拥有成员属性的,只要它们仅基于构造器中那个基础值计算，或者从可以静态解析的某个值或对象计算 - 来自单例，顶级对象，常量等。

```
object Conversions {
    const val MINUTES_PER_HOUR = 60    
}

inline class Hours(val value: Int) {
    val valueAsMinutes get() = value * Conversions.MINUTES_PER_HOUR
}
```

不允许类继承 - 内联类不能继承另一个类，并且它们不能被另一个类继承。
(Kotlin 1.3-M1在技术上确实允许内联类继承另一个类，但在即将发布的版本中会对此进行更正)


```
open class TimeUnit
inline class Seconds(val value: Int) : TimeUnit() // nope - cannot extend classes

open inline class Minutes(val value: Int) // nope - "Inline classes can only be final"
```

如果您需要将内联类作为子类型，那很好 - 您可以实现接口而不是继承基类。

```
interface TimeUnit {
	val value: Int
}

inline class Hours(override val value: Int) : TimeUnit  // yes!
```
内联类必须在顶级声明。嵌套/内部类不能内联的。

```
class Outer {
	 // nope - "Inline classes are only allowed on top level"
    inline class Inner(val value: Int)
}

inline class TopLevelInline(val value: Int) // yes!
```
目前，也不支持内联枚举类。
```
// nope - "Modifier 'inline' is not applicable to 'enum class'"
inline enum class TimeUnits(val value: Int) {
    SECONDS_PER_MINUTE(60),
    MINUTES_PER_HOUR(60),
    HOURS_PER_DAY(24)
}
```

### Type Aliases(类型别名) 与 Inline Classes(内联类)对比

因为它们都包含基础类型，所以内联类很容易与类型别名混淆。但是有一些关键的差异使它们在不同的场景下得以应用。

类型别名为基础类型提供备用名称。例如，您可以为String这样的常见类型添加别名，并为其指定在特定上下文中有意义的描述性名称，比如Username。Username类型的变量实际上是源代码和编译代码中String类型的变量同一个东西，只是不同名称而已。例如，您可以这样做：

```
typealias Username = String

fun validate(name: Username) {
    if(name.length < 5) {
        println("Username $name is too short.")
    }
}
```

注意到我们是可以在**name**上直接调用.**length**的，这是因为**name**实际上就是个**String**,尽管我们在声明参数类型的时候使用的是别名Username.

在另一面，内联类实际上是基础类型的包装器，因此当你需要使用基础值的时候，需要做拆箱操作。例如我们使用内联类来重写上面别名的例子:


```
inline class Username(val value: String)

fun validate(name: Username) {
    if (name.value.length < 5) {
        println("Username ${name.value} is too short.")
    }
}
```
注意到我们是必须这样**name.value**.length而不是**name.length**,我们必须解开这个包装器取出里面的值。

但是最大的区别在于与分配兼容性有关。内联类为你提供类型的安全性，类型别名则没有。 类型别名与其基础类型相同。例如，看如下代码：


```
typealias Username = String
typealias Password = String

fun authenticate(user: Username, pass: Password) { /* ... */ }

fun main(args: Array<String>) {
    val username: Username = "joe.user"
    val password: Password = "super-secret"
    authenticate(password, username)
}
```
在这种情况下，Username和Password仅仅是String另一个不同名称而已，甚至你可以将Username和Password任意调换位置。实际上，这正是我们在上面的代码中所做的 - 当我们调用authenticate（）函数时，即使我们将Username和Password位置弄反了，但编译器依然认为是合法的。

另一方面，如果你对上面同一个案例使用内联类，那么编译器将会很幸运告诉你这是不合法的：

```
inline class Username(val value: String)
inline class Password(val value: String)

fun authenticate(user: Username, pass: Password) { /* ... */ }

fun main(args: Array<String>) {
    val username: Username = Username("joe.user")
    val password: Password = Password("super-secret")
    authenticate(password, username) // <--- Compiler error here! =)
}
```

### ps：

我们知道在**java**中使用泛型的时候，无法通过泛型来得到**Class**，一般我们会将**Class**通过参数传过去。
在kotlin中一个内联函数（**inline**）可以被具体化（**reified**），这意味着我们可以得到使用泛型类型的Class。

```
inline fun <reified T> Retrofit.getService(): T {
    return create(T::class.java)
}
```
可以看到使用**T::class.java**就可以得到使用泛型类型的**Class**。





