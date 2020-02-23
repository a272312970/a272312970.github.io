---
layout: post
title: "闲谈Flutter中的Hot Reload"
date: 2020-2-23
description: "Flutter Hot Reload"
tag: Flutter 
--- 
Flutter开发一大优势便是热重载，热重载是指，在不中断 App 正常运行的情况下，动态注入修改后的代码片段。而这一切的背后，离不开 Flutter 所提供的运行时编译能力。这里我们来深入理解 Flutter 的热重载实现原理

### JIT
JIT（Just In Time），指的是即时编译或运行时编译，在 Debug 模式中使用，可以动态下发和执行代码，启动速度快，但执行性能受运行时编译影响。
![](/images/hotreload/hotreload_1.jpg)
### AOT
AOT（Ahead Of Time），指的是提前编译或运行前编译，在 Release 模式中使用，可以为特定的平台生成稳定的二进制代码，执行性能好、运行速度快，但每次执行均需提前编译，开发调试效率低。
![](/images/hotreload/hotreload_2.jpg)
可以看到，Flutter 提供的两种编译模式中，AOT 是静态编译，即**编译成设备可直接执行的二进制码**；而 JIT 则是动态编译，即将 Dart 代码编译成中间代码（Script Snapshot），在**运行时设备需要 Dart VM 解释执行**。
而热重载之所以只能在 Debug 模式下使用，是因为 Debug 模式下，Flutter 采用的是 JIT 动态编译（而 Release 模式下采用的是 AOT 静态编译）。JIT 编译器将 Dart 代码编译成可以运行在 Dart VM 上的 Dart Kernel，而 Dart Kernel 是可以动态更新的，这就实现了代码的实时更新功能，原理如下图。
![](/images/hotreload/hotreload_3.jpg)
总体来说，完成热重载的可以分为扫描工程改动、增量编译、推送更新、代码合并、Widget 重建 5 个步骤。

1. 工程改动。热重载模块会逐一扫描工程中的文件，检查是否有新增、删除或者改动，直到找到在上次编译之后，发生变化的 Dart 代码；
2. 增量编译。热重载模块会将发生变化的 Dart 代码，通过编译转化为增量的 Dart Kernel 文件；
3. 推送更新。热重载模块将增量的 Dart Kernel 文件通过 HTTP 端口，发送给正在移动设备上运行的 Dart VM；
4. 代码合并。Dart VM 会将收到的增量 Dart Kernel 文件，与原有的 Dart Kernel 文件进行合并，然后重新加载新的Dart Kernel 文件；
5. Widget 重建。在确认 Dart VM 资源加载成功后，Flutter 会将其 UI 线程重置，通知 Flutter Framework 重建 Widget。

可以看到，Flutter 提供的热重载在收到代码变更后，并不会让 App 重新启动执行，而只会触发 Widget 树的重新绘制，因此可以保持改动前的状态，这就大大节省了调试复杂交互界面的时间。

比如，我们需要为一个视图栈很深的页面调整 UI 样式，若采用重新编译的方式，不仅需要漫长的全量编译时间，而为了恢复视图栈，也需要重复之前的多次点击交互，才能重新进入到这个页面查看改动效果。但如果是采用热重载的方式，不仅没有编译时间，而且页面的视图栈状态也得以保留，完成热重载之后马上就可以预览 UI 效果了，相当于进行了局部界面刷新

### 不支持热重载的场景
Flutter 提供的亚秒级热重载一直是开发者的调试利器。通过热重载，我们可以快速修改 UI、修复 Bug，无需重启应用即可看到改动效果，从而大大提升了 UI 调试效率。

不过，Flutter 的热重载也有一定的局限性。因为涉及到状态保存与恢复，所以并不是所有的代码改动都可以通过热重载来更新。以下是Flutter开发中几个不支持热重载的典型场景：

- 代码出现编译错误；
- Widget 状态无法兼容；
- 全局变量和静态属性的更改；
- main 方法里的更改；
- initState 方法里的更改；
- 枚举和泛类型更改。

我们就具体看看这几种场景的问题，应该如何解决吧！

### 代码出现编译错误
当代码更改导致编译错误时，热重载会提示编译错误信息。比如下面的例子中，代码中漏写了一个反括号，在使用热重载时，编译器直接报错,在这种情况下，只需更正上述代码中的错误，就可以继续使用热重载。
### Widget 状态无法兼容

当代码更改会影响 Widget 的状态时，会使得热重载前后 Widget 所使用的数据不一致，即应用程序保留的状态与新的更改不兼容。

这时，热重载也是无法使用的。比如下面的代码中，我们将某个类的定义从 StatelessWidget 改为 StatefulWidget 时，热重载就会直接报错，如下所示。

