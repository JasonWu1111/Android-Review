# 进程和线程
当某个应用组件启动且该应用没有运行其他任何组件时，Android 系统会使用单个执行线程为应用启动新的 Linux 进程。默认情况下，同一应用的所有组件在相同的进程和线程（称为“主”线程）中运行。

各类组件元素的清单文件条目``<activity>``、``<service>``、``<receiver>`` 和 ``<provider>``—均支持 android:process 属性，此属性可以指定该组件应在哪个进程运行。

## 进程生命周期
**1、前台进程**
- 托管用户正在交互的 Activity（已调用 Activity 的 ``onResume()`` 方法）
- 托管某个 Service，后者绑定到用户正在交互的 Activity
- 托管正在“前台”运行的 Service（服务已调用 ``startForeground()``）
- 托管正执行一个生命周期回调的 Service（``onCreate()``、``onStart()`` 或 ``onDestroy()``）
- 托管正执行其 ``onReceive()`` 方法的 BroadcastReceiver

**2、可见进程**  
- 托管不在前台、但仍对用户可见的 Activity（已调用其 ``onPause()`` 方法）。例如，如果 re前台 Activity 启动了一个对话框，允许在其后显示上一 Activity，则有可能会发生这种情况。
- 托管绑定到可见（或前台）Activity 的 Service

**3、服务进程**  
- 正在运行已使用 startService() 方法启动的服务且不属于上述两个更高类别进程的进程。

**4、后台进程**
- 包含目前对用户不可见的 Activity 的进程（已调用 Activity 的 ``onStop()`` 方法）。通常会有很多后台进程在运行，因此它们会保存在 LRU （最近最少使用）列表中，以确保包含用户最近查看的 Activity 的进程最后一个被终止。

**5、空进程**
- 不含任何活动应用组件的进程。保留这种进程的的唯一目的是用作缓存，以缩短下次在其中运行组件所需的启动时间。 为使总体系统资源在进程缓存和底层内核缓存之间保持平衡，系统往往会终止这些进程。\

## 多进程
如果注册的四大组件中的任意一个组件时用到了多进程，运行该组件时，都会创建一个新的 Application 对象。对于多进程重复创建 Application 这种情况，只需要在该类中对当前进程加以判断即可。
```java
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        Log.d("MyApplication", getProcessName(android.os.Process.myPid()));
        super.onCreate();
    }

    /**
     * 根据进程 ID 获取进程名
     * @param pid 进程id
     * @return 进程名
     */
    public  String getProcessName(int pid){
        ActivityManager am = (ActivityManager)getSystemService(Context.ACTIVITY_SERVICE);
        List<ActivityManager.RunningAppProcessInfo> processInfoList = am.getRunningAppProcesses();
        if (processInfoList == null) {
            return null;
        }
        for (ActivityManager.RunningAppProcessInfo processInfo : processInfoList) {
            if (processInfo.pid == pid) {
                return processInfo.processName;
            }
        }
        return null;
    }
}
```

## 进程存活
### OOM_ADJ
| ADJ级别 | 取值 | 解释
|-----|-----|------
| UNKNOWN_ADJ | 16 | 一般指将要会缓存进程，无法获取确定值
| CACHED_APP_MAX_ADJ | 15 | 不可见进程的adj最大值
| CACHED_APP_MIN_ADJ | 9 | 不可见进程的adj最小值
| SERVICE_B_AD | 8 | B List中的Service（较老的、使用可能性更小）
| PREVIOUS_APP_ADJ | 7 | 上一个App的进程(往往通过按返回键)
| HOME_APP_ADJ | 6 | Home进程
| SERVICE_ADJ | 5 | 服务进程(Service process)
| HEAVY_WEIGHT_APP_ADJ | 4 | 后台的重量级进程，system/rootdir/init.rc文件中设置
| BACKUP_APP_ADJ | 3 | 备份进程
| PERCEPTIBLE_APP_ADJ | 2 | 可感知进程，比如后台音乐播放
| VISIBLE_APP_ADJ | 1 | 可见进程(Visible process)
| FOREGROUND_APP_ADJ | 0 | 前台进程（Foreground process)
| PERSISTENT_SERVICE_ADJ | -11 | 关联着系统或persistent进程
| PERSISTENT_PROC_ADJ | -12 | 系统persistent进程，比如telephony
| SYSTEM_ADJ | -16 | 系统进程
| NATIVE_ADJ | -17 | native进程（不被系统管理）

