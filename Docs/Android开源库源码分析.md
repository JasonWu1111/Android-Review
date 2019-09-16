# LeakCanary
![](http://ww1.sinaimg.cn/large/006dXScfly1fj22w7flt4j30z00mrtc0.jpg)

## 初始化注册
在清单文件中注册了一个 ContentProvider 用于在应用启动时初始化代码：  

``leakcanary-leaksentry/*/AndroidManifest.xml``
```xml
···
    <application>
        <provider
            android:name="leakcanary.internal.LeakSentryInstaller"
            android:authorities="${applicationId}.leak-sentry-installer"
            android:exported="false"/>
    </application>
···
```

在 LeakSentryInstaller 生命周期 ``onCreate()`` 方法中完成初始化步骤：

``LeakSentryInstaller.kt``
```kotlin
internal class LeakSentryInstaller : ContentProvider() {

    override fun onCreate(): Boolean {
        CanaryLog.logger = DefaultCanaryLog()
        val application = context!!.applicationContext as Application
        InternalLeakSentry.install(application)
        return true
    }
···
```

然后分别注册 Activity/Fragment 的监听：

``InternalLeakSentry.kt``
```kotlin
···
    fun install(application: Application) {
        CanaryLog.d("Installing LeakSentry")
        checkMainThread()
        if (this::application.isInitialized) {
        return
        }
        InternalLeakSentry.application = application

        val configProvider = { LeakSentry.config }
        ActivityDestroyWatcher.install(
            application, refWatcher, configProvider
        )
        FragmentDestroyWatcher.install(
            application, refWatcher, configProvider
        )
        listener.onLeakSentryInstalled(application)
    }
···
```
``ActivityDestroyWatcher.kt``
```kotlin
···
    fun install(application: Application,refWatcher: RefWatcher,configProvider: () -> Config
        ) {
            val activityDestroyWatcher = ActivityDestroyWatcher(refWatcher, configProvider)
            application.registerActivityLifecycleCallbacks(activityDestroyWatcher.lifecycleCallbacks)
        }
    }
···
```
``AndroidOFragmentDestroyWatcher.kt``
```kotlin
···
    override fun watchFragments(activity: Activity) {
        val fragmentManager = activity.fragmentManager
        fragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
    }
···
```
``AndroidXFragmentDestroyWatcher.kt``
```kotlin
···
    override fun watchFragments(activity: Activity) {
        if (activity is FragmentActivity) {
        val supportFragmentManager = activity.supportFragmentManager
        supportFragmentManager.registerFragmentLifecycleCallbacks(fragmentLifecycleCallbacks, true)
        }
    }
···
```

## 引用泄漏观察
``RefWatcher.kt``
```kotlin
···
    @Synchronized fun watch(watchedInstance: Any, name: String) {
        if (!isEnabled()) {
            return
        }
        removeWeaklyReachableInstances()
        val key = UUID.randomUUID().toString()
        val watchUptimeMillis = clock.uptimeMillis()
        val reference = KeyedWeakReference(watchedInstance, key, name, watchUptimeMillis, queue)
        CanaryLog.d(
            "Watching %s with key %s",
            ((if (watchedInstance is Class<*>) watchedInstance.toString() else "instance of ${watchedInstance.javaClass.name}") + if (name.isNotEmpty()) " named $name" else ""), key
        )

        watchedInstances[key] = reference
        checkRetainedExecutor.execute {
            moveToRetained(key)
        }
    }

    @Synchronized private fun moveToRetained(key: String) {
        removeWeaklyReachableInstances()
        val retainedRef = watchedInstances[key]
        if (retainedRef != null) {
            retainedRef.retainedUptimeMillis = clock.uptimeMillis()
            onInstanceRetained()
        }
    }
···
```

``InternalLeakCanary.kt``
```kotlin
···
    override fun onReferenceRetained() {
        if (this::heapDumpTrigger.isInitialized) {
            heapDumpTrigger.onReferenceRetained()
        }
    }
···
```

## Dump Heap
发现泄漏之后，获取 Heamp Dump 相关文件：

``AndroidHeapDumper.kt``
```kotlin
···
    override fun dumpHeap(): File? {
        val heapDumpFile = leakDirectoryProvider.newHeapDumpFile() ?: return null
        ···
        return try {
            Debug.dumpHprofData(heapDumpFile.absolutePath)
            if (heapDumpFile.length() == 0L) {
                CanaryLog.d("Dumped heap file is 0 byte length")
                null
            } else {
                heapDumpFile
            }
        } catch (e: Exception) {
            CanaryLog.d(e, "Could not dump heap")
            // Abort heap dump
            null
        } finally {
            cancelToast(toast)
            notificationManager.cancel(R.id.leak_canary_notification_dumping_heap)
        }
    }
···
```

``HeapDumpTrigger.kt``
```kotlin
···
    private fun checkRetainedInstances(reason: String) {
        ···
        val heapDumpFile = heapDumper.dumpHeap()
        ···
        lastDisplayedRetainedInstanceCount = 0
        refWatcher.removeInstancesWatchedBeforeHeapDump(heapDumpUptimeMillis)

        HeapAnalyzerService.runAnalysis(application, heapDumpFile)
    }
···
```

启动一个 HeapAnalyzerService 来分析 heapDumpFile：

``HeapAnalyzerService.kt``
```kotlin
···
    override fun onHandleIntentInForeground(intent: Intent?) {
        ···
        val heapAnalyzer = HeapAnalyzer(this)
        val config = LeakCanary.config

        val heapAnalysis =
        heapAnalyzer.checkForLeaks(
            heapDumpFile, config.referenceMatchers, config.computeRetainedHeapSize, config.objectInspectors,
            if (config.useExperimentalLeakFinders) config.objectInspectors else listOf(
                AndroidObjectInspectors.KEYED_WEAK_REFERENCE
            )
        )

        config.analysisResultListener(application, heapAnalysis)
    }
···
```

>Heap Dump 之后，可以查看以下内容：
>- 应用分配了哪些类型的对象，以及每种对象的数量。
>- 每个对象使用多少内存。
>- 代码中保存对每个对象的引用。
>- 分配对象的调用堆栈。（调用堆栈当前仅在使用Android 7.1及以下时有效。）

<!-- # Glide
![](https://raw.githubusercontent.com/JsonChao/Awesome-Third-Library-Source-Analysis/master/ScreenShots/Glide%E6%A1%86%E6%9E%B6%E5%9B%BE.jpg) -->