```

//改动前
class MyWidget extends StatelessWidget {
  Widget build(BuildContext context) {
    return GestureDetector(onTap: () => print('T'));
  }
}
 
//改动后
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => MyWidgetState();
}

class MyWidgetState extends State<MyWidget> { /*...*/ }
```
当遇到这种情况时，我们需要重启应用，才能看到更新后的程序的运行效果。
### 全局变量和静态属性的更改

在 Flutter 中，全局变量和静态属性都被视为状态，在第一次运行应用程序时，会将它们的值设为初始化语句的执行结果，因此在热重载期间不会重新初始化。

比如下面的代码中，我们修改了一个静态 Text 数组的初始化元素。虽然热重载并不会报错，但由于静态变量并不会在热重载之后初始化，因此这个改变并不会产生效果，代码如下。

```

//改动前
final sampleText = [
  Text("T1"),
  Text("T2"),
  Text("T3"),
  Text("T4"),
];
 
//改动后
final sampleText = [
  Text("T1"),
  Text("T2"),
  Text("T3"),
  Text("T10"),    //改动点
];
```
如果需要更改全局变量和静态属性的初始化语句，需要重启应用才能查看更改效果。

### main 方法里代码更改
在 Flutter 中，由于热重载之后只会根据原来的根节点重新创建控件树，因此 main 函数的任何改动并不会在热重载后重新执行。所以，如果我们改动了 main 函数体内的代码，是无法通过热重载看到更新效果的。


```

//更新前
class MyAPP extends StatelessWidget {
@override
  Widget build(BuildContext context) {
    return const Center(child: Text('Hello World', textDirection: TextDirection.ltr));
  }
}
 
void main() => runApp(new MyAPP());
 
//更新后
void main() => runApp(const Center(child: Text('Hello, 2019', textDirection: TextDirection.ltr)));
```
由于 main 函数并不会在热重载后重新执行，因此以上改动是无法通过热重载查看更新的。
### initState 方法里代码更改
在热重载时，Flutter 会保存 Widget 的状态，然后重建 Widget。而 initState 方法是 Widget 状态的初始化方法，这个方法里的更改会与状态保存发生冲突，因此热重载后不会产生效果。

例如，在下面的例子中，我们将计数器的初始值由 10 改为 100，代码如下：


```
//更改前
class _MyHomePageState extends State<MyHomePage> {
  int _counter;
  @override
  void initState() {
    _counter = 10;
    super.initState();
  }
  ...
}
 
//更改后
class _MyHomePageState extends State<MyHomePage> {
  int _counter;
  @override
  void initState() {
    _counter = 100;
    super.initState();
  }
  ...

```
### 枚举和泛型类型更改
在 Flutter 中，枚举和泛型也被视为状态，因此对它们的修改也不支持热重载。

比如在下面的代码中，我们将一个枚举类型改为普通类，并为其增加了一个泛型参数，代码如下。


```
//更改前
enum Color {
  red,
  green,
  blue
}
 
class C<U> {
  U u;
}
 
//更改后
class Color {
  Color(this.r, this.g, this.b);
  final int r;
  final int g;
  final int b;
}
 
class C<U, V> {
  U u;
  V v;

```

### Hot Reload 与 Hot Restart
针对上面不能使用 Hot Reload 的情况，就需要使用 Hot Restart。Hot Restart 可以完全重启您的应用程序，但却不用结束调试会话。

对于Android Studio来说， 执行 Hot Restart无需 stop操作，再Run 一下，就是 Hot Restart。

对于VS Code 来说，打开命令面板，输入 **Flutter: Hot Restart ** 或者 直接快捷键 Ctrl+F5，就可以使用 Hot Restart。

### 总结

Flutter 的热重载是基于 JIT 编译模式的代码增量同步。由于 JIT 属于动态编译，能够将 Dart 代码编译成生成中间代码，让 Dart VM 在运行时解释执行，因此可以通过动态更新中间代码实现增量同步。

热重载的流程可以分为 5 步，包括：扫描工程改动、增量编译、推送更新、代码合并、Widget 重建。Flutter 在接收到代码变更后，并不会让 App 重新启动执行，而只会触发 Widget 树的重新绘制，因此可以保持改动前的状态，大大缩短了从代码修改到看到修改产生的变化之间所需要的时间。

另一方面，由于涉及到状态的保存与恢复，涉及状态兼容与状态初始化的场景，热重载是无法支持的，如改动前后 Widget 状态无法兼容、全局变量与静态属性的更改、main 方法里的更改、initState 方法里的更改、枚举和泛型的更改等。

可以发现，热重载提高了调试 UI 的效率，非常适合写界面样式这样需要反复查看修改效果的场景。但由于其状态保存的机制所限，热重载本身也有一些无法支持的边界。

如果你在写业务逻辑的时候，不小心碰到了热重载无法支持的场景，也不需要进行漫长的重新编译加载等待，只要点击位于工程面板左下角的热重启（Hot Restart）按钮，就可以以秒级的速度进行代码重新编译以及程序重启了，同样也很快。