### 进程被杀情况
![](https://pic3.zhimg.com/80/18b6bfb1bf54433619a7122c3a8e606e_hd.png)

### 进程保活方案
- 开启一个像素的Activity
- 使用前台服务
- 多进程相互唤醒
- JobSheduler唤醒
- 粘性服务&与系统服务捆绑


## 线程
应用启动时，系统会为应用创建一个名为“主线程”的执行线程( UI 线程)。 此线程非常重要，因为它负责将事件分派给相应的用户界面小部件，其中包括绘图事件。 此外，它也是应用与 Android UI 工具包组件（来自 ``android.widget`` 和 ``android.view`` 软件包的组件）进行交互的线程。

系统不会为每个组件实例创建单独的线程。运行于同一进程的所有组件均在 UI 线程中实例化，并且对每个组件的系统调用均由该线程进行分派。 因此，响应系统回调的方法（例如，报告用户操作的 onKeyDown() 或生命周期回调方法）始终在进程的 UI 线程中运行。

Android 的单线程模式必须遵守两条规则:
- 不要阻塞 UI 线程
- 不要在 UI 线程之外访问 Android UI 工具包

为解决此问题，Android 提供了几种途径来从其他线程访问 UI 线程:
- ``Activity.runOnUiThread(Runnable)``
- ``View.post(Runnable)``
- ``View.postDelayed(Runnable, long)``

# IPC
IPC 即 Inter-Process Communication (进程间通信)。Android 基于 Linux，而 Linux 出于安全考虑，不同进程间不能之间操作对方的数据，这叫做“进程隔离”。
> 在 Linux 系统中，虚拟内存机制为每个进程分配了线性连续的内存空间，操作系统将这种虚拟内存空间映射到物理内存空间，每个进程有自己的虚拟内存空间，进而不能操作其他进程的内存空间，只有操作系统才有权限操作物理内存空间。 进程隔离保证了每个进程的内存安全。

## IPC方式
| 名称 | 优点 | 缺点 | 适用场景
|----|-----|----|----
| Bundle | 简单易用 | 只能传输Bundle支持的数据类型 | 四大组件间的进程间通信
| 文件共享 | 简单易用 | 不适合高并发场景，并且无法做到进程间即时通信|无并发访问情形，交换简单的数据实时性不高的场景
| AIDL|功能强大，支持一对多并发通信，支持实时通信 | 使用稍复杂，需要处理好线程同步 | 一对多通信且有RPC需求
| Messenger | 功能一般，支持一对多串行通信，支持实时通信|不能很处理高并发清醒，不支持RPC，数据通过Message进行传输，因此只能传输Bundle支持的数据类型|低并发的一对多即时通信，无RPC需求，或者无需返回结果的RPC需求
| ContentProvider | 在数据源访问方面功能强大，支持一对多并发数据共享，可通过Call方法扩展其他操作|可以理解为受约束的AIDL，主要提供数据源的CRUD操作 | 一对多的进程间数据共享
| Socket | 功能请打，可以通过网络传输字节流，支持一对多并发实时通信 | 实现细节稍微有点烦琐，不支持直接的RPC | 网络数据交换

## AIDL
Android Interface Definition Language

- **新建AIDL接口文件**
```java
// RemoteService.aidl
package com.example.mystudyapplication3;

interface IRemoteService {

    int getUserId();

}
```
- **创建远程服务**
```java
public class RemoteService extends Service {

    private int mId = -1;

    private Binder binder = new IRemoteService.Stub() {

        @Override
        public int getUserId() throws RemoteException {
            return mId;
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        mId = 1256;
        return binder;
    }
}
```
- **声明远程服务**
```java
<service
    android:name=".RemoteService"
    android:process=":aidl" />
```
- **绑定远程服务**
```java
public class MainActivity extends AppCompatActivity {

    public static final String TAG = "wzq";

    IRemoteService iRemoteService;
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            iRemoteService = IRemoteService.Stub.asInterface(service);
            try {
                Log.d(TAG, String.valueOf(iRemoteService.getUserId()));
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            iRemoteService = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bindService(new Intent(MainActivity.this, RemoteService.class), mConnection, Context.BIND_AUTO_CREATE);
    }
}
```

## Messenger
Messenger可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，底层实现是AIDL。

# Context
Context本身是一个抽象类，是对一系列系统服务接口的封装，包括：内部资源、包、类加载、I/O操作、权限、主线程、IPC和组件启动等操作的管理。ContextImpl, Activity, Service, Application这些都是Context的直接或间接子类, 关系如下:

![](http://gityuan.com/images/context/context.jpg)

ContextWrapper是代理Context的实现，简单地将其所有调用委托给另一个Context（mBase）。

Application、Activity、Service通过``attach() ``调用父类ContextWrapper的``attachBaseContext()``, 从而设置父类成员变量mBase为ContextImpl对象;, ontextWrapper的核心工作都是交给mBase(即ContextImpl)来完成.

# Activity
## 生命周期  
![](https://developer.android.google.cn/guide/components/images/activity_lifecycle.png)

- Activity A 启动另一个Activity B，回调如下:  
Activity A 的onPause() → Activity B的onCreate() → onStart() → onResume() → Activity A的onStop()；如果B是透明主题又或则是个DialogActivity，则不会回调A的onStop；

## 启动模式
| LaunchMode | 说明                      
|----------|-----|
| standard | 系统在启动它的任务中创建activity的新实例 |
| singleTop | 如果activity的实例已存在于当前任务的顶部，则系统通过调用其onNewIntent() |
| singleTask | 系统创建新task并在task的根目录下实例化activity。但如果activity的实例已存在于单独的任务中，则调用其onNewIntent()方法。一次只能存在一个activity实例 |
| singleInstance | 相同"singleTask"，activity始终是其task的唯一成员; 任何由此开始的activity都在一个单独的task中打开 |
&nbsp;
| 使用Intent标志 | 说明                      
|----------|-----|
| FLAG_ACTIVITY_NEW_TASK | 同singleTask |
| FLAG_ACTIVITY_SINGLE_TOP | 同singleTop |
| FLAG_ACTIVITY_CLEAR_TOP | 如果正在启动的activity已在当前task中运行，则不会启动该activity的新实例，而是销毁其上的activity，并调用其onNewIntent() |

## 启动过程
![](https://img-blog.csdn.net/20180427173504903)

``ActivityThread.java``
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    ActivityInfo aInfo = r.activityInfo;
    if (r.packageInfo == null) {
        //step 1: 创建LoadedApk对象
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                Context.CONTEXT_INCLUDE_CODE);
    }
    ... //component初始化过程

    java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
    //step 2: 创建Activity对象
    Activity activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
    ...

    //step 3: 创建Application对象
    Application app = r.packageInfo.makeApplication(false, mInstrumentation);

    if (activity != null) {
        //step 4: 创建ContextImpl对象
        Context appContext = createBaseContextForActivity(r, activity);
        CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
        Configuration config = new Configuration(mCompatConfiguration);
        //step5: 将Application/ContextImpl都attach到Activity对象 [见小节4.1]
        activity.attach(appContext, this, getInstrumentation(), r.token,
                r.ident, app, r.intent, r.activityInfo, title, r.parent,
                r.embeddedID, r.lastNonConfigurationInstances, config,
                r.referrer, r.voiceInteractor);

        ...
        int theme = r.activityInfo.getThemeResource();
        if (theme != 0) {
            activity.setTheme(theme);
        }

        activity.mCalled = false;
        if (r.isPersistable()) {
            //step 6: 执行回调onCreate
            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
        } else {
            mInstrumentation.callActivityOnCreate(activity, r.state);
        }

        r.activity = activity;
        r.stopped = true;
        if (!r.activity.mFinished) {
            activity.performStart(); //执行回调onStart
            r.stopped = false;
        }
        if (!r.activity.mFinished) {
            //执行回调onRestoreInstanceState
            if (r.isPersistable()) {
                if (r.state != null || r.persistentState != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                            r.persistentState);
                }
            } else if (r.state != null) {
                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
            }
        }
        ...
        r.paused = true;
        mActivities.put(r.token, r);
    }

    return activity;
}

```

# Fragment
## 特点
- Fragment 解决Activity间的切换不流畅，轻量切换】
- 可以从startActivityForResult中接收到返回结果,但是View不能
- 只能在 Activity 保存其状态（用户离开 Activity）之前使用 commit() 提交事务。如果您试图在该时间点后提交，则会引发异常。 这是因为如需恢复 Activity，则提交后的状态可能会丢失。 对于丢失提交无关紧要的情况，请使用 commitAllowingStateLoss()。

## 生命周期  
![](https://developer.android.google.cn/images/fragment_lifecycle.png)![](https://developer.android.google.cn/images/activity_fragment_lifecycle.png)  

## 与Activity通信
执行此操作的一个好方法是，在片段内定义一个回调接口，并要求宿主 Activity 实现它。
```java
public static class FragmentA extends ListFragment {
    ...
    // Container Activity must implement this interface
    public interface OnArticleSelectedListener {
        public void onArticleSelected(Uri articleUri);
    }
    ...
}

public static class FragmentA extends ListFragment {
    OnArticleSelectedListener mListener;
    ...
    @Override
    public void onAttach(Activity activity) {
        super.onAttach(activity);
        try {
            mListener = (OnArticleSelectedListener) activity;
        } catch (ClassCastException e) {
            throw new ClassCastException(activity.toString());
        }
    }
    ...
}
```

# Service
## 生命周期：
![](https://upload-images.jianshu.io/upload_images/944365-cf5c1a9d2dddaaca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/456/format/webp)
| 值 | 说明 |
|-----|-----|
| START_NOT_STICKY | 如果系统在 onStartCommand() 返回后终止服务，则除非有挂起 Intent 要传递，否则系统不会重建服务。这是最安全的选项，可以避免在不必要时以及应用能够轻松重启所有未完成的作业时运行服务 |
| START_STICKY | 如果系统在 onStartCommand() 返回后终止服务，则会重建服务并调用 onStartCommand()，但不会重新传递最后一个 Intent。相反，除非有挂起 Intent 要启动服务（在这种情况下，将传递这些 Intent ），否则系统会通过空 Intent 调用 onStartCommand()。这适用于不执行命令、但无限期运行并等待作业的媒体播放器（或类似服务 |
| START_REDELIVER_INTENT | 如果系统在 onStartCommand() 返回后终止服务，则会重建服务，并通过传递给服务的最后一个 Intent 调用 onStartCommand()。任何挂起 Intent 均依次传递。这适用于主动执行应该立即恢复的作业（例如下载文件）的服务 |

## 启用前台服务：
```java
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
```
```java
Notification notification = new Notification(icon, text, System.currentTimeMillis());
Intent notificationIntent = new Intent(this, ExampleActivity.class);
PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, 0);
notification.setLatestEventInfo(this, title, mmessage, pendingIntent);
startForeground(ONGOING_NOTIFICATION_ID, notification);
```
 
# 数据存储
| 存储方式 | 说明 |
|-----|-----|
| SharedPreferences | 在键值对中存储私有原始数据 |
| 内部存储 | 在设备内存中存储私有数据 |
| 外部存储 | 在共享的外部存储中存储公共数据 |
| SQLite 数据库 | 在私有数据库中存储结构化数据 |

# SharedPreferences
SharedPreferences采用key-value（键值对）形式, 主要用于轻量级的数据存储, 尤其适合保存应用的配置参数, 但不建议使用SharedPreferences来存储大规模的数据, 可能会降低性能.

SharedPreferences采用xml文件格式来保存数据, 该文件所在目录位于``/data/data/<package name>/shared_prefs``，如：
```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
   <string name="blog">https://github.com/JasonWu1111/Android-Review</string>
