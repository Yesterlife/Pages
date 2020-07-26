---
layout: post
title: "数据之韵"
description: "浅析 LiveData 源码"
cover_url: https://i.loli.net/2020/05/17/xGcfpsC4w2OtXZI.png
cover_meta: illustration by [Waisshu](https://www.pixiv.net/artworks/76304928)
tags: 
  - Android
---

LiveData 作为一个具备生命周期感知能力的可观察数据存储器，有效解决了界面数据同步与生命周期持有对象释放的问题，避免内存泄漏，由于过于好用，使用起来不停地喊芜湖。

## 让生命周期持有者了解数据变化

### 添加监听器

接下来通过添加监听器的方法 `observer()` 入手，看看 LiveData 是如何管理生命周期的：

``` java
private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers =
            new SafeIterableMap<>();

@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    // 方法必须在主线程调用
    assertMainThread("observe");
  
    // 如果生命周期持有者已经被销毁，则直接返回，忽略所有操作
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
  
    // 将数据监听器包装为 wrapper，生命周期持有者与数据监听器被关联起来
    //
    // 如果已经在该 LiveData 绑定过数据监听器，则
    // 1. 若新关联的生命周期持有者与旧的不同，抛出异常，不可重复绑定
    // 2. 若新关联的生命周期持有者与旧的相同，则新的替代旧的，相当于无事发生
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
  
    // 让绑定所得的 wrapper（实现了 LifecycleObserver）监听生命周期持有者的生命周期
    owner.getLifecycle().addObserver(wrapper);
}
```

除此之外，LiveData 还有一个 `observerForever()` 方法，采用该方法添加的监听器不需要绑定生命周期持有者，LiveData 不需要判断生命周期持有者的活跃状态。

``` java
@MainThread
public void observeForever(@NonNull Observer<? super T> observer) {
    // 方法必须在主线程调用
    assertMainThread("observeForever");
  
    // 将数据监听器包装为 wrapper
    // 与 LifecycleBoundObserver 不同，AlwaysActiveObserver 不需要关联任何生命周期持有者
    //
    // 如果已经在该 LiveData 绑定过数据监听器，则
    // 1. 若该数据监听器原本与生命周期持有者关联，抛出异常
    // 2. 若该数据监听器原本已经添加，则新的替代旧的，相当于无事发生
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    wrapper.activeStateChanged(true);
}
```

从两个方法的代码中我们可以得出以下几点：

- 一个数据监听器至多与一个生命周期持有者关联，生命周期持有者与数据监听器的关系是一对多的关系

- 数据监听器如果关联生命周期持有者（即 `observer()` 方法添加），那数据监听器只有在关联的生命周期持有者处于活跃状态时才会收到数据更新信息

- 数据监听器如果不关联任何生命周期持有者（即 `observeForever()` 方法添加），那数据监听器能否收到数据更新信息与任何生命周期无关

### 移除监听器

监听器可以被添加，那么也可以被移除。

移除监听器的一种做法是调用 `removeObserver()` 方法，代码逻辑并不复杂。

``` java
@MainThread
public void removeObserver(@NonNull final Observer<? super T> observer) {
    // 方法必须在主线程调用
    assertMainThread("removeObserver");
  
    // 移除之后，需要进行处理，确保 LiveData 不再监听生命周期持有者的生命周期，防止内存泄漏
    ObserverWrapper removed = mObservers.remove(observer);
    if (removed == null) {
        return;
    }
    removed.detachObserver();
  
    // 移除的 wrapper 需要标记为非活跃状态，否则会影响 LiveData 的相关计数器，下文会提及
    removed.activeStateChanged(false);
}
```

也可以通过调用 `removeObservers()` 方法移除所有与生命周期持有者相关联的数据监听器。

``` java
@MainThread
public void removeObservers(@NonNull final LifecycleOwner owner) {
    // 方法必须在主线程调用
    assertMainThread("removeObservers");
  
    // 遍历移除
    for (Map.Entry<Observer<? super T>, ObserverWrapper> entry : mObservers) {
        if (entry.getValue().isAttachedTo(owner)) {
            removeObserver(entry.getKey());
        }
    }
}
```

### 如何记录生命周期持有者与数据监听器

从添加监听器的代码可以得出，用于将生命周期持有者与数据监听器相关联的 wrapper 对象，其类 `LifecycleBoundObserver` 实现了 `LifecycleObserver`，而不与生命周期持有者相关联的数据监听器也被包装为另一种类型的 wrapper 对象，其类为 `AlwaysActiveObserver`。两种 wrapper 对象的类实际上都实现了 `ObserverWrapper` 抽象类，那么看一看这部分的源码：

``` java
// LiveData 这一属性记录了当前绑定的生命周期持有者有多少个处于活跃状态
int mActiveCount = 0;

// 数据监听器与生命周期持有者相关联的包装
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull
    final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        // 可以看到每个生命周期持有者的活跃状态周期为 ON_START ~ ON_STOP
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
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

// 数据监听器不与任何生命周期持有者相关联的包装
private class AlwaysActiveObserver extends ObserverWrapper {

    AlwaysActiveObserver(Observer<? super T> observer) {
        super(observer);
    }

    @Override
    boolean shouldBeActive() {
        return true;
    }
}

private abstract class ObserverWrapper {
    final Observer<? super T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;

    ObserverWrapper(Observer<? super T> observer) {
        mObserver = observer;
    }

    abstract boolean shouldBeActive();

    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        mActive = newActive;

        // 判断改变之前的 LiveData 状态是否为未活跃状态
        // （即是否所有绑定的生命周期持有者均为未活跃状态）
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        LiveData.this.mActiveCount += mActive ? 1 : -1;
      
        // 若绑定的活跃生命周期持有者从无到有，调用 onActive()
        if (wasInactive && mActive) {
            onActive();
        }
      
        // 若绑定的活跃生命周期持有者从有到无，调用 onInactive()
        // 之所以在判断条件内不采用 wasInactive 而是重新比较，是考虑到多线程下不会频繁切换
        if (LiveData.this.mActiveCount == 0 && !mActive) {
            onInactive();
        }
      
        // 如果 ObserverWrapper 对应的生命周期持有者转为活跃状态，立即通知当前的值
        if (mActive) {
            dispatchingValue(this);
        }
    }
}
```

### 如何通知数据监听器发生数据变更

在 `ObserverWrapper.activeStateChanged()` 中调用了一个 `dispatchingValue()` 方法，作用是在生命周期持有者转为活跃状态时，通知 wrapper 对应的数据监听器发生了数据变更。

那么我们可以从 `dispatchingValue()` 方法看看数据是如何派发给数据监听器的。

``` java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    // 上一份数据是否仍在派发中
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
  
    // 开始派发数据
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false
        
        if (initiator != null) {
            // 如果传入参数不为空，说明是添加数据监听器引发的调用，
            // 那么仅需要将数据派发给新增的监听器即可
            considerNotify(initiator);
            initiator = null;
        } else {
            // 如果传入参数为空，说明是更改 LiveData 数据引发的调用，
            // 那么数据将派发给所有的监听器
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                // 如果中途该方法被其它地方再次调用，停止这轮派发
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated); // 上方停止这轮派发之后，由于数据已经更新，需要重新开始派发
    mDispatchingValue = false;
}

private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
  
    // 检测当前生命周期持有者是否处于活跃状态周期内，如果未处于周期内则设置其为未活跃状态
    // 这一步是避免生命周期更新后没能及时根据活跃状态进行操作
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
  
    // 在这一步对监听器进行回调
    observer.mObserver.onChanged((T) mData);
}
```

通过以上代码可以得出：

- 不一定每次数据的变更都会派发给所有处于活跃状态的数据监听器，如果两次数据变更时间间隔过短，上一次数据变更可能有部分监听器无法得知。

- 当有生命周期持有者的活跃状态发生改变时，只有与其相关联的数据监听器会被派发数据，与其它生命周期相关联的数据监听器不会被调用。

## 数据处理

既然 LiveData 的本质是一个可观察的数据存储器，那么少不了数据的读写操作。

### 获取 LiveData 数据

先看看 LiveData 所存储值的方法。

``` java
static final Object NOT_SET = new Object();

private volatile Object mData = NOT_SET;

@Nullable
public T getValue() {
    Object data = mData;
    if (data != NOT_SET) {
        //noinspection unchecked
        return (T) data;
    }
    return null;
}
```

可以看到，LiveData 的值存放在 `mData` 中，若 `mData` 不等于 `NOT_SET`（即 LiveData 还未初始化），则返回空值。

> 在 LiveData 中，`NOT_SET` 在效果上更偏向于一种非空的占位作用，因为 LiveData 所存储的值是可空的，尽管这一点在 `getValue()` 这一方法上无法得到体现。

### 修改 LiveData 数据

LiveData 类型的实例外部是无法修改数据的，我们可以通过看看一个可修改数据的子类：`MutableLiveData`。

实际 `MutableLiveData` 的逻辑非常简单，`LiveData` 本身具有修改数据的方法，而 `MutableLiveData` 将其对外暴露。

``` java
public class MutableLiveData<T> extends LiveData<T> {
    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```

先看看 `setValue()` 方法。

``` java
@MainThread
protected void setValue(T value) {
    // 方法必须在主线程调用
    assertMainThread("setValue");
  
    // 版本（即更新次数）自增
    mVersion++;
  
    // 修改数据后通过 dispatchingValue() 将数据进行派发
    mData = value;
    dispatchingValue(null);
}
```

逻辑上非常简单，`dispatchingValue()` 的实现前文也已经提及，但这一方法只能在主线程调用，如果要在其它线程修改 LiveData 的值，则需要调用 `postValue()` 方法。

``` java
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        // 判断数据暂存区是否已经存有数据
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
  
    // 如果数据暂存区已经存有数据，
    // 说明 mPostValueRunnable 已经被推送到主线程等待执行，不需要再次推送
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            // 设置完成数据后，重置数据暂存区
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //noinspection unchecked
        setValue((T) newValue);
    }
};
```

也就是说，`postValue()` 的实质是在主线程调用 `setValue()` 更新数据，但在 `mPostRunnable` 运行之前，无论调用多少次 `postValue()`，都只有最后一个数据会传递到 `setValue()` 中，可得：

- 调用 `postValue()` 修改数据不一定会派发到数据监听器，如果两次调用时间间隔过短，上一次数据变更可能有部分监听器无法得知。

- 调用 `postValue()` 不一定会更新 LiveData 的版本（即 `mVersion`）。

- 由于 `postValue()` 最终是采用在主线程调用 `setValue()` 更新数据（实际上无论由任何情况引起的派发，派发过程的代码都在主线程上），因此保证了数据监听器会在主线程被调用。

## 多数据源监听

很方便地，数据源可以直接返回 LiveData 让生命周期持有者监听，比如 Room，通过返回 LiveData 的方式，我们无需进行多次请求即可随时得知数据库的变化。

好用么？好用炸了！但有些情况下我们需要一个监听器同时监听两个或以上的 LiveData，比如同时监听本地和网络数据源，虽然可以对这些 LiveData 绑定同一个数据监听器，但毕竟这做法还是捞了点，而且想移除监听器也不方便，要是遇到如 DataBinding 等传递单个 LiveData 的情况就更难顶了。更好地方法是，可以通过 `MediatorLiveData` 将多个 LiveData 的数据变更事件整合到一起，当 `MediatorLiveData` 的数据源 LiveData 内部数据变动时，`MediatorLiveData` 的数据监听器也会接收到数据。

``` kotlin
val liveData = MediatorLiveData<String>()

val sourceA = MutableLiveData<Int>()
val sourceB = MutableLiveData<Int>()

// 添加数据源 LiveData，当数据源改变时，对应的 Observer 也会被调用
liveData.addSource(sourceA) { liveData.value = "get $it from A" }
liveData.addSource(sourceA) { liveData.value = "get $it from B" }

liveData.observe(lifecycleOwner, Observer {
    println(it)
})

sourceA.value = 1
sourceB.value = 2
```

```
get 1 from A
get 2 from B
```

### 添加数据源

为了了解 `MediatorLiveData` 是如何运作的，先看看 `addSource()` 方法内部做了哪些步骤。

``` java
@MainThread
public <S> void addSource(@NonNull LiveData<S> source, @NonNull Observer<? super S> onChanged) {
    // 将数据源 LiveData 与监听器进行关联，包装为一个 source，命名为 e
    //
    // 这部分操作与 LiveData 的 observe() 方法逻辑类似
    // 1. 若数据源关联的监听器与旧的相同，抛出异常，不可重复添加
    // 2. 若数据源关联的监听器与旧的不同，则新的代替旧的，旧有监听器无效化
    Source<S> e = new Source<>(source, onChanged);
    Source<?> existing = mSources.putIfAbsent(source, e);
    if (existing != null && existing.mObserver != onChanged) {
        throw new IllegalArgumentException(
                "This source was already added with the different observer");
    }
    if (existing != null) {
        return;
    }
  
    // 如果 MediatorLiveData 处于活跃状态（即存在关联的生命周期持有者中处于活跃状态），
    // 则将 e 这个 source 作为监听器绑定到数据源
    if (hasActiveObservers()) {
        e.plug();
    }
}
```

> `hasActiveObservers()` 是 `LiveData` 的一个方法，判断是否存在活跃状态的生命周期持有者，逻辑简单到炸。
>
> ``` java
> public boolean hasActiveObservers() {
>     return mActiveCount > 0;
> }
> ```

### 移除数据源

移除数据源的方法为 `removeSource()`。

``` java
@MainThread
public <S> void removeSource(@NonNull LiveData<S> toRemote) {
    Source<?> source = mSources.remove(toRemote);
  
    // 移除数据源后，将 source 与数据源取消绑定
    if (source != null) {
        source.unplug();
    }
}
```

### 关联数据源与数据源监听器

前面曾提到过，将 `source` 作为监听器绑定到数据源，实际便是 `Source` 实现了监听器的接口。

``` java
private static class Source<V> implements Observer<V> {
    final LiveData<V> mLiveData;
    final Observer<? super V> mObserver;
    int mVersion = START_VERSION;

    Source(LiveData<V> liveData, final Observer<? super V> observer) {
        mLiveData = liveData;
        mObserver = observer;
    }

    void plug() {
        mLiveData.observeForever(this);
    }

    void unplug() {
        mLiveData.removeObserver(this);
    }

    @Override
    public void onChanged(@Nullable V v) {
        // 比较数据源的版本，如果版本不一致说明数据源已经更新，此时调用数据源监听器
        if (mVersion != mLiveData.getVersion()) {
            mVersion = mLiveData.getVersion();
            mObserver.onChanged(v);
        }
    }
}
```

整个类的逻辑较为简单，但也可以总结出一些有帮助的信息：

- 数据源的监听器与任何生命周期没有关联，因此无论 `MediatorLiveData` 关联的那些生命周期持有者状态如何，数据源监听器都会在数据源更新时被调用。

- 前文提到过，监听器均会在主线程被调用，因此数据源监听器内仅需采用 `setValue()`，而无需采用 `postValue()`。

## 参考链接

- [Android Developers - LiveData Overview](https://developer.android.com/topic/libraries/architecture/livedata)

- [Android Code Search - LiveData](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:lifecycle/lifecycle-livedata-core/src/main/java/androidx/lifecycle/LiveData.java)

- [Android Code Search - MutableLiveData](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:lifecycle/lifecycle-livedata-core/src/main/java/androidx/lifecycle/MutableLiveData.java)

- [Android Code Search - MediatorLiveData](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:lifecycle/lifecycle-livedata/src/main/java/androidx/lifecycle/MediatorLiveData.java)