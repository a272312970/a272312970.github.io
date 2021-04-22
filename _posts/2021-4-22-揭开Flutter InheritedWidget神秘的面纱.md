---
layout: post
title: "揭开Flutter InheritedWidget神秘的面纱"
date: 2021-4-22
description: "Flutter InheritedWidget源码探析"
tag: Flutter 
--- 

#### 在Flutter中，InheritedWidget可以说是无所不在，很多状态管理框架，比如Bloc，Redux，Provider等等都是使用了InheritedWidget，并且我们在实际的开发中很多也会使用这个Widget，足以看清楚他的分量，但是我们在使用它的过程中脑子里肯定会有一些问题。
####  1.InheritedWidget是什么，可以做什么
####  2.context.dependOnInheritedWidgetOfExactType<T>();是怎么实现的，他跟context.findAncestorWidgetOfExactType的区别是什么
####  3.为什么说InheritedWidget高效
####  4.InheritedWidget是如何实现的。
####  5.context.dependOnInheritedWidgetOfExactType<T>()为什么只能获取最近的节点
带着这些问题，我们来分析一下源码

首先，按照官方的注解，InheritedWidget是一个允许去订阅改变他的子部件值的Widget。其实Inherited翻译过来有继承的意思，这个名字取的也挺有意思，因为他内部的实现确实有集成的意思，这个我们后面分析再说
```
an inherited widget that allows clients to subscribe to changes for subparts of the value.
```
InheritedWidget是一个抽象类，其实我们在使用InheritedWidget的时候，一般都会继承InheritedWidget，实现updateShouldNotify方法，然后定义一个of静态方法，比如：
```
class InheritDemoWidget extends InheritedWidget {
  final String value;

  InheritDemoWidget({Key key, @required this.value, Widget child})
      : super(key: key, child: child);

  static InheritDemoWidget of(BuildContext context) {
    return context.dependOnInheritedWidgetOfExactType<InheritDemoWidget>();
  }

  @override
  bool updateShouldNotify(InheritDemoWidget oldWidget) {
    return oldWidget.value != value;
  }
}
```
在子类中的调用

```
class ChildText extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Text('改变值:${context.dependOnInheritedWidgetOfExactType<InheritDemoWidget>().value}');
  }
}
```
从context.dependOnInheritedWidgetOfExactType<InheritDemoWidget>()；开始分析，点进源码

```
@override
  T dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object aspect}) {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[T];
    if (ancestor != null) {
      assert(ancestor is InheritedElement);
      return dependOnInheritedElement(ancestor, aspect: aspect) as T;
    }
    _hadUnsatisfiedDependencies = true;
    return null;
  }
```


再点进dependOnInheritedElement方法

```
  @override
  InheritedWidget dependOnInheritedElement(InheritedElement ancestor, { Object aspect }) {
    assert(ancestor != null);
    _dependencies ??= HashSet<InheritedElement>();
    _dependencies.add(ancestor);
    ancestor.updateDependencies(this, aspect);
    return ancestor.widget;
  }
```
其实就是返回了传进去的ancestor，也就是_inheritedWidgets里面取出来的那个值，_inheritedWidgets是一个Map<Type, InheritedElement>，里面保存了InheritedElement。
那这个map是怎么来的？为什么里面会保存InheritedElement。
其实在每一个element mounted的时候会调用一个_updateInheritance方法，

```
 @mustCallSuper
  void mount(Element parent, dynamic newSlot) {
    //... 
    _updateInheritance();
    //...
  }
```
该方法分两部分实现
1.在普通的element里面的实现是

```
 void _updateInheritance() {
    assert(_active);
    _inheritedWidgets = _parent?._inheritedWidgets;
  }
```
这也就是为什么刚开始说Inherited这个名字取的比较有意思，可以看出，在每一个element中都继承了父elemnt的_inheritedWidgets，每一个子类中都保存一份副本，该副本始终保存了所有父节点中所有的InheritedElement。

2.在InheritedElement中的实现是

```
  @override
  void _updateInheritance() {
    assert(_active);
    final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
    if (incomingWidgets != null)
      _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
    else
      _inheritedWidgets = HashMap<Type, InheritedElement>();
    _inheritedWidgets[widget.runtimeType] = this;
  }
```
这个除了继承父节点的_inheritedWidgets外，还把自身也加进map了，到此整个链路就完成了，所以该map里面始终保存着父节点所有的InheritedElement，但是由于是map保存的，如果是父节点有两个或更多的同一类型的InheritedElement，那么Map中只会保存最近的一个，其余的会被覆盖掉，这也是为什么我们使用context.dependOnInheritedWidgetOfExactType<T>();只会找到最近节点的原因。

这里回到dependOnInheritedElement方法，

```
 @override
  InheritedWidget dependOnInheritedElement(InheritedElement ancestor, { Object aspect }) {
    assert(ancestor != null);
    _dependencies ??= HashSet<InheritedElement>();
    _dependencies.add(ancestor);
    ancestor.updateDependencies(this, aspect);
    return ancestor.widget;
  }
```
这里除了返回InheritedElement，其实还又一部分逻辑，_dependencies其实就是保存在Set，里面保存了返回的那个InheritedElement，这个其实就是为后面的更新做二次判断，这里不赘述，有趣的是 ancestor.updateDependencies(this, aspect);这一行代码，我们点进去