</map>
```

从Android N开始, 创建的SP文件模式, 不允许``MODE_WORLD_READABLE``和``MODE_WORLD_WRITEABLE``模块, 否则会直接抛出异常SecurityException。``MODE_MULTI_PROCESS``这种多进程的方式也是Google不推荐的方式, 后续同样会不再支持。

当设置MODE_MULTI_PROCESS模式, 则每次getSharedPreferences过程, 会检查SP文件上次修改时间和文件大小, 一旦所有修改则会重新从磁盘加载文件.

## 获取方式
### getPreferences
Activity.getPreferences(mode): 以当前Activity的类名作为SP的文件名. 即xxxActivity.xml
``Activity.java``
```java
public SharedPreferences getPreferences(int mode) {
    return getSharedPreferences(getLocalClassName(), mode);
}
```

### getDefaultSharedPreferences
PreferenceManager.getDefaultSharedPreferences(Context): 以包名加上_preferences作为文件名, 以MODE_PRIVATE模式创建SP文件. 即packgeName_preferences.xml.
```java
public static SharedPreferences getDefaultSharedPreferences(Context context) {
    return context.getSharedPreferences(getDefaultSharedPreferencesName(context),
           getDefaultSharedPreferencesMode());
}
```

### getSharedPreferences
直接调用Context.getSharedPreferences(name, mode)，所有的方法最终都是调用到如下方法：
```java
class ContextImpl extends Context {
    private ArrayMap<String, File> mSharedPrefsPaths;

