---
layout: post
title: "Kotlin中关于Companion Object的那些事"
date: 2019-7-28
description: "Kotlin"
tag: Kotlin 
--- 
## 概述
Kotlin给Java开发者带来最大改变之一就是废弃了**static**修饰符。与Java不同的是在Kotlin的类中不允许你声明静态成员或方法。相反，你必须向类中添加**Companion**对象来包装这些静态引用: 差异看起来似乎很小，但是它有一些明显的不同。


首先，companion伴生对象是个实际对象的单例实例。你实际上可以在你的类中声明一个单例，并且可以像companion伴生对象那样去使用它。**这就意味着在实际开发中，你不仅仅只能使用一个静态对象来管理你所有的静态属性**! companion这个关键字实际上只是一个快捷方式，允许你通过类名访问该对象的内容(如果伴生对象存在一个特定的类中，并且只是用到其中的方法或属性名称，那么伴生对象的类名可以省略不写)。就编译而言，下面的**testCompanion**（）方法中的三行都是有效的语句。

```
class TopLevelClass {

    companion object {
        fun doSomeStuff() {
            ...
        }
    }

    object FakeCompanion {
        fun doOtherStuff() {
            ...
        }
    }
}

fun testCompanion() {
    TopLevelClass.doSomeStuff()
    TopLevelClass.Companion.doSomeStuff()
    TopLevelClass.FakeCompanion.doOtherStuff()
}
```

为了兼容的公平性，companion关键字还提供了更多选项，尤其是与Java互操作性相关选项。果您尝试在Java类中编写相同的测试代码，调用方式可能会略有不同：


```
public void testCompanion() {
    TopLevelClass.Companion.doSomeStuff();
    TopLevelClass.FakeCompanion.INSTANCE.doOtherStuff();
}
```

区别在于: Companion作为Java代码中静态成员开放(实际上它是一个对象实例，但是由于它的名称是以大写的**C**开头，所以有点存在误导性)，而FakeCompanion引用了我们的第二个单例对象的类名。在第二个方法调用中，我们需要使用它的INSTANCE属性来实际访问Java中的实例(你可以打开IntelliJ IDEA或AndroidStudio中的"Show Kotlin Bytecode"菜单栏，并点击里面"Decompile"按钮来查看反编译后对应的Java代码)

在这两种情况下(不管是Kotlin还是Java)，使用伴生对象**Companion**类比FakeCompanion类那种调用语法更加简短。此外，由于Kotlin提供一些注解，可以让编译器生成一些简短的调用方式，以便于在Java代码中依然可以像在Kotlin中那样简短形式调用。

@JvmField注解，例如告诉编译器不要生成**getter**和**setter**,而是生成Java中成员。在伴生对象的作用域内使用该注解标记某个成员，它产生的副作用是标记这个成员不在伴生对象内部作用域，而是作为一个Java最外层类的静态成员存在。从Kotlin的角度来看，这没有什么太大区别，但是如果你看一下反编译的字节代码，你就会注意到伴生对象以及他的成员都声明和最外层类的静态成员处于同一级别。

另一个有用的注解 **@JvmStatic**.这个注解允许你调用伴生对象中声明的方法就像是调用外层的类的静态方法一样。但是需要注意的是：在这种情况下，方法不会和上面的成员一样移出伴生对象的内部作用域。因为编译器只是向外层类中添加一个**额外的静态方法**，然后在该方法内部又委托给伴生对象。

一起来看一下这个简单的Kotlin类例子:


```
class MyClass {
    companion object {
        @JvmStatic
        fun aStaticFunction() {}
    }
}
```

这是相应编译后的Java简化版代码:


```
public class MyClass {
    public static final MyClass.Companion Companion = new MyClass.Companion();
    fun aStaticFunction() {//外层类中添加一个额外的静态方法
        Companion.aStaticFunction();//方法内部又委托给伴生对象的aStaticFunction方法
    }
    public static final class Companion {
         public final void aStaticFunction() {}
    }
}
```
这里存在一个非常细微的差别，但在某些特殊的情况下可能会出问题。例如，考虑一下Dagger中的module(模块)。当定义一个Dagger模块时，你可以使用静态方法去提升性能，但是如果你选择这样做，如果您的模块包含静态方法以外的任何内容，则编译将失败。由于Kotlin在类中既包含静态方法，也保留了静态伴生对象，因此无法以这种方式编写仅仅包含静态方法的Kotlin类。

但是不要那么快放弃! 这并不意味着你不能这样做，只是它需要一个稍微不同的处理方式：在这种特殊的情况下，**你可以使用Kotlin单例(使用object对象表达式而不是class类)替换含有静态方法的Java类并在每个方法上使用@JvmStatic注解**。如下例所示：在这种情况下，生成的字节代码不再显示任何伴生对象，静态方法会附加到类中。



```
@Module
object MyModule {

    @Provides
    @Singleton
    @JvmStatic
    fun provideSomething(anObject: MyObject): MyInterface {
        return myObject
    }
}
```
这又让你再一次明白了伴生对象仅仅是单例对象的一个特例。但它至少表明与许多人的认知是相反的，**你不一定需要一个伴生对象来维护静态方法或静态变量**。你甚至根本不需要一个对象来维护，只要考虑顶层函数或常量：它们将作为静态成员被包含在一个自动生成的类中(默认情况下，例如MyFileKt会作为MyFile.kt文件生成的类名，一般生成类名以Kt为后缀结尾)

我们有点偏离这篇文章的主题了，所以让我们继续回到伴生对象上来。现在你已经了解了伴生对象实质就是对象，也应该意识到它开放了更多的可能性，例如继承和多态。

这意味着你的伴生对象并不是没有类型或父类的匿名对象。它不仅可以拥有父类，而且它甚至可以实现接口以及含有对象名。它不需要被称为companion。这就是为什么你可以这样写一个Parcelable类：


```
class ParcelableClass() : Parcelable {

    constructor(parcel: Parcel) : this()

    override fun writeToParcel(parcel: Parcel, flags: Int) {}

    override fun describeContents() = 0

    companion object CREATOR : Parcelable.Creator<ParcelableClass> {
        override fun createFromParcel(parcel: Parcel): ParcelableClass = ParcelableClass(parcel)

        override fun newArray(size: Int): Array<ParcelableClass?> = arrayOfNulls(size)
    }
}
```
这里, 伴生对象名为CREATOR，它实现了Android中的Parcelable.Creator接口，允许遵守Parcelable约定，同时保持比使用@JvmField注释在伴随对象内添加Creator对象更直观。Kotlin中引入了@Parcelize注解，以便于可以获得所有样板代码，但是在这不是重点…

为了使它变得更简洁，如果你的伴生对象可以实现接口，它甚至可以使用Kotlin中的代理来执行此操作：


```
class MyObject {
    companion object : Runnable by MyRunnable()
}
```
这将允许您同时向多个对象中添加静态方法！请注意，伴生对象在这种情况下甚至不需要作用域体，因为它是由代理提供的。

最后但同样重要的是，你可以为伴生对象定义扩展函数! 这就意味着你可以在现有的类中添加静态方法或静态属性，如下例所示:

```
class MyObject {

    companion object

    fun useCompanionExtension() {
        someExtension()
    }

}

fun MyObject.Companion.someExtension() {}//定义扩展函数
```
