---
layout: post
title: "Architecture Components之LiveData"
date: 2019-02-12
description: "Architecture Components谷歌官方组件"
tag: Android
---
##  概述
- LiveData 是一个持有数据的类，它持有的数据是可以被观察者订阅的，当数据被修改时就会通知观察者。观察者可以是 Activity、Fragment、Service 等。
- LiveData 能够感知观察者的生命周期，只有当观察者处于激活状态（STARTED、RESUMED）才会接收到数据更新的通知，在未激活时会自动解注册观察者，以减少内存泄漏。
- LiveData 遵守应用程序组件的生命周期，以便 Observer 可以指定一个其应该遵守的 Lifecycle。
- 使用 LiveData 保存数据时，由于数据和组件是分离的，当组件重建时可以保证数据不会丢失。

## 优点
- 确保 UI 界面始终和数据状态保持一致。
- 没有内存泄漏，观察者绑定到 Lifecycle 对象并在其相关生命周期 destroyed 后自行解除绑定。
- 不会因为 Activity 停止了而奔溃，如 Activity finish 了，它就不会收到任何 LiveData 事件了。
- UI 组件只需观察相关数据，不需要停止或恢复观察，LiveData 会自动管理这些操作，因为 LiveData 可以感知生命周期状态的更改。
- 在生命周期从非激活状态变为激活状态，始终保持最新数据，如后台 Activity 在返回到前台后可以立即收到最新数据。
- 当配置发生更改（如屏幕旋转）而重建 Activity / Fragment，它会立即收到最新的可用数据。
- LiveData 很适合用于组件（Activity / Fragment）之间的通信。

## 带着问题看源码

### 问题1:
#### LiveData实现整体流程
### 问题2:
#### LiveData是通过观察者模式来观察数据变化，如何在观察中绑定生命周期的？
### 问题3:
#### LiveData如何实现不接收订阅之前的消息？
### 问题4，
#### LiveData具体应用

### 源码：

```
 @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        // 判断当前LifeCycle的状态
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            // 如果处于销毁状态，直接返回
            return;
        }
        // 将LifecycleOwner和Observer包装了一层
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        // 判断Observer有没有被添加过，如果没有则添加到一个HashMap中
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            // 如果添加过，抛出异常
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        // 将包装过的Observer添加到LifecycleRegistry
        owner.getLifecycle().addObserver(wrapper);
    }

```
#### 我们从最核心的方法ovserve开始，这个方法的调用说明观察者与被观察者的订阅关系成立，参数的传入是两个接口，第一个是LifecycleOwner，就是说观察者必须是要一个实现LifecycleOwner的类，我们的Activity以及Fragment等等都是默认实现了LifecycleOwner的，第二个参数是一个Observer，里面就一个抽象方法  void onChanged(@Nullable T t)，用于之后的事件回调

```
 if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
```
#### 首先一开始，先判断LifecycleOwner的目前状态是不是DESTROYED，如果是DESTROYED，直接return，在DESTROYED状态下绑定是没意义的


```
 LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
```
#### 然后生成一个LifecycleBoundObserver类，这个类把owner和observer绑定起来，然后mObservers.putIfAbsent(observer, wrapper)把observer存储在map里，

```
public V putIfAbsent(@NonNull K key, @NonNull V v) {
        Entry<K, V> entry = get(key);
        if (entry != null) {
            return entry.mValue;
        }
        put(key, v);
        return null;
    }
```
#### 这里就是把observer存储进来，如果之前已经存储过了就直接取出来，否则返回null，取出来后判断，是否重复observe过同一个被观察者对象，oberver过就return，否则就 owner.getLifecycle().addObserver(wrapper)添加进去

#### 接下来看看LifecycleBoundObserver

```
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
        @NonNull final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        @Override
        public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
               // 移除观察者，在这个方法中会移除生命周期监听并回调activeStateChanged 方法 removeObserver(mObserver);
                return;
            }
            activeStateChanged(shouldBeActive());
        }

        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }
    }
```

#### 继承于ObserverWrapper，ObserverWrapper等下再分析，LifecycleBoundObserver这个类，在其onStateChanged中，先判断状态是DESTROYED的话就直接把Observer移除然后return，然后调用activeStateChanged(shouldBeActive());这个方法是ObserverWrapper中的方法

```
private abstract class ObserverWrapper {
        final Observer<T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;

        ObserverWrapper(Observer<T> observer) {
            mObserver = observer;
        }
     
        ...

        void activeStateChanged(boolean newActive) {
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            mActive = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += mActive ? 1 : -1;
            // 如果现在第一次新增活跃的观察者，那么回调 onActive ，onActive 是个空方法
            if (wasInactive && mActive) {
                onActive();
            }
            // 如果现在没有活跃的观察者了，那么回调 onInactive ，onInactive 是个空方法
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
            // 向观察者发送 LiveData 的值
            if (mActive) {
                dispatchingValue(this);
            }
        }
}
```
#### 在其方法中， boolean wasInactive = LiveData.this.mActiveCount == 0;判断回调之前的状态是不是非活跃的， 
```
if (wasInactive && mActive) {
                onActive(); 
    
}
```
####                表示从非活跃状态变成活跃状态时，调用onActive()，onActive()是一个空方法，可供子类使用

```
if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
```
#### 表示从活跃状态变成非活跃状态，调用onInactive()，onInactive()也是一个空方法，可供子类使用


```
if (mActive) {
                dispatchingValue(this);
            }
```
#### 如果当前是活跃状态，接着调用dispatchingValue(this)；


```
private void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }
```
#### 其实就把mObservers中存的ObserverWrapper一个一个取出来调用considerNotify（）方法

```
private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        //noinspection unchecked
        observer.mObserver.onChanged((T) mData);
    }
```
#### 在其considerNotify（）中

