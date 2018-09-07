---
layout: post
title: "ThreadLocal的源码分析以及应用"
date: 2018-2-5
description: "ThreadLocal的源码分析"
tag: Android
---   
## ThreadLocal是java包中的一个类，在安卓中的应用的最经典的例子就是在Looper的启动过程中，保存了Looper, 之前知道ThreadLocal的特性是可以把变量存在当前的线程中，其他的线程去取的话是取不出来的,但是ThreadLocal到底是个什么样的存在呢，实现的原理是什么呢，今天从源码的角度分析一遍

### 首先看它的set方法
```
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

```
### 可以看到，set一个值的时候是通过getMap得到一个ThreadLocalMap，没有的话，就新建一个ThreadLocalMap，然后通过ThreadLocalMap把值设置进去


```
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

### getMap中其实得到的是Thread中的一个变量threadLocals


```
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

### 而在Thread中threadLocals其实就是一个ThreadLocalMap，而且threadLocals的默认值就是一个null，现在让我们来看一下ThreadLocalMap到底是个什么东西
 
```
static class ThreadLocalMap {

       ................
       
        static class Entry extends WeakReference<ThreadLocal<?>> {
          
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        ........
}
```
### ThreadLocalMap是ThreadLocal的一个内部类，可以看成一个定制化的HashMap，其中有一个继承ThreadLocalMap的内部类Entry，其实保存的值就是放在Entry里面的,所以ThreadLocal的set值的时候，首先通过getMap（）得到当前线程的threadLocals，然后把值set到了当前线程的成员变量threadLocals的Entry中了，如果threadLocals的值为null，说明没有初始化过，就通过createMap(t, value)，新建一个threadLocals，然后把值set进去

```
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

### 再来看看ThreadLocal的get方法


```
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
### 可以看出，get方法中首先取出当前线程的ThreadLocalMap即threadLocals值，如果threadLocals不为null就把threadLocals中的值取出来返回，否则的话则通过setInitialValue()方法返回一个默认值，那么setInitialValue()这个方法返回的默认值是什么呢？

```
  private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
### 可以看出通过initialValue()方法给个值然后set到ThreadLocalMap中去

```
  protected T initialValue() {
        return null;
    }
```
### 但是，其实看源码，initialValue()这个值默认返回的就是一个null,所以，如果没有set的话，ThreadLocal返回的其实是一个null，并且是一个protect方法，所以我们在创建ThreadLocal对象的时候可以重写此方法。

### 实际中的应用，比如现在我们要创建一个数据库的连接类，创建对象的时候我们第一要考虑到的是多线程问题，这个我们一般是通过把构造方法私有，然后定义静态方法提供对象，并在提供对象的时候加锁。比如：

```
  public static class DBConnection{
        public static DBConnection mDBConnection;
        private DBConnection(){}
        public static DBConnection getConnection() {
            if(mDBConnection == null){
                synchronized (DBConnection.class){
                    if(mDBConnection == null){
                        mDBConnection = new DBConnection();
                    }
                }
            }
            return mDBConnection;
        }
    }
 

```
### 这样写可以保证线程安全，并且全局单例，但是众所周知这样写因为加锁的缘故对性能效率会有所影响，那么有更好的方法没，这里就可以用到我们的ThreadLocal了，因为每个线程都会在自己的线程内创建一个DBConnection对象保存起来，所以线程访问的都是自己线程内的对象，不存在线程安全问题。比如：

```
private static ThreadLocal<DBConnection> connectionHolder = new ThreadLocal<DBConnection>() {
    @Override
    public DBConnection initialValue() {
        return DBManager.getConnection(DB_URL);
    }
};
 
public static DBConnection getConnection() {
    return connectionHolder.get();
}
```


## 总结：
### 1，ThreadLocal的特性是在本线程set的东西，只有在本线程中才能get出来。
### 2，ThreadLocal之所以有这个特性是因为ThreadLocal把set的值存到了每一个线程Thread类的成员变量threadLocals中，而每次取值的时候都是先get到当前线程Thread，然后得到threadLocals取出值
### 3，ThreadLocal只有在set或者get的时候才会去创建ThreadLocalMap对象
### 4，ThreadLocal在本线程中如果没有set直接get是默认返回null的，但是可以重写initialValue()改变返回的默认值
### 5，因为ThreadLocal在存取变量的时候会在每个线程创建一个变量的副本，所以实际使用过程中需要考虑资源的消耗