    public SharedPreferences getSharedPreferences(String name, int mode) {
        File file;
        synchronized (ContextImpl.class) {
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            //先从mSharedPrefsPaths查询是否存在相应文件
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
                //如果文件不存在, 则创建新的文件 
                file = getSharedPreferencesPath(name);
                mSharedPrefsPaths.put(name, file);
            }
        }
 
        return getSharedPreferences(file, mode);
    }
}
```

## 架构
![](http://gityuan.com/images/sp/shared_preference.jpg)

SharedPreferences与Editor只是两个接口. SharedPreferencesImpl和EditorImpl分别实现了对应接口. 另外, ContextImpl记录着SharedPreferences的重要数据。

``putxxx()``操作把数据写入到EditorImpl.mModified；

``apply()/commit()``操作先调用commitToMemory(`, 将数据同步到SharedPreferencesImpl的mMap, 并保存到MemoryCommitResult的mapToWriteToDisk，再调用enqueueDiskWrite(), 写入到磁盘文件; 先之前把原有数据保存到.bak为后缀的文件,用于在写磁盘的过程出现任何异常可恢复数据;

``getxxx()``操作从SharedPreferencesImpl.mMap读取数据.

## apply / commit
- apply没有返回值, commit有返回值能知道修改是否提交成功  
- apply是将修改提交到内存，再异步提交到磁盘文件，而commit是同步的提交到磁盘文件
- 多并发的提交commit时，需等待正在处理的commit数据更新到磁盘文件后才会继续往下执行，从而降低效率; 而apply只是原子更新到内存，后调用apply函数会直接覆盖前面内存数据，从一定程度上提高很多效率。