```
 if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
```
#### 校验了observer的状态看是不是活跃的，如果不是活跃的，则调用observer.activeStateChanged(false);来停止通知，如果是活跃的，

```
 if (observer.mLastVersion >= mVersion) {
            return;
        }
```
#### 在这里，mVersion初始值是LiveData中的一个常量START_VERSION，也就是-1，而每次在setValue，或者postValue的时候，mVersion会+1

```
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }
```
#### 所以if (observer.mLastVersion >= mVersion)，这里就是为了拦截已经回调过的消息

### 具体应用

#### 1，因为LiveData本身就已经绑定了生命周期，所以可以把一些监听或者广播在其内部处理注册或者解绑的逻辑，避免内存泄露
```
public class LocationListener extends LiveData<Location> {
    public static LocationListener mListener;


    public static LocationListener get() {

        if (mListener == null) {
            synchronized (LocationListener.class) {
               if (mListener == null){
                   mListener = new LocationListener();
               }
            }
        }
        return mListener;

    }


    private LocationListener() {
        com.hellobike.mapbundle.LocationManager.getInstance().addLocationChangedListener(new LocationSource.OnLocationChangedListener() {
            @Override
            public void onLocationChanged(Location location) {
                setValue(location);
            }
        });

    }

    @Override
    protected void onActive() {
        super.onActive();
        com.hellobike.mapbundle.LocationManager.getInstance().startLocation(EVehicleContextHolder.getEVehicleContext().getApplication());
    }

    @Override
    protected void onInactive() {
        super.onInactive();
        com.hellobike.mapbundle.LocationManager.getInstance().stopLocation();
    }
}
```

#### 2，可利用Transformations的静态方法map或者switchMap进行转换
```
private final LiveData<Result<EVehicleBikeDetail>> bikeDetailLiveData = Transformations
            .switchMap(bikeDetailEvent, new Function<String, LiveData<Result<EVehicleBikeDetail>>>() {
                @Override
                public LiveData<Result<EVehicleBikeDetail>> apply(String input) {
                    return Transformations.map(findBikeRepository.getBikeDetailByBikeNo(input), new Function<Resource<EVehicleBikeDetail>, Result<EVehicleBikeDetail>>() {
                        @Override
                        public Result<EVehicleBikeDetail> apply(Resource<EVehicleBikeDetail> input) {
                            return Result.map(input);
                        }
                    });
                }
            });
```
#### 3，可以做成LiveDataBus替代EventBus等工具，可以不用导入任何第三方库进来，并且因为已经绑定了生命周期，还可以不用处理反注册逻辑

```
public final class LiveDataBus {

    private final Map<String, BusLiveData<Object>> mCacheBus;

    private LiveDataBus() {
        mCacheBus = new ArrayMap<>();
    }

    private static class SingletonHolder {
        private static final LiveDataBus DEFAULT_BUS = new LiveDataBus();
    }

    public static LiveDataBus get() {
        return SingletonHolder.DEFAULT_BUS;
    }

    public static MutableLiveData<Object> with(@Nullable String key) {
        return get().withInfo(key, Object.class, false);
    }

    public static <T> MutableLiveData<T> with(@Nullable String key, Class<T> type) {
        return get().withInfo(key, type, false);
    }

    public static <T> MutableLiveData<T> with(@Nullable String key, Class<T> type, boolean needCurrentDataWhenNewObserve) {
        return get().withInfo(key, type, needCurrentDataWhenNewObserve);
    }

    private <T> MutableLiveData<T> withInfo(String key, Class<T> type, boolean needData) {
        if (!mCacheBus.containsKey(key)) {
            mCacheBus.put(key, new BusLiveData<>(key));
        }
        BusLiveData<Object> data = mCacheBus.get(key);
        data.mNeedCurrentDataWhenFirstObserve = needData;
        return (MutableLiveData<T>) mCacheBus.get(key);
    }


    private static class BusLiveData<T> extends MutableLiveData<T> {

        // 比如BusLiveData 在添加 observe的时候，同一个界面对应的一个事件只能注册一次
        // 自己添加逻辑定制
        private String mEventType;

        //首次注册的时候，是否需要当前LiveData 最新数据
        private boolean mNeedCurrentDataWhenFirstObserve;

        private BusLiveData(String eventType) {
            this.mEventType = eventType;
        }

        //主动触发数据更新事件才通知所有Observer
        private boolean mIsStartChangeData = false;

        @Override
        public void setValue(T value) {
            mIsStartChangeData = true;
            super.setValue(value);
        }

        @Override
        public void postValue(T value) {
            mIsStartChangeData = true;
            super.postValue(value);
        }

        //添加注册对应事件type的监听
        @Override
        public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
            super.observe(owner, new ObserverWrapper<>(observer, this));
        }

        //数据更新一直通知刷新
        @Override
        public void observeForever(@NonNull Observer<T> observer) {
            super.observeForever(observer);
        }

        @Override
        public void removeObserver(@NonNull Observer<T> observer) {
            super.removeObserver(observer);
        }
    }

    private static class ObserverWrapper<T> implements Observer<T> {

        private Observer<T> observer;
        private BusLiveData<T> liveData;

        private ObserverWrapper(Observer<T> observer, BusLiveData<T> liveData) {
            this.observer = observer;
            this.liveData = liveData;
            //mIsStartChangeData 可过滤掉liveData首次创建监听，之前的遗留的值
            liveData.mIsStartChangeData = liveData.mNeedCurrentDataWhenFirstObserve;
        }

        @Override
        public void onChanged(@Nullable T t) {
            if (liveData.mIsStartChangeData) {
                if (observer != null) {
                    observer.onChanged(t);
                }
            }
        }
    }


}
```



























