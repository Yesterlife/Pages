---
layout: post
title: "寻忆"
description: "浅析 LeakCanary 核心机制"
cover_url: https://i.loli.net/2020/11/24/yIbFlQMpqjCvdrG.png
cover_meta: illustration by [Someya Mai](https://www.pixiv.net/artworks/71306760)
tags: 
  - Android
---

> 本文仅对核心进行浅析，因此不说明堆栈数据解析与展示的相关功能。
{:.side-note}

LeakCanary 提供了一个在开发阶段方便地寻找内存泄漏问题的方式，通过它可以快速定位发生内存泄漏的组件，并获得其完整的引用链。

虽然 LeakCanary 的功能非常强大，但其核心功能的源码却意料之外简洁易懂。

> 金丝雀（Canary）是一种对一氧化碳极为敏感的鸟类，17 世纪开始，英国的矿工们就将金丝雀带入矿井中，一旦金丝雀发生异样，则矿工们可提前感知危险到来，以迅速撤出矿井。
>
> 因此金丝雀具有报警器的含义。而在现在的软件开发中，该词也通常用于表示最新的测试版本。

## 原理

LeakCanary 通过监听各个具备生命周期组件的生命周期状态，并通过弱引用记录生命周期已进入销毁阶段的组件，当保留对象超过阈值时，LeakCanary 将记录虚拟机堆栈信息，并对其进行分析，之后向开发者展示。

## 源码解析

### 初始化

无需手动初始化的魔法来自 Content Provider。

``` kotlin
/**
 * Content providers are loaded before the application class is created. [AppWatcherInstaller] is
 * used to install [leakcanary.AppWatcher] on application start.
 */
internal sealed class AppWatcherInstaller : ContentProvider() {

  /**
   * [MainProcess] automatically sets up the LeakCanary code that runs in the main app process.
   */
  internal class MainProcess : AppWatcherInstaller()

  /**
   * When using the `leakcanary-android-process` artifact instead of `leakcanary-android`,
   * [LeakCanaryProcess] automatically sets up the LeakCanary code
   */
  internal class LeakCanaryProcess : AppWatcherInstaller()

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }

  ...
}
```

在 `onCreate()` 中调用了 `AppWatcher.manualInstall()`，实际内部是对非公开方法 `InternalAppWatcher.install()` 的一个包装。

``` kotlin
/**
 * [AppWatcher] is automatically installed in the main process on startup. You can
 * disable this behavior by overriding the `leak_canary_watcher_auto_install` boolean resource:
 *
 * ```
 * <?xml version="1.0" encoding="utf-8"?>
 * <resources>
 *   <bool name="leak_canary_watcher_auto_install">false</bool>
 * </resources>
 * ```
 *
 * If you disabled automatic install then you can call this method to install [AppWatcher].
 */
fun manualInstall(application: Application) {
  InternalAppWatcher.install(application)
}
```

`InternalAppWatcher.install()` 实现如下。

``` kotlin
fun install(application: Application) {
  checkMainThread()
  if (this::application.isInitialized) {
    return
  }
  InternalAppWatcher.application = application
  if (isDebuggableBuild) {
    SharkLog.logger = DefaultCanaryLog()
  }

  val configProvider = { AppWatcher.config }
  ActivityDestroyWatcher.install(application, objectWatcher, configProvider)
  FragmentDestroyWatcher.install(application, objectWatcher, configProvider)
  onAppWatcherInstalled(application)
}
```

在将 `application` 作为成员变量保存后，通过 `ActivityDestroyWatcher.install()` 与 `FragmentDestroyWatcher.install()` 对 Activity 与 Fragment 的内存泄漏问题进行监控，最后调用由外部设置的 Lambda 回调 `onAppWatcherInstalled`。

### Activity 监控

进入 `ActivityDestroyWatcher.install()`。

``` kotlin
internal class ActivityDestroyWatcher private constructor(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        if (configProvider().watchActivities) {
          objectWatcher.watch(
              activity, "${activity::class.java.name} received Activity#onDestroy() callback"
          )
        }
      }
    }

  companion object {
    fun install(
      application: Application,
      objectWatcher: ObjectWatcher,
      configProvider: () -> Config
    ) {
      val activityDestroyWatcher =
        ActivityDestroyWatcher(objectWatcher, configProvider)
      application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
    }
  }
}
```

`install()` 向 `application` 中注册了一个监听 Activity 销毁的生命周期监听器，当 `activity` 销毁时将其添加到 `objectWatcher` 中。

> `noOpDelegate()` 是一个带实化泛型的内联函数，作用是通过动态代理创建一个空实现接口的实例。

``` kotlin
/**
 * Watches the provided [watchedObject].
 *
 * @param description Describes why the object is watched.
 */
@Synchronized fun watch(
  watchedObject: Any,
  description: String
) {
  if (!isEnabled()) {
    return
  }
  removeWeaklyReachableObjects()
  val key = UUID.randomUUID()
      .toString()
  val watchUptimeMillis = clock.uptimeMillis()
  val reference =
    KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
  
  ...

  watchedObjects[key] = reference
  checkRetainedExecutor.execute {
    moveToRetained(key)
  }
}
```

`ObjectWatcher.watch()` 将传入的对象通过自定义弱引用类型 `KeyedWeakReference` 保存到成员函数映射 `watchedObjects` 中，并通过引用队列成员函数 `queue` 检查对象是否被释放。不过在此前后，`watch()` 都进行了一些操作。

添加对象前，`watch()` 调用了 `removeWeaklyReachableObjects()` 移除了 `watchedObjects` 中所有已添加到 `queue` 中的弱引用（即已经被回收对象的弱引用）。

``` kotlin
private fun removeWeaklyReachableObjects() {
  // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
  // reachable. This is before finalization or garbage collection has actually happened.
  var ref: KeyedWeakReference?
  do {
    ref = queue.poll() as KeyedWeakReference?
    if (ref != null) {
      watchedObjects.remove(ref.key)
    }
  } while (ref != null)
}
```

添加对象后，`watch()` 通过向 `checkRetainedExecutor` 推送一个执行 `moveToRetained()` 的 `Runnable`，当在 `watchedObjects` 中发现其弱引用后记录记录时间，并调用回调。

`checkRetainedExecutor` 默认实现是由 `InternalAppWatcher` 构建的主线程 Executor，内部调用了 `mainHandler.postDelayed()` 在延迟 `AppWatcher.config.watchDurationMillis` 后执行 `Runnable`。

``` kotlin
@Synchronized private fun moveToRetained(key: String) {
  removeWeaklyReachableObjects()
  val retainedRef = watchedObjects[key]
  if (retainedRef != null) {
    retainedRef.retainedUptimeMillis = clock.uptimeMillis()
    onObjectRetainedListeners.forEach { it.onObjectRetained() }
  }
}
```

最终由核心外部注册的 `OnObjectRetainedListener` 将会被调用，进行堆栈分析。

### Fragment 监控

`FragmentDestroyWatcher.install()` 的逻辑相比 `ActivityDestroyWatcher.install()` 较为复杂，但原理相同。

``` kotlin
fun install(
  application: Application,
  objectWatcher: ObjectWatcher,
  configProvider: () -> AppWatcher.Config
) {
  val fragmentDestroyWatchers = mutableListOf<(Activity) -> Unit>()

  if (SDK_INT >= O) {
    fragmentDestroyWatchers.add(
        AndroidOFragmentDestroyWatcher(objectWatcher, configProvider)
    )
  }

  getWatcherIfAvailable(
      ANDROIDX_FRAGMENT_CLASS_NAME,
      ANDROIDX_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
      objectWatcher,
      configProvider
  )?.let {
    fragmentDestroyWatchers.add(it)
  }

  getWatcherIfAvailable(
      ANDROID_SUPPORT_FRAGMENT_CLASS_NAME,
      ANDROID_SUPPORT_FRAGMENT_DESTROY_WATCHER_CLASS_NAME,
      objectWatcher,
      configProvider
  )?.let {
    fragmentDestroyWatchers.add(it)
  }

  if (fragmentDestroyWatchers.size == 0) {
    return
  }

  application.registerActivityLifecycleCallbacks(object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
    override fun onActivityCreated(
      activity: Activity,
      savedInstanceState: Bundle?
    ) {
      for (watcher in fragmentDestroyWatchers) {
        watcher(activity)
      }
    }
  })
}
```

通过对 *Android 8.0*、*AndroidX*、*Android Support Library* 三种情况进行分析并创建监听器后，监听器最终在对 Activity 创建的生命周期监听中设置到 Activity 中。

在三种类型的监听器中，以 *AndroidX* 对应的监听器 `AndroidXFragmentDestroyWatcher` 为例，通过生命周期监听的方式监听 Fragment 与内部 View 实现销毁对象观测功能的实现与 `ActivityDestroyWatcher` 基本一致。

``` kotlin
internal class AndroidXFragmentDestroyWatcher(
  private val objectWatcher: ObjectWatcher,
  private val configProvider: () -> Config
) : (Activity) -> Unit {

  private val fragmentLifecycleCallbacks = object : FragmentManager.FragmentLifecycleCallbacks() {

    override fun onFragmentCreated(
      fm: FragmentManager,
      fragment: Fragment,
      savedInstanceState: Bundle?
    ) {
      ViewModelClearedWatcher.install(fragment, objectWatcher, configProvider)
    }

    override fun onFragmentViewDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      val view = fragment.view
      if (view != null && configProvider().watchFragmentViews) {
        objectWatcher.watch(
            view, "${fragment::class.java.name} received Fragment#onDestroyView() callback " +
            "(references to its views should be cleared to prevent leaks)"
        )
      }
    }

    override fun onFragmentDestroyed(
      fm: FragmentManager,
      fragment: Fragment
    ) {
      if (configProvider().watchFragments) {
        objectWatcher.watch(
            fragment, "${fragment::class.java.name} received Fragment#onDestroy() callback"
        )
      }
    }
  }

  override fun invoke(activity: Activity) {
    if (activity is FragmentActivity) {
      val supportFragmentManager = activity.supportFragmentManager
      supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
      ViewModelClearedWatcher.install(activity, objectWatcher, configProvider)
    }
  }
}
```

至此完成核心源码的分析过程。