## 注意
- 强烈建议不要在sp里面存储特别大的key/value，有助于减少卡顿/anr
- 不要高频地使用apply，尽可能地批量提交
- 不要使用MODE_MULTI_PROCESS
- 高频写操作的key与高频读操作的key可以适当地拆分文件，由于减少同步锁竞争
- 不要连续多次edit()，应该获取一次获取edit()，然后多次执行putxxx()，减少内存波动

# View
![](https://user-gold-cdn.xitu.io/2019/6/12/16b4a8a388f3a91a?imageslim)
ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立关联

View的整个绘制流程可以分为以下三个阶段：
- measure: 判断是否需要重新计算View的大小，需要的话则计算
- layout: 判断是否需要重新计算View的位置，需要的话则计算
- draw: 判断是否需要重新绘制View，需要的话则重绘制

![](https://img-blog.csdn.net/20180510164327114)

## MeasureSpec
MeasureSpec表示的是一个32位的整形值，它的高2位表示测量模式SpecMode，低30位表示某种测量模式下的规格大小SpecSize。MeasureSpec是View类的一个静态内部类，用来说明应该如何测量这个View

| Mode | 说明 |
|-----|-----|
| UNSPECIFIED | 不指定测量模式, 父视图没有限制子视图的大小，子视图可以是想要的任何尺寸，通常用于系统内部，应用开发中很少用到。 |
| EXACTLY | 精确测量模式，视图宽高指定为match_parent或具体数值时生效，表示父视图已经决定了子视图的精确大小，这种模式下View的测量值就是SpecSize的值|
| AT_MOST | 最大值测量模式，当视图的宽高指定为wrap_content时生效，此时子视图的尺寸可以是不超过父视图允许的最大尺寸的任何尺寸 |

对于DecorView而言，它的MeasureSpec由窗口尺寸和其自身的LayoutParams共同决定；对于普通的View，它的MeasureSpec由父视图的MeasureSpec和其自身的LayoutParams共同决定

直接继承View的控件需要重写onMeasure方法并设置wrap_content时的自身大小，否则在布局中使用wrap_content就相当于使用match_parent。解决方式如下：
```java
protected void onMeasure(int widthMeasureSpec, int height MeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int widtuhSpecSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    // 在wrap_content的情况下指定内部宽/高(mWidth和mHeight)
    int heightSpecSize = MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(mWidth, mHeight);
    } else if (widthSpecMode == MeasureSpec.AT_MOST) {
        setMeasureDimension(mWidth, heightSpecSize);
    } else if (heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasureDimension(widthSpecSize, mHeight);
    }
}
```
## 在Activity中获取某个View的宽高
- Activity/View#onWindowFocusChanged
```
// 此时View已经初始化完毕
// 当Activity的窗口得到焦点和失去焦点时均会被调用一次
// 如果频繁地进行onResume和onPause，那么onWindowFocusChanged也会被频繁地调用
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    if (hasFocus) {
        int width = view.getMeasureWidth();
        int height = view.getMeasuredHeight();
    }
}
```
- view.post(runnable)
```
// 通过post可以将一个runnable投递到消息队列的尾部，// 然后等待Looper调用次runnable的时候，View也已经初
// 始化好了
protected void onStart() {
    super.onStart();
    view.post(new Runnable() {

        @Override
        public void run() {
            int width = view.getMeasuredWidth();
            int height = view.getMeasuredHeight();
        }
    });
}
```
- ViewTreeObserver
```java
// 当View树的状态发生改变或者View树内部的View的可见// 性发生改变时，onGlobalLayout方法将被回调
protected void onStart() {
    super.onStart();

    ViewTreeObserver observer = view.getViewTreeObserver();
    observer.addOnGlobalLayoutListener(new OnGlobalLayoutListener() {

        @SuppressWarnings("deprecation")
        @Override
        public void onGlobalLayout() {
            view.getViewTreeObserver().removeGlobalOnLayoutListener(this);
            int width = view.getMeasuredWidth();
            int height = view.getMeasuredHeight();
        }
    });
}
```

## Draw的基本流程
```java
// 绘制基本上可以分为六个步骤
public void draw(Canvas canvas) {
    ...
    // 步骤一：绘制View的背景
    drawBackground(canvas);
    ...
    // 步骤二：如果需要的话，保持canvas的图层，为fading做准备
    saveCount = canvas.getSaveCount();
    ...
    canvas.saveLayer(left, top, right, top + length, null, flags);
    ...
    // 步骤三：绘制View的内容
    onDraw(canvas);
    ...
    // 步骤四：绘制View的子View
    dispatchDraw(canvas);
    ...
    // 步骤五：如果需要的话，绘制View的fading边缘并恢复图层
    canvas.drawRect(left, top, right, top + length, p);
    ...
    canvas.restoreToCount(saveCount);
    ...
    // 步骤六：绘制View的装饰(例如滚动条等等)
    onDrawForeground(canvas)
}
```

# Bitmap
![](https://upload-images.jianshu.io/upload_images/2618044-cd996dd172cce293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

## 配置信息与压缩方式
**Bitmap中有两个内部枚举类：**
- Config是用来设置颜色配置信息
- CompressFormat是用来设置压缩方式

| Config | 单位像素所占字节数 | 解析 
|-------|-------|------
| Bitmap.Config.ALPHA_8 | 1 | 颜色信息只由透明度组成，占8位 
| Bitmap.Config.ARGB_4444 | 2 |颜色信息由rgba四部分组成，每个部分都占4位，总共占16位 
| Bitmap.Config.ARGB_8888 | 4 |颜色信息由rgba四部分组成，每个部分都占8位，总共占32位。是Bitmap默认的颜色配置信息，也是最占空间的一种配置
| Bitmap.Config.RGB_565 | 2 | 颜色信息由rgb三部分组成，R占5位，G占6位，B占5位，总共占16位
| RGBA_F16 | 8 | Android 8.0 新增（更丰富的色彩表现HDR）
| HARDWARE | Special | Android 8.0 新增 （Bitmap直接存储在graphic memory）

> 通常我们优化Bitmap时，当需要做性能优化或者防止OOM，我们通常会使用Bitmap.Config.RGB_565这个配置，因为Bitmap.Config.ALPHA_8只有透明度，显示一般图片没有意义，Bitmap.Config.ARGB_4444显示图片不清楚，Bitmap.Config.ARGB_8888占用内存最多。

| CompressFormat | 解析 
|-------|-------
| Bitmap.CompressFormat.JPEG | 表示以JPEG压缩算法进行图像压缩，压缩后的格式可以是".jpg"或者".jpeg"，是一种有损压缩 |
| Bitmap.CompressFormat.PNG | 颜色信息由rgba四部分组成，每个部分都占4位，总共占16位 |
| Bitmap.Config.ARGB_8888 | 颜色信息由rgba四部分组成，每个部分都占8位，总共占32位。是Bitmap默认的颜色配置信息，也是最占空间的一种配置
| Bitmap.Config.RGB_565 | 颜色信息由rgb三部分组成，R占5位，G占6位，B占5位，总共占16位

## 常用操作
### 裁剪、缩放、旋转、移动
```java
Matrix matrix = new Matrix();  
// 缩放 
matrix.postScale(0.8f, 0.9f);  
// 左旋，参数为正则向右旋
matrix.postRotate(-45);  
// 平移, 在上一次修改的基础上进行再次修改 set 每次操作都是最新的 会覆盖上次的操作
matrix.postTranslate(100, 80);
// 裁剪并执行以上操作
Bitmap bitmap = Bitmap.createBitmap(source, 0, 0, source.getWidth(), source.getHeight(), matrix, true);
````
> 虽然Matrix还可以调用postSkew方法进行倾斜操作，但是却不可以在此时创建Bitmap时使用。

### Bitmap与Drawable转换
```java
// Drawable -> Bitmap
public static Bitmap drawableToBitmap(Drawable drawable) {
    Bitmap bitmap = Bitmap.createBitmap(drawable.getIntrinsicWidth(), drawable.getIntrinsicHeight(), drawable.getOpacity() != PixelFormat.OPAQUE ? Bitmap.Config.ARGB_8888 : Bitmap.Config.RGB_565);
    Canvas canvas = new Canvas(bitmap);
    drawable.setBounds(0, 0, drawable.getIntrinsicWidth(), drawable.getIntrinsicHeight();
    drawable.draw(canvas);
    return bitmap;
}

// Bitmap -> Drawable
public static Drawable bitmapToDrawable(Resources resources, Bitmap bm) {
    Drawable drawable = new BitmapDrawable(resources, bm);
    return drawable;
}
```

### 保存与释放
```java
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.test);
File file = new File(getFilesDir(),"test.jpg");
if(file.exists()){
    file.delete();
}
try {
    FileOutputStream outputStream=new FileOutputStream(file);
    bitmap.compress(Bitmap.CompressFormat.JPEG,90,outputStream);
    outputStream.flush();
    outputStream.close();
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
//释放bitmap的资源，这是一个不可逆转的操作
bitmap.recycle();
```

### 图片压缩
```java
public static Bitmap compressImage(Bitmap image) {
    if (image == null) {
        return null;
    }
    ByteArrayOutputStream baos = null;
    try {
        baos = new ByteArrayOutputStream();
        image.compress(Bitmap.CompressFormat.JPEG, 100, baos);
        byte[] bytes = baos.toByteArray();
        ByteArrayInputStream isBm = new ByteArrayInputStream(bytes);
        Bitmap bitmap = BitmapFactory.decodeStream(isBm);
        return bitmap;
    } catch (OutOfMemoryError e) {
        e.printStackTrace();
    } finally {
        try {
            if (baos != null) {
                baos.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    return null;
}
```

## BitmapFactory
### Bitmap创建流程
![](https://upload-images.jianshu.io/upload_images/2618044-9c2046ca5054da05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)


### Option类
| 常用方法 | 说明
|-----|------
| boolean inJustDecodeBounds | 如果设置为true，不获取图片，不分配内存，但会返回图片的高度宽度信息
| int inSampleSize | 图片缩放的倍数
| int outWidth | 获取图片的宽度值
| int outHeight | 获取图片的高度值
| int inDensity | 用于位图的像素压缩比
| int inTargetDensity | 用于目标位图的像素压缩比（要生成的位图）
| byte[] inTempStorage | 创建临时文件，将图片存储
| boolean inScaled | 设置为true时进行图片压缩，从inDensity到inTargetDensity
| boolean inDither | 如果为true,解码器尝试抖动解码
| Bitmap.Config inPreferredConfig | 设置解码器这个值是设置色彩模式，默认值是ARGB_8888，在这个模式下，一个像素点占用4bytes空间，一般对透明度不做要求的话，一般采用RGB_565模式，这个模式下一个像素点占用2bytes
| String outMimeType | 设置解码图像
| boolean inPurgeable | 当存储Pixel的内存空间在系统内存不足时是否可以被回收
| boolean inInputShareable | inPurgeable为true情况下才生效，是否可以共享一个InputStream
| boolean inPreferQualityOverSpeed | 为true则优先保证Bitmap质量其次是解码速度
| boolean inMutable | 配置Bitmap是否可以更改，比如：在Bitmap上隔几个像素加一条线段
| int inScreenDensity | 当前屏幕的像素密度

### 基本使用
```java
try {
    FileInputStream fis = new FileInputStream(filePath);
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    // 设置inJustDecodeBounds为true后，再使用decodeFile()等方法，并不会真正的分配空间，即解码出来的Bitmap为null，但是可计算出原始图片的宽度和高度，即options.outWidth和options.outHeight
    BitmapFactory.decodeFileDescriptor(fis.getFD(), null, options);
    float srcWidth = options.outWidth;
    float srcHeight = options.outHeight;
    int inSampleSize = 1;

    if (srcHeight > height || srcWidth > width) {
        if (srcWidth > srcHeight) {
            inSampleSize = Math.round(srcHeight / height);
        } else {
            inSampleSize = Math.round(srcWidth / width);
        }
    }

    options.inJustDecodeBounds = false;
    options.inSampleSize = inSampleSize;

    return BitmapFactory.decodeFileDescriptor(fis.getFD(), null, options);
} catch (Exception e) {
    e.printStackTrace();
}
```

## 内存回收
```java
if(bitmap != null && !bitmap.isRecycled()){ 
    // 回收并且置为null
    bitmap.recycle(); 
    bitmap = null; 
} 
```
Bitmap类的构造方法都是私有的，所以开发者不能直接new出一个Bitmap对象，只能通过BitmapFactory类的各种静态方法来实例化一个Bitmap。仔细查看BitmapFactory的源代码可以看到，生成Bitmap对象最终都是通过JNI调用方式实现的。所以，加载Bitmap到内存里以后，是包含两部分内存区域的。简单的说，一部分是Java部分的，一部分是C部分的。这个Bitmap对象是由Java部分分配的，不用的时候系统就会自动回收了，但是那个对应的C可用的内存区域，虚拟机是不能直接回收的，这个只能调用底层的功能释放。所以需要调用recycle()方法来释放C部分的内存。从Bitmap类的源代码也可以看到，recycle()方法里也的确是调用了JNI方法了的。

# Handler
Handler有两个主要用途：（1）安排Message和runnables在将来的某个时刻执行; （2）将要在不同于自己的线程上执行的操作排入队列。(在多个线程并发更新UI的同时保证线程安全。)

Handler创建的时候会采用当前线程的Looper来构造消息循环系统，需要注意的是，线程默认是没有Looper的，如果需要使用Handler就必须为线程创建Looper，因为默认的UI主线程，也就是ActivityThread，ActivityThread被创建的时候就会初始化Looper，这也是在主线程中默认可以使用Handler的原因。

- Message：Handler接收和处理的消息对象
- MessageQueue：Message的队列，先进先出，每一个线程最多可以拥有一个
- Looper：消息泵，是MessageQueue的管理者，会不断从MessageQueue中取出消息，并将消息分给对应的Handler处理，每个线程只有一个Looper。

# AsyncTask
- 异步任务的实例必须在UI线程中创建，即AsyncTask对象必须在UI线程中创建。
- execute(Params... params)方法必须在UI线程中调用。
- 不要手动调用onPreExecute()，doInBackground()，onProgressUpdate()，onPostExecute()这几个方法。
- 不能在doInBackground()中更改UI组件的信息。
- 一个任务实例只能执行一次，如果执行第二次将会抛出异常。
```java
import android.os.AsyncTask;

public class DownloadTask extends AsyncTask<String, Integer, Boolean> {

    @Override
    protected void onPreExecute() {
        super.onPreExecute();
    }

    @Override
    protected Boolean doInBackground(String... strings) {
        return null;
    }

    @Override
    protected void onProgressUpdate(Integer... values) {
        super.onProgressUpdate(values);
    }

    @Override
    protected void onPostExecute(Boolean aBoolean) {
        super.onPostExecute(aBoolean);
    }
}
```

# Serializable/Parcelable
- Serializable 使用 I/O 读写存储在硬盘上，而 Parcelable 是直接 在内存中读写
- Serializable 会使用反射，序列化和反序列化过程需要大量 I/O 操作， Parcelable 自已实现封送和解封（marshalled &unmarshalled）操作不需要用反射，数据也存放在 Native 内存中，效率要快很多

# 屏幕适配
## 单位
- dpi
每英寸像素数(dot per inch)  

- dp  
密度无关像素 - 一种基于屏幕物理密度的抽象单元。 这些单位相对于160 dpi的屏幕，因此一个dp是160 dpi屏幕上的一个px。 dp与像素的比率将随着屏幕密度而变化，但不一定成正比。为不同设备的UI元素的实际大小提供了一致性。

- sp  
与比例无关的像素 - 这与dp单位类似，但它也可以通过用户的字体大小首选项进行缩放。建议在指定字体大小时使用此单位，以便根据屏幕密度和用户偏好调整它们。
```
dpi = px / inch

density = dpi / 160

dp = px / density
```
## 头条适配方案
```java
private static void setCustomDensity(@NonNull Activity activity, @NonNull final Application application) {
    final DisplayMetrics appDisplayMetrics = application.getResources().getDisplayMetrics();
    if (sNoncompatDensity == 0) {
        sNoncompatDensity = appDisplayMetrics.density;
        sNoncompatScaledDensity = appDisplayMetrics.scaledDensity;
        // 监听字体切换
        application.registerComponentCallbacks(new ComponentCallbacks() {
            @Override
            public void onConfigurationChanged(Configuration newConfig) {
                if (newConfig != null && newConfig.fontScale > 0) {
                    sNoncompatScaledDensity = application.getResources().getDisplayMetrics().scaledDensity;
                }
            }

            @Override
            public void onLowMemory() {

            }
        });
    }
    
    // 适配后的dpi将统一为360dpi
    final float targetDensity = appDisplayMetrics.widthPixels / 360;
    final float targetScaledDensity = targetDensity * (sNoncompatScaledDensity / sNoncompatDensity);
    final int targetDensityDpi = (int)(160 * targetDensity);

    appDisplayMetrics.density = targetDensity;
    appDisplayMetrics.scaledDensity = targetScaledDensity;
    appDisplayMetrics.densityDpi = targetDensityDpi;

    final DisplayMetrics activityDisplayMetrics = activity.getResources().getDisplayMetrics();
    activityDisplayMetrics.density = targetDensity;
    activityDisplayMetrics.scaledDensity = targetScaledDensity;
    activityDisplayMetrics.densityDpi = targetDensityDpi
}
```