```
@protected
  void updateDependencies(Element dependent, Object aspect) {
    setDependencies(dependent, null);
  }

```

```
 @protected
  void setDependencies(Element dependent, Object value) {
    _dependents[dependent] = value;
  }
```

```
  final Map<Element, Object> _dependents = HashMap<Element, Object>();
```

这里我们在找到的那个父节点中，也会有一个_dependents，这个是一个map，final Map<Element, Object>，


```
@override
  void updated(InheritedWidget oldWidget) {
    if (widget.updateShouldNotify(oldWidget))
      super.updated(oldWidget);
  }
```

这里先看一下InheritedElement里面，在update方法中，有一行似曾相识的代码，widget.updateShouldNotify(oldWidget)，这个是我们在使用InheritedWidget的时候经常会复写的一个方法，这里可以看出只有widget.updateShouldNotify(oldWidget)为true才会触发super.updated方法，update是该Element更新的触发方法，我们去在InheritedElement的父类ProxyElement去看看super.updated方法里面的逻辑，


```
  @protected
  void updated(covariant ProxyWidget oldWidget) {
    notifyClients(oldWidget);
  }
```
这里可以猜到，这个notifyClients方法就是触发_dependents里面所有元素渲染流水线更新的方法，所以这里就大概就清楚了，widget.updateShouldNotify(oldWidget)方法决定了_dependents里面的元素更新不更新的时机。我们接着分析源码。

在notifyClients方法在InheritedElement中的实现是

```
  @override
  void notifyClients(InheritedWidget oldWidget) {
         //...
    for (final Element dependent in _dependents.keys) {
        //...
      notifyDependent(oldWidget, dependent);
    }
  }
```
_dependents中的所有元素都会调用notifyDependent方法，接着看notifyDependent的实现。
```
  @protected
  void notifyDependent(covariant InheritedWidget oldWidget, Element dependent) {
    dependent.didChangeDependencies();
  }
```
注意看，这里调用了element的didChangeDependencies();方法，这个didChangeDependencies();方法是不是有点似曾相识，因为我们的state也有一个didChangeDependencies()方法，中间是不是有什么联系这个后面可以再分析，我们先看didChangeDependencies的实现，

```
 @mustCallSuper
  void didChangeDependencies() {
    //...
    markNeedsBuild();
  }
```
其实就是触发了渲染更新流水线，到这_dependents里面所有的元素都会触发更新，分析完之后，有没有感觉到其实就是一个观察者模式，巧妙的是，订阅的时机在你调用ontext.dependOnInheritedWidgetOfExactType<T>();的时候，InheritedWidget每次更新了就会按照updateShouldNotify的策略来决定通不通知订阅者更新。


### 延伸扩展：
#### 1.state和element的didChangeDependencies的联系。
其实我们的element和state的didChangeDependencies方法是完全不同的两个方法，但是在element的didChangeDependencies方法触发更新之后，往往会触发state的didChangeDependencies回调，众所周知，state的didChangeDependencies一般是在firstBuild也就是initState之后会回调，其实还有一个调用时机，就是在InheritedWidget更新之后，只要updateShouldNotify为true，依赖类就会触发，那是怎么触发的呢，我们可以看一下源码，在我们上面分析到触发  markNeedsBuild();的时候，渲染流水线触发势必就会触发element的performRebuild回调，在StatefulElement里面performRebuild的实现是

```
  @override
  void performRebuild() {
    if (_didChangeDependencies) {
      _state.didChangeDependencies();
      _didChangeDependencies = false;
    }
    super.performRebuild();
  }

```
在performRebuild回调后会判断_didChangeDependencies这个值是否为true，如果是则会回调_state.didChangeDependencies()，那_didChangeDependencies什么时候为true的呢，

```
 @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _didChangeDependencies = true;
  }

```
其实在之前触发，element的didChangeDependencies方法的时候，_didChangeDependencies就已经为true了，所以只要是element的didChangeDependencies触发，通常都会跟着触发state的didChangeDependencies方法

#### 2.context.dependOnInheritedWidgetOfExactType<T>()和context.findAncestorWidgetOfExactType的区别

context.findAncestorWidgetOfExactType也能找到父节点，那他是怎么实现的呢

```
  @override
  T findAncestorWidgetOfExactType<T extends Widget>() {
    assert(_debugCheckStateIsActiveForAncestorLookup());
    Element ancestor = _parent;
    while (ancestor != null && ancestor.widget.runtimeType != T)
      ancestor = ancestor._parent;
    return ancestor?.widget as T;
  }

```
可以看到，这个方法比较粗糙，从当前节点开始，往父节点递归，直到找到runtimeType跟泛型相同的时候直接返回，所以跟context.dependOnInheritedWidgetOfExactType<T>()相比，这个方法的效率会特别低，但是这个方法也并不是毫无用处，因为我们刚才知道context.dependOnInheritedWidgetOfExactType<T>()必须在父节点为InheritedWidget的时候才能找到，但是findAncestorWidgetOfExactType这个方法就抛掉了这个限制，可以直接暴力性的往上搜寻任何一个节点。















