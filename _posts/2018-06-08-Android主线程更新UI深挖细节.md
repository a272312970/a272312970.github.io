---
layout: post
title: "Android主线程更新UI深挖细节"
date: 2018-06-08
description: "Android主线程更新UI"
tag: Android
---

## 在Android的设计中，有一条很重要的原则，就是非UI线程不能更新UI，意思就是只要主线程是可以去修改UI的，这个对大多数开发者来说都是非常熟悉的，但是我今天发现了一个有趣的现象，并且根据这一现象去源码挖了一下源码看看这条原则是如何去实现的

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
                //这样是不会崩溃的
                TextView tv = findViewById(R.id.tv);
                tv.setText("啦啦啦啦");
            }
        }
        ).start();
    }
}

```
### 在这一串代码中，我试图开启一个子线程，在子线程中去更新UI，但是程序并没有崩溃，但是如果我在其中把线程休眠一段时间，程序马上就崩溃了

```
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
              //这样会崩溃
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                TextView tv = findViewById(R.id.tv);
                tv.setText("啦啦啦啦");
            }
        }
        ).start();
    }
```
### 崩溃信息是

```
android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
                                                       at android.view.ViewRootImpl.checkThread(ViewRootImpl.java:7834)
                                                       at android.view.ViewRootImpl.requestLayout(ViewRootImpl.java:1273)
```
### 意思就是说只有创造这个View的原始线程才能更新这个view，那创造这个view的原始线程肯定就是说的主线程了。那么为什么会有这个现象呢？根据崩溃日志，我们先确定抛异常的地方是在一个叫ViewRootImpl的requestLayout()方法里


## ViewRootImpl是什么？
### ViewRoot的实现类，view的三大流程（measure,layout,draw）均是通过ViewRoot来完成的，ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时创建ViewRootImpl对象并和DecorView建立联系，最主要是一点就是View的绘制是在这里面完成的

### 而上面的崩溃日志显示程序是在requestLayout的时候崩溃的

```
 @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```
### scheduleTraversals()里面包含了view的绘制方法，可以看出，在View绘制之前，都要通过checkThread()检查一下
```
 void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```
### 可以看出mThread != Thread.currentThread()是判断当前的线程是不是和mThread是一样的。那么这个mThread线程又是什么线程呢？

```
public ViewRootImpl(Context context, Display display) {
           
            //...省略其他代码
           mThread = Thread.currentThread();
            //...省略其他代码
}
```
### 这里可以看出mThread是在ViewRootImpl的构造方法里被赋值的，赋值的对象就是当前线程，那ViewRootImpl是在什么时候初始化的呢？初始化的时候又是在什么线程呢？上网查了一下，说ViewRootImpl是在Onresume之后初始化的，我们打开ActivityThread，然后找到handleResumeActivity()方法,

```
 final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
                // ......省略其他代码..........
  // TODO Push resumeArgs into the activity for consideration
        r = performResumeActivity(token, clearHide, reason);
        // ......
    // ......省略其他代码..........
 r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    r.activity.makeVisible();
                }
     // ......省略其他代码..........
     // ......
     }
```
### 有关代码就这么几行，首先点进去看看r.activity.makeVisible();

```
void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }
```
### ViewManager wm = getWindowManager();点进getWindowManager()可以发现这个wm其实对应的就是WindowManagerImpl,而WindowManager只是一个接口，他对应的实现类是WindowManagerImpl,这里可以看出WindowManager把DecorView加到窗口去了，我们再找到WindowManagerImpl的add方法

```
@Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
```
### 继续找mGlobal的addView方法

```
 public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
     // ......省略其他代码..........
      ViewRootImpl root;
      View panelParentView = null;
      root = new ViewRootImpl(view.getContext(), display);
                view.setLayoutParams(wparams);
                mViews.add(view);
                mRoots.add(root);
                mParams.add(wparams);
     // ......省略其他代码..........
            
    }
```
### 到这里就可以看出，ViewRootImpl已经创建成功了，而且可以看出，当前的线程就是主线程，所以上面ViewRootImpl构造函数里 mThread = Thread.currentThread();赋值的就是主线程。那怎么证明这个过程是在Activity的onresume()方法之后呢，再回到handleResumeActivity()里。

```
 final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume, int seq, String reason) {
    // ......省略其他代码..........
    // TODO Push resumeArgs into the activity for consideration
       r = performResumeActivity(token, clearHide, reason);
        // ......
    // ......省略其他代码..........
       r.activity.mVisibleFromServer = true;
        mNumVisibleActivities++;
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
     // ......省略其他代码..........
     // ......
     }
```
### 我们刚才证明r.activity.makeVisible();这行代码已经构建了一个ViewRootImpl了，他上面还有一行代码 r = performResumeActivity(token, clearHide, reason);其实这个就是onresume的回调，怎么证明呢？点进去

```
 public final ActivityClientRecord performResumeActivity(IBinder token,
            boolean clearHide, String reason) {
        // ......省略其他代码..........
        r.activity.performResume();
         // ......省略其他代码..........
    }
```
### 再点进Activity里面的performResume()方法

```
 final void performResume() {
  // ......省略其他代码..........
     mLastNonConfigurationInstances = null;

        mCalled = false;
        // mResumed is set by the instrumentation
        mInstrumentation.callActivityOnResume(this);
        if (!mCalled) {
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onResume()");
        }
         // ......省略其他代码..........
    }
```
### 再点进 mInstrumentation.callActivityOnResume(this);

```
 public void callActivityOnResume(Activity activity) {
        activity.mResumed = true;
        activity.onResume();
        
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    am.match(activity, activity, activity.getIntent());
                }
            }
        }
    }
```
### 在这里就可以看出来，activity.onResume();表示activity的onResume()已经回调了，这样就可以证明，ViewRootImpl的创建确实是在主线程并且是在onresume之后

## **而上面那个例子也就水落石出了，在 activity的oncreate里面开线程去更新UI只要不设置延迟或者线程休眠是不会报错的，因为oncreate是在onresume之前执行的，此时ViewRootImpl还没有构造，检查线程的方法也不会调用**

### 其实Android设置这条原则是因为UI线程是非同步不安全的，如果多线程都能修改UI的话，肯定会造成UI的数据错乱和调用混乱，然后UI线程也是不可能去做到线程同步，因为UI线程要处理的东西太多了，线程同步会造成效率低下甚至直接卡死，所以Android提供了Handler通信机制来处理这一问题。

### 最后，还有一点需要特殊说明的是，Toast.show()并不是所谓的更新UI操作，Toast本质上是一个window，跟activity是平级的，而checkThread只是Activity维护的View树的行为,使用，如果在子线程中执行Toast.show()是不会报任何异常的(但是要注意Looper.prepare()和Looper.loop());




