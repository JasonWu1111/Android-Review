# ART
ART 代表 Android Runtime，其处理应用程序执行的方式完全不同于 Dalvik，Dalvik 是依靠一个 Just-In-Time (JIT) 编译器去解释字节码。开发者编译后的应用代码需要通过一个解释器在用户的设备上运行，这一机制并不高效，但让应用能更容易在不同硬件和架构上运 行。ART 则完全改变了这套做法，在应用安装时就预编译字节码到机器语言，这一机制叫 Ahead-Of-Time (AOT）编译。在移除解释代码这一过程后，应用程序执行将更有效率，启动更快。

## ART 功能
### 预先 (AOT) 编译
ART 引入了预先编译机制，可提高应用的性能。ART 还具有比 Dalvik 更严格的安装时验证。在安装时，ART 使用设备自带的 dex2oat 工具来编译应用。该实用工具接受 DEX 文件作为输入，并为目标设备生成经过编译的应用可执行文件。该工具应能够顺利编译所有有效的 DEX 文件。

### 垃圾回收优化
垃圾回收 (GC) 可能有损于应用性能，从而导致显示不稳定、界面响应速度缓慢以及其他问题。ART 通过以下几种方式对垃圾回收做了优化：
- 只有一次（而非两次）GC 暂停
- 在 GC 保持暂停状态期间并行处理
- 在清理最近分配的短时对象这种特殊情况中，回收器的总 GC 时间更短
- 优化了垃圾回收的工效，能够更加及时地进行并行垃圾回收，这使得 GC_FOR_ALLOC 事件在典型用例中极为罕见
- 压缩 GC 以减少后台内存使用和碎片

### 开发和调试方面的优化
- 支持采样分析器
  
一直以来，开发者都使用 Traceview 工具（用于跟踪应用执行情况）作为分析器。虽然 Traceview 可提供有用的信息，但每次方法调用产生的开销会导致 Dalvik 分析结果出现偏差，而且使用该工具明显会影响运行时性能

ART 添加了对没有这些限制的专用采样分析器的支持，因而可更准确地了解应用执行情况，而不会明显减慢速度。KitKat 版本为 Dalvik 的 Traceview 添加了采样支持。


- 支持更多调试功能

ART 支持许多新的调试选项，特别是与监控和垃圾回收相关的功能。例如，查看堆栈跟踪中保留了哪些锁，然后跳转到持有锁的线程；询问指定类的当前活动的实例数、请求查看实例，以及查看使对象保持有效状态的参考；过滤特定实例的事件（如断点）等。

- 优化了异常和崩溃报告中的诊断详细信息
  
当发生运行时异常时，ART 会为您提供尽可能多的上下文和详细信息。ART 会提供 ``java.lang.ClassCastException``、``java.lang.ClassNotFoundException`` 和 ``java.lang.NullPointerException`` 的更多异常详细信息（较高版本的 Dalvik 会提供 ``java.lang.ArrayIndexOutOfBoundsException`` 和 ``java.lang.ArrayStoreException`` 的更多异常详细信息，这些信息现在包括数组大小和越界偏移量；ART 也提供这类信息）。

## ART GC
ART 有多个不同的 GC 方案，这些方案包括运行不同垃圾回收器。默认方案是 CMS（并发标记清除）方案，主要使用粘性 CMS 和部分 CMS。粘性 CMS 是 ART 的不移动分代垃圾回收器。它仅扫描堆中自上次 GC 后修改的部分，并且只能回收自上次 GC 后分配的对象。除 CMS 方案外，当应用将进程状态更改为察觉不到卡顿的进程状态（例如，后台或缓存）时，ART 将执行堆压缩。

除了新的垃圾回收器之外，ART 还引入了一种基于位图的新内存分配程序，称为 RosAlloc（插槽运行分配器）。此新分配器具有分片锁，当分配规模较小时可添加线程的本地缓冲区，因而性能优于 DlMalloc。

与 Dalvik 相比，ART CMS 垃圾回收计划在很多方面都有一定的改善：

- 与 Dalvik 相比，暂停次数从 2 次减少到 1 次。Dalvik 的第一次暂停主要是为了进行根标记，即在 ART 中进行并发标记，让线程标记自己的根，然后马上恢复运行。

- 与 Dalvik 类似，ART GC 在清除过程开始之前也会暂停 1 次。两者在这方面的主要差异在于：在此暂停期间，某些 Dalvik 环节在 ART 中并发进行。这些环节包括 java.lang.ref.Reference 处理、系统弱清除（例如，jni 弱全局等）、重新标记非线程根和卡片预清理。在 ART 暂停期间仍进行的阶段包括扫描脏卡片以及重新标记线程根，这些操作有助于缩短暂停时间。

- 相对于 Dalvik，ART GC 改进的最后一个方面是粘性 CMS 回收器增加了 GC 吞吐量。不同于普通的分代 GC，粘性 CMS 不移动。系统会将年轻对象保存在一个分配堆栈（基本上是 java.lang.Object 数组）中，而非为其设置一个专属区域。这样可以避免移动所需的对象以维持低暂停次数，但缺点是容易在堆栈中加入大量复杂对象图像而使堆栈变长。

ART GC 与 Dalvik 的另一个主要区别在于 ART GC 引入了移动垃圾回收器。使用移动 GC 的目的在于通过堆压缩来减少后台应用使用的内存。目前，触发堆压缩的事件是 ActivityManager 进程状态的改变。当应用转到后台运行时，它会通知 ART 已进入不再“感知”卡顿的进程状态。此时 ART 会进行一些操作（例如，压缩和监视器压缩），从而导致应用线程长时间暂停。目前正在使用的两个移动 GC 是同构空间压缩和半空间压缩。

- 半空间压缩将对象在两个紧密排列的碰撞指针空间之间进行移动。这种移动 GC 适用于小内存设备，因为它可以比同构空间压缩稍微多节省一点内存。额外节省出的空间主要来自紧密排列的对象，这样可以避免 RosAlloc/DlMalloc 分配器占用开销。由于 CMS 仍在前台使用，且不能从碰撞指针空间中进行收集，因此当应用在前台使用时，半空间还要再进行一次转换。这种情况并不理想，因为它可能引起较长时间的暂停。

- 同构空间压缩通过将对象从一个 RosAlloc 空间复制到另一个 RosAlloc 空间来实现。这有助于通过减少堆碎片来减少内存使用量。这是目前非低内存设备的默认压缩模式。相比半空间压缩，同构空间压缩的主要优势在于应用从后台切换到前台时无需进行堆转换。

# Apk 包体优化
## Apk 组成结构
| 文件/文件夹 | 作用/功能
|--|--
| res | 包含所有没有被编译到 .arsc 里面的资源文件
| lib | 引用库的文件夹
| assets | assets文件夹相比于 res 文件夹，还有可能放字体文件、预置数据和web页面等,通过 AssetManager 访问
| META_INF | 存放的是签名信息，用来保证 apk 包的完整性和系统的安全。在生成一个APK的时候，会对所有的打包文件做一个校验计算，并把结果放在该目录下面
| classes.dex | 包含编译后的应用程序源码转化成的dex字节码。APK 里面，可能会存在多个 dex 文件
| resources.arsc | 一些资源和标识符被编译和写入这个文件
| Androidmanifest.xml | 编译时，应用程序的 AndroidManifest.xml 被转化成二进制格式

## 整体优化
- 分离应用的独立模块，以插件的形式加载
- 解压APK，重新用 7zip 进行压缩
- 用 apksigner 签名工具 替代 java 提供的 jarsigner 签名工具

## 资源优化 
- 可以只用一套资源图片，一般采用 xhdpi 下的资源图片
- 通过扫描文件的 MD5 值，找出名字不同，内容相同的图片并删除
- 通过 Lint 工具扫描工程资源，移除无用资源
- 通过 Gradle 参数配置 shrinkResources=true
- 对 png 图片压缩
- 图片资源考虑采用 WebP 格式
- 避免使用帧动画，可使用 Lottie 动画库
- 优先考虑能否用 shape 代码、.9 图、svg 矢量图、VectorDrawable 类来替换传统的图片

## 代码优化
- 启用混淆以移除无用代码
- 剔除 R 文件
- 用注解替代枚举

## .arsc文件优化 
- 移除未使用的备用资源来优化 .arsc 文件
```groovy
android {
    defaultConfig {
        ...
        resConfigs "zh", "zh_CN", "zh_HK", "en"
    }
}
```

## lib目录优化
- 只提供对主流架构的支持，比如 arm，对于 mips 和 x86 架构可以考虑不提供支持
```groovy
android {
    defaultConfig {
        ...
        ndk {
            abiFilters  "armeabi-v7a"
        }
    }
}
```

# Hook
## 基本流程
1、根据需求确定 要 hook 的对象  
2、寻找要hook的对象的持有者，拿到要 hook 的对象  
3、定义“要 hook 的对象”的代理类，并且创建该类的对象  
4、使用上一步创建出来的对象，替换掉要 hook 的对象

## 使用示例
```java
/**
* hook的核心代码
* 这个方法的唯一目的：用自己的点击事件，替换掉 View 原来的点击事件
*
* @param view hook的范围仅限于这个view
*/
@SuppressLint({"DiscouragedPrivateApi", "PrivateApi"})
public static void hook(Context context, final View view) {//
    try {
        // 反射执行View类的getListenerInfo()方法，拿到v的mListenerInfo对象，这个对象就是点击事件的持有者
        Method method = View.class.getDeclaredMethod("getListenerInfo");
        method.setAccessible(true);//由于getListenerInfo()方法并不是public的，所以要加这个代码来保证访问权限
        Object mListenerInfo = method.invoke(view);//这里拿到的就是mListenerInfo对象，也就是点击事件的持有者

        // 要从这里面拿到当前的点击事件对象
        Class<?> listenerInfoClz = Class.forName("android.view.View$ListenerInfo");// 这是内部类的表示方法
        Field field = listenerInfoClz.getDeclaredField("mOnClickListener");
        final View.OnClickListener onClickListenerInstance = (View.OnClickListener) field.get(mListenerInfo);//取得真实的mOnClickListener对象

        // 2. 创建我们自己的点击事件代理类
        //   方式1：自己创建代理类
        //   ProxyOnClickListener proxyOnClickListener = new ProxyOnClickListener(onClickListenerInstance);
        //   方式2：由于View.OnClickListener是一个接口，所以可以直接用动态代理模式
        // Proxy.newProxyInstance的3个参数依次分别是：
        // 本地的类加载器;
        // 代理类的对象所继承的接口（用Class数组表示，支持多个接口）
        // 代理类的实际逻辑，封装在new出来的InvocationHandler内
        Object proxyOnClickListener = Proxy.newProxyInstance(context.getClass().getClassLoader(), new Class[]{View.OnClickListener.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Log.d("HookSetOnClickListener", "点击事件被hook到了");//加入自己的逻辑
                return method.invoke(onClickListenerInstance, args);//执行被代理的对象的逻辑
            }
        });
        // 3. 用我们自己的点击事件代理类，设置到"持有者"中
        field.set(mListenerInfo, proxyOnClickListener);
    } catch (Exception e) {
        e.printStackTrace();
    }
}

// 自定义代理类
static class ProxyOnClickListener implements View.OnClickListener {
    View.OnClickListener oriLis;

    public ProxyOnClickListener(View.OnClickListener oriLis) {
        this.oriLis = oriLis;
    }

    @Override
    public void onClick(View v) {
        Log.d("HookSetOnClickListener", "点击事件被hook到了");
        if (oriLis != null) {
            oriLis.onClick(v);
        }
    }
}
```

# Proguard
Proguard 具有以下三个功能：
- 压缩（Shrink）: 检测和删除没有使用的类，字段，方法和特性
- 优化（Optimize） : 分析和优化Java字节码
- 混淆（Obfuscate）: 使用简短的无意义的名称，对类，字段和方法进行重命名

## 公共模板
```
#############################################
#
# 对于一些基本指令的添加
#
#############################################
# 代码混淆压缩比，在 0~7 之间，默认为 5，一般不做修改
-optimizationpasses 5

# 混合时不使用大小写混合，混合后的类名为小写
-dontusemixedcaseclassnames

# 指定不去忽略非公共库的类
-dontskipnonpubliclibraryclasses

# 这句话能够使我们的项目混淆后产生映射文件
# 包含有类名->混淆后类名的映射关系
-verbose

# 指定不去忽略非公共库的类成员
-dontskipnonpubliclibraryclassmembers

# 不做预校验，preverify 是 proguard 的四个步骤之一，Android 不需要 preverify，去掉这一步能够加快混淆速度。
-dontpreverify

# 保留 Annotation 不混淆
-keepattributes *Annotation*,InnerClasses

# 避免混淆泛型
-keepattributes Signature

# 抛出异常时保留代码行号
-keepattributes SourceFile,LineNumberTable

# 指定混淆是采用的算法，后面的参数是一个过滤器
# 这个过滤器是谷歌推荐的算法，一般不做更改
-optimizations !code/simplification/cast,!field/*,!class/merging/*


#############################################
#
# Android开发中一些需要保留的公共部分
#
#############################################

# 保留我们使用的四大组件，自定义的 Application 等等这些类不被混淆
# 因为这些子类都有可能被外部调用
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Appliction
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.app.backup.BackupAgentHelper
-keep public class * extends android.preference.Preference
-keep public class * extends android.view.View
-keep public class com.android.vending.licensing.ILicensingService


# 保留 support 下的所有类及其内部类
-keep class android.support.** { *; }

# 保留继承的
-keep public class * extends android.support.v4.**
-keep public class * extends android.support.v7.**
-keep public class * extends android.support.annotation.**

# 保留 R 下面的资源
-keep class **.R$* { *; }

# 保留本地 native 方法不被混淆
-keepclasseswithmembernames class * {
    native <methods>;
}

# 保留在 Activity 中的方法参数是view的方法，
# 这样以来我们在 layout 中写的 onClick 就不会被影响
-keepclassmembers class * extends android.app.Activity {
    public void *(android.view.View);
}

# 保留枚举类不被混淆
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# 保留我们自定义控件（继承自 View）不被混淆
-keep public class * extends android.view.View {
    *** get*();
    void set*(***);
    public <init>(android.content.Context);
    public <init>(android.content.Context, android.util.AttributeSet);
    public <init>(android.content.Context, android.util.AttributeSet, int);
}

# 保留 Parcelable 序列化类不被混淆
-keep class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

# 保留 Serializable 序列化的类不被混淆
-keepnames class * implements java.io.Serializable
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    !static !transient <fields>;
    !private <fields>;
    !private <methods>;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}

# 对于带有回调函数的 onXXEvent、**On*Listener 的，不能被混淆
-keepclassmembers class * {
    void *(**On*Event);
    void *(**On*Listener);
}

# webView 处理，项目中没有使用到 webView 忽略即可
-keepclassmembers class fqcn.of.javascript.interface.for.webview {
    public *;
}
-keepclassmembers class * extends android.webkit.webViewClient {
    public void *(android.webkit.WebView, java.lang.String, android.graphics.Bitmap);
    public boolean *(android.webkit.WebView, java.lang.String);
}
-keepclassmembers class * extends android.webkit.webViewClient {
    public void *(android.webkit.webView, java.lang.String);
}

# js
-keepattributes JavascriptInterface
-keep class android.webkit.JavascriptInterface { *; }
-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}

# @Keep
-keep,allowobfuscation @interface android.support.annotation.Keep
-keep @android.support.annotation.Keep class *
-keepclassmembers class * {
    @android.support.annotation.Keep *;
}
```

##  常用的自定义混淆规则
```xml
# 通配符*，匹配任意长度字符，但不含包名分隔符(.)
# 通配符**，匹配任意长度字符，并且包含包名分隔符(.)

# 不混淆某个类
-keep public class com.jasonwu.demo.Test { *; }

# 不混淆某个包所有的类
-keep class com.jasonwu.demo.test.** { *; }

# 不混淆某个类的子类
-keep public class * com.jasonwu.demo.Test { *; }

# 不混淆所有类名中包含了 ``model`` 的类及其成员
-keep public class **.*model*.** {*;}

# 不混淆某个接口的实现
-keep class * implements com.jasonwu.demo.TestInterface { *; }

# 不混淆某个类的构造方法
-keepclassmembers class com.jasonwu.demo.Test { 
  public <init>(); 
}

# 不混淆某个类的特定的方法
-keepclassmembers class com.jasonwu.demo.Test { 
  public void test(java.lang.String); 
}
```


## aar中增加独立的混淆配置
``build.gralde``
```gradle
android {
    ···
    defaultConfig {
        ···
        consumerProguardFile 'proguard-rules.pro'
    }
    ···
}
```

## 检查混淆和追踪异常
开启 Proguard 功能，则每次构建时 ProGuard 都会输出下列文件：

- dump.txt  
说明 APK 中所有类文件的内部结构。

- mapping.txt  
提供原始与混淆过的类、方法和字段名称之间的转换。

- seeds.txt  
列出未进行混淆的类和成员。

- usage.txt  
列出从 APK 移除的代码。

这些文件保存在 /build/outputs/mapping/release/ 中。我们可以查看 seeds.txt 里面是否是我们需要保留的，以及 usage.txt 里查看是否有误删除的代码。 mapping.txt 文件很重要，由于我们的部分代码是经过重命名的，如果该部分出现 bug，对应的异常堆栈信息里的类或成员也是经过重命名的，难以定位问题。我们可以用 retrace 脚本（在 Windows 上为 retrace.bat；在 Mac/Linux 上为 retrace.sh）。它位于 /tools/proguard/ 目录中。该脚本利用 mapping.txt 文件和你的异常堆栈文件生成没有经过混淆的异常堆栈文件,这样就可以看清是哪里出问题了。使用 retrace 工具的语法如下：

```shell
retrace.bat|retrace.sh [-verbose] mapping.txt [<stacktrace_file>]
```

# 架构
## MVC
![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCwLhGyLdicyLzgUDKFTZVt1OgU6iaSx2IUwnygzmQzW7Renaa8hmQ62cQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在 Android 中，三者的关系如下：

![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCicNvEVMO9vDgukUR29Z1DCacZJwmmH1EEb7gUOZmDxolWexP01O8jfg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

由于在 Android 中 xml 布局的功能性太弱，所以 Activity 承担了绝大部分的工作，所以在 Android 中 mvc 更像：

![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCOq89MLQX4UM3dgBTQfU72desHb1XbOWRQZINnXOCCdZCuicUiaTHhtEg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

总结：
- 具有一定的分层，model 解耦，controller 和 view 并没有解耦
- controller 和 view 在 Android 中无法做到彻底分离，Controller 变得臃肿不堪
- 易于理解、开发速度快、可维护性高

## MVP
![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCLVgibsuVQFguBI8FBdZibLNfpvbpd6njkdGWdyR2UL6TzMOhKHFqLC0Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过引入接口 BaseView，让相应的视图组件如 Activity，Fragment去实现 BaseView，把业务逻辑放在 presenter 层中，弱化 Model 只有跟 view 相关的操作都由 View 层去完成。

总结：
- 彻底解决了 MVC 中 View 和 Controller 傻傻分不清楚的问题
- 但是随着业务逻辑的增加，一个页面可能会非常复杂，UI 的改变是非常多，会有非常多的 case，这样就会造成 View 的接口会很庞大
- 更容易单元测试

## MVVM
![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCMygIDD6xo5djkq6Y3jZo53sT2A4kKNaz8JEVRwmUnTmcAwJm0pZVWg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在 MVP 中 View 和 Presenter 要相互持有，方便调用对方，而在 MVP 中 View 和 ViewModel 通过 Binding 进行关联，他们之前的关联处理通过  DataBinding 完成。

总结：
- 很好的解决了 MVC 和 MVP 的问题
- 视图状态较多，ViewModel 的构建和维护的成本都会比较高
- 但是由于数据和视图的双向绑定，导致出现问题时不太好定位来源

# Jetpack
## 架构
![](https://developer.android.google.cn/topic/libraries/architecture/images/final-architecture.png)

## 使用示例
``build.gradle``
```groovy
android {
    ···
    dataBinding {
        enabled = true
    }
}
dependencies {
    ···
    implementation "androidx.fragment:fragment-ktx:$rootProject.fragmentVersion"
    implementation "androidx.lifecycle:lifecycle-extensions:$rootProject.lifecycleVersion"
    implementation "androidx.lifecycle:lifecycle-livedata-ktx:$rootProject.lifecycleVersion"
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$rootProject.lifecycleVersion"
}
```

``fragment_plant_detail.xml``
```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="viewModel"
            type="com.google.samples.apps.sunflower.viewmodels.PlantDetailViewModel" />
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            ···
            android:text="@{viewModel.plant.name}"/>

    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```


``PlantDetailFragment.kt``
```kotlin
class PlantDetailFragment : Fragment() {

    private val args: PlantDetailFragmentArgs by navArgs()
    private lateinit var shareText: String

    private val plantDetailViewModel: PlantDetailViewModel by viewModels {
        InjectorUtils.providePlantDetailViewModelFactory(requireActivity(), args.plantId)
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        val binding = DataBindingUtil.inflate<FragmentPlantDetailBinding>(
                inflater, R.layout.fragment_plant_detail, container, false).apply {
            viewModel = plantDetailViewModel
            lifecycleOwner = this@PlantDetailFragment
        }

        plantDetailViewModel.plant.observe(this) { plant ->
            // 更新相关 UI
        }

        return binding.root
    }
}
```

``Plant.kt``
```kotlin
data class Plant (
    val name: String
)
```

``PlantDetailViewModel.kt``
```kotlin
class PlantDetailViewModel(
    plantRepository: PlantRepository,
    private val plantId: String
) : ViewModel() {

    val plant: LiveData<Plant>

    override fun onCleared() {
        super.onCleared()
        viewModelScope.cancel()
    }

    init {
        plant = plantRepository.getPlant(plantId)
    }
}
```

``PlantDetailViewModelFactory.kt``
```kotlin
class PlantDetailViewModelFactory(
    private val plantRepository: PlantRepository,
    private val plantId: String
) : ViewModelProvider.NewInstanceFactory() {

    @Suppress("UNCHECKED_CAST")
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return PlantDetailViewModel(plantRepository, plantId) as T
    }
}
```

``InjectorUtils.kt``
```kotlin
object InjectorUtils {
    private fun getPlantRepository(context: Context): PlantRepository {
        ···
    }

    fun providePlantDetailViewModelFactory(
        context: Context,
        plantId: String
    ): PlantDetailViewModelFactory {
        return PlantDetailViewModelFactory(getPlantRepository(context), plantId)
    }
}
```

# 设计模式
| 模式 & 描述 | 包括
|--|--
| **创建型模式**<br>提供了一种在创建对象的同时隐藏创建逻辑的方式。| 工厂模式（Factory Pattern）<br>抽象工厂模式（Abstract Factory Pattern）<br>单例模式（Singleton Pattern）<br>建造者模式（Builder Pattern）<br>原型模式（Prototype Pattern）
| **结构型模式**<br>关注类和对象的组合。| 适配器模式（Adapter Pattern）<br>桥接模式（Bridge Pattern）<br>过滤器模式（Filter、Criteria Pattern）<br>组合模式（Composite Pattern）<br>装饰器模式（Decorator Pattern）<br>外观模式（Facade Pattern）<br>享元模式（Flyweight Pattern）<br>代理模式（Proxy Pattern）
| **行为型模式**<br>特别关注对象之间的通信。| 责任链模式（Chain of Responsibility Pattern）<br>命令模式（Command Pattern）<br>解释器模式（Interpreter Pattern）<br>迭代器模式（Iterator Pattern）<br>中介者模式（Mediator Pattern）<br>备忘录模式（Memento Pattern）<br>观察者模式（Observer Pattern）<br>状态模式（State Pattern）<br>空对象模式（Null Object Pattern）<br>策略模式（Strategy Pattern）<br>模板模式（Template Pattern）<br>访问者模式（Visitor Pattern）

## 面向对象六大原则
| 原则 | 描述
|--|--
| 单一职责原则 | 一个类只负责一个功能领域中的相应职责。
| 开闭原则 | 对象应该对于扩展是开放的，对于修改是封闭的。
| 里氏替换原则 | 所有引用基类的地方必须能透明地使用其子类的对象。
| 依赖倒置原则 | 高层模块不依赖低层模块，两者应该依赖其对象；抽象不应该依赖细节；细节应该依赖抽象。
| 接口隔离原则 | 类间的依赖关系应该建立在最小的接口上。
| 迪米特原则 | 也称最少知识原则，一个对象对其他对象有最少的了解。

## 工厂模式
适用于复杂对象的创建。

示例：
```java
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.demo);
```

``BitmapFactory.java``
```java
// 生成 Bitmap 对象的工厂类 BitmapFactory
public class BitmapFactory {
    ···
    public static Bitmap decodeFile(String pathName) {
        ···
    }
    ···

    public static Bitmap decodeResource(Resources res, int id, Options opts) {
        validate(opts);
        Bitmap bm = null;
        InputStream is = null; 
        
        try {
            final TypedValue value = new TypedValue();
            is = res.openRawResource(id, value);

            bm = decodeResourceStream(res, value, is, null, opts);
        } 
        ···
        return bm;
    }
    ···
}

```

## 单例模式
确保某一个类只有一个实例，并自动实例化向整个系统提供这个实例，且可以避免产生多个对象消耗资源。

示例：

``InputMethodManager.java``
```java
/**
* Retrieve the global InputMethodManager instance, creating it if it
* doesn't already exist.
* @hide
*/
public static InputMethodManager getInstance() {
    synchronized (InputMethodManager.class) {
        if (sInstance == null) {
            try {
                sInstance = new InputMethodManager(Looper.getMainLooper());
            } catch (ServiceNotFoundException e) {
                throw new IllegalStateException(e);
            }
        }
        return sInstance;
    }
}
```

## 建造者模式
将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示，适用于初始化的对象比较复杂且参数较多的情况。

示例：
```java
AlertDialog.Builder builder = new AlertDialog.Builder(this)
        .setTitle("Title")
        .setMessage("Message");
AlertDialog dialog = builder.create();
dialog.show();
```

``AlertDialog.java``
```java
public class AlertDialog extends Dialog implements DialogInterface {
    ···

    public static class Builder {
        private final AlertController.AlertParams P;
        ···

        public Builder(Context context) {
            this(context, resolveDialogTheme(context, ResourceId.ID_NULL));
        }
        ···

        public Builder setTitle(CharSequence title) {
            P.mTitle = title;
            return this;
        }
        ···

        public Builder setMessage(CharSequence message) {
            P.mMessage = message;
            return this;
        }
        ···

        public AlertDialog create() {
            // Context has already been wrapped with the appropriate theme.
            final AlertDialog dialog = new AlertDialog(P.mContext, 0, false);
            P.apply(dialog.mAlert);
            ···
            return dialog;
        }
        ···
    }
}
```

## 原型模式
用原型模式实例指定创建对象的种类，并通过拷贝这些原型创建新的对象。

示例：

```java
ArrayList<T> newArrayList = (ArrayList<T>) arrayList.clone();
```

``ArrayList.java``
```java
/**
 * Returns a shallow copy of this <tt>ArrayList</tt> instance.  (The
 * elements themselves are not copied.)
 *
 * @return a clone of this <tt>ArrayList</tt> instance
 */
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
}
```

## 适配器模式
适配器模式把一个类的接口变成客户端所期待的另一种接口，从而使原因接口不匹配而无法一起工作的两个类能够在一起工作。

示例：

```java
RecyclerView recyclerView = findViewById(R.id.recycler_view);
recyclerView.setAdapter(new MyAdapter());

private class MyAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        ···
    }

    ···
}
```

``RecyclerView.java``
```java
···
private void setAdapterInternal(@Nullable Adapter adapter, boolean compatibleWithPrevious,
        boolean removeAndRecycleViews) {
    if (mAdapter != null) {
        mAdapter.unregisterAdapterDataObserver(mObserver);
        mAdapter.onDetachedFromRecyclerView(this);
    }
    ···
    mAdapterHelper.reset();
    final Adapter oldAdapter = mAdapter;
    mAdapter = adapter;
    if (adapter != null) {
        adapter.registerAdapterDataObserver(mObserver);
        adapter.onAttachedToRecyclerView(this);
    }
    if (mLayout != null) {
        mLayout.onAdapterChanged(oldAdapter, mAdapter);
    }
    mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
    mState.mStructureChanged = true;
}

···
public final class Recycler {
    @Nullable
    ViewHolder tryGetViewHolderForPositionByDeadline(int position,
            boolean dryRun, long deadlineNs) {
        ···
        ViewHolder holder = null;
        ···
        if (holder == null) {
            ···
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            ···
        }
        ···
        return holder;
    }
}

···
public abstract static class Adapter<VH extends ViewHolder> {
    ···
    @NonNull
    public abstract VH onCreateViewHolder(@NonNull ViewGroup parent, int viewType);

    @NonNull
    public final VH createViewHolder(@NonNull ViewGroup parent, int viewType) {
        try {
            TraceCompat.beginSection(TRACE_CREATE_VIEW_TAG);
            final VH holder = onCreateViewHolder(parent, viewType);
            ···
            holder.mItemViewType = viewType;
            return holder;
        } finally {
            TraceCompat.endSection();
        }
    }
    ···
}
```

## 观察者模式
定义对象间一种一对多的依赖关系，使得每当一个对象改变状态，则所有依赖于它的对象都会得到通知并被自动更新。

示例：
```java
MyAdapter adapter = new MyAdapter();
recyclerView.setAdapter(adapter);
adapter.notifyDataSetChanged();
```

``RecyclerView.java``
```java
···
private final RecyclerViewDataObserver mObserver = new RecyclerViewDataObserver();

···
private void setAdapterInternal(@Nullable Adapter adapter, boolean compatibleWithPrevious,
        boolean removeAndRecycleViews) {
    ···
    mAdapter = adapter;
    if (adapter != null) {
        adapter.registerAdapterDataObserver(mObserver);
        adapter.onAttachedToRecyclerView(this);
    }
    ···
}

···
public abstract static class Adapter<VH extends ViewHolder> {
    private final AdapterDataObservable mObservable = new AdapterDataObservable();
    ···
    public void registerAdapterDataObserver(@NonNull AdapterDataObserver observer) {
        mObservable.registerObserver(observer);
    }

    ···
    public final void notifyDataSetChanged() {
        mObservable.notifyChanged();
    }
}

static class AdapterDataObservable extends Observable<AdapterDataObserver> {
    ···
    public void notifyChanged() {
        for (int i = mObservers.size() - 1; i >= 0; i--) {
            mObservers.get(i).onChanged();
        }
    }
    ···
}

private class RecyclerViewDataObserver extends AdapterDataObserver {
    ···
    @Override
    public void onChanged() {
        assertNotInLayoutOrScroll(null);
        mState.mStructureChanged = true;

        processDataSetCompletelyChanged(true);
        if (!mAdapterHelper.hasPendingUpdates()) {
            requestLayout();
        }
    }
    ···
}
```

## 代理模式
为其他的对象提供一种代理以控制对这个对象的访问。适用于当无法或不想直接访问某个对象时通过一个代理对象来间接访问，为了保证客户端使用的透明性，委托对象与代理对象需要实现相同的接口。

示例：

``Context.java``
```java
public abstract class Context {
    ···
    public abstract void startActivity(@RequiresPermission Intent intent);
    ···
}
```

``ContextWrapper.java``
```java
public class ContextWrapper extends Context {
    Context mBase; // 代理类，实为 ContextImpl 对象
    ···

    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    ···

    @Override
    public void startActivity(Intent intent) {
        mBase.startActivity(intent); // 核心工作交由给代理类对象 mBase 实现
    }
    ···
}
```
``ContextImpl.java``
```java
// Context 的真正实现类
class ContextImpl extends Context {
    ...
    @Override
    public void startActivity(Intent intent) {
        warnIfCallingFromSystemProcess();
        startActivity(intent, null);
    }
    ...
}
```

## 责任链模式
使多个对象都有机会处理请求，从而避免了请求的发送者和接受者之间的耦合。将这些对象连成一条链，并沿着这条链传递该请求，直到有对象处理它为止。

``ViewGroup.java``
```java
@UiThread
public abstract class ViewGroup extends View implements ViewParent, ViewManager {
    ···
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
        ···
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    ···
                    // 获取子 view 处理的结果
                    handled = child.dispatchTouchEvent(event);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // Perform any necessary transformations and dispatch.
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            ···
            // 获取子 view 处理的结果
            handled = child.dispatchTouchEvent(transformedEvent);
        }
        ···
        return handled;
    }
    ···
}
```

## 策略模式
策略模式定义了一系列的算法，并封装起来，提供针对同一类型问题的多种处理方式。

示例：

```java
// 匀速
animation.setInterpolator(new LinearInterpolator());
// 加速
animation.setInterpolator(new AccelerateInterpolator());
···
```

``BaseInterpolator.java``
```java
/**
 * An abstract class which is extended by default interpolators.
 */
abstract public class BaseInterpolator implements Interpolator {
    private @Config int mChangingConfiguration;
    /**
     * @hide
     */
    public @Config int getChangingConfiguration() {
        return mChangingConfiguration;
    }

    /**
     * @hide
     */
    void setChangingConfiguration(@Config int changingConfiguration) {
        mChangingConfiguration = changingConfiguration;
    }
}
```

``LinearInterpolator.java``
```java
@HasNativeInterpolator
public class LinearInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {
    ···
}
```

``AccelerateInterpolator.java``
```java
@HasNativeInterpolator
public class AccelerateInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {
    ···
}
```

## 备忘录模式
在不破坏封闭的前提下，在对象之外保存保存对象的当前状态，并且在之后可以恢复到此状态。

示例：

``Activity.java``
```java
// 保存状态
protected void onSaveInstanceState(Bundle outState) {
    // 存储当前窗口的视图树的状态
    outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());

    outState.putInt(LAST_AUTOFILL_ID, mLastAutofillId);
    // 存储 Fragment 的状态
    Parcelable p = mFragments.saveAllState();
    if (p != null) {
        outState.putParcelable(FRAGMENTS_TAG, p);
    }
    if (mAutoFillResetNeeded) {
        outState.putBoolean(AUTOFILL_RESET_NEEDED, true);
        getAutofillManager().onSaveInstanceState(outState);
    }
    // 调用 ActivityLifecycleCallbacks 的 onSaveInstanceState 进行存储状态
    getApplication().dispatchActivitySaveInstanceState(this, outState);
}

···
// onCreate 方法中恢复状态
protected void onCreate(@Nullable Bundle savedInstanceState) {
    ···
    if (savedInstanceState != null) {
        mAutoFillResetNeeded = savedInstanceState.getBoolean(AUTOFILL_RESET_NEEDED, false);
        mLastAutofillId = savedInstanceState.getInt(LAST_AUTOFILL_ID,
                View.LAST_APP_AUTOFILL_ID);

        if (mAutoFillResetNeeded) {
            getAutofillManager().onCreate(savedInstanceState);
        }

        Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
        mFragments.restoreAllState(p, mLastNonConfigurationInstances != null
                ? mLastNonConfigurationInstances.fragments : null);
    }
    mFragments.dispatchCreate();
    getApplication().dispatchActivityCreated(this, savedInstanceState);
    ···
    mRestoredFromBundle = savedInstanceState != null;
    mCalled = true;
}
```

``ActivityThread.java``
```java
@Override
public void handleStartActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions) {
    final Activity activity = r.activity;
    ···
    // Start
    activity.performStart("handleStartActivity");
    r.setState(ON_START);
    ···
    // Restore instance state
    if (pendingActions.shouldRestoreInstanceState()) {
        if (r.isPersistable()) {
            if (r.state != null || r.persistentState != null) {
                mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                        r.persistentState);
            }
        } else if (r.state != null) {
            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
        }
    }
    ···
}
```


# NDK 开发
> NDK 全称是 Native Development Kit，是一组可以让你在 Android 应用中编写实现 C/C++ 的工具，可以在项目用自己写源代码构建，也可以利用现有的预构建库。

使用 NDK 的使用目的有：
- 从设备获取更好的性能以用于计算密集型应用，例如游戏或物理模拟  
- 重复使用自己或其他开发者的 C/C++ 库，便利于跨平台。  
- NDK 集成了譬如 OpenSL、Vulkan 等 API 规范的特定实现，以实现在 java 层无法做到的功能如提升音频性能等  
- 增加反编译难度

## JNI 基础
### 数据类型
- 基本数据类型
  
| Java 类型 | Native 类型 | 符号属性 | 字长
|--|--|--|--
| boolean | jboolean | 无符号 | 8位
| byte | jbyte | 无符号 | 8位
| char | jchar | 无符号 | 16位
| short | jshort | 有符号 | 16位
| int | jnit | 有符号 | 32位
| long | jlong | 有符号 | 64位
| float | jfloat | 有符号 | 32位
| double | jdouble | 有符号 | 64位

- 引用数据类型

| Java 引用类型	| Native 类型 | Java 引用类型 | Native 类型
|--|--|--|--
| All objects | jobject | char[] | jcharArray
| java.lang.Class | jclass | short[] | jshortArray
| java.lang.String | jstring | int[] | jintArray
| Object[] | jobjectArray | long[] | jlongArray
| boolean[] | jbooleanArray | float[] | jfloatArray
| byte[] | jbyteArray | double[] | jdoubleArray
| java.lang.Throwable | jthrowable	

### String 字符串函数操作
| JNI 函数 | 描述
|--|--
| GetStringChars / ReleaseStringChars | 获得或释放一个指向 Unicode 编码的字符串的指针（指 C/C++ 字符串）
| GetStringUTFChars / ReleaseStringUTFChars | 获得或释放一个指向 UTF-8 编码的字符串的指针（指 C/C++ 字符串）
| GetStringLength | 返回 Unicode 编码的字符串的长度
| getStringUTFLength | 返回 UTF-8 编码的字符串的长度
| NewString | 将 Unicode 编码的 C/C++ 字符串转换为 Java 字符串
| NewStringUTF | 将 UTF-8 编码的 C/C++ 字符串转换为 Java 字符串
| GetStringCritical / ReleaseStringCritical | 获得或释放一个指向字符串内容的指针(指 Java 字符串)
| GetStringRegion | 获取或者设置 Unicode 编码的字符串的指定范围的内容
| GetStringUTFRegion | 获取或者设置 UTF-8 编码的字符串的指定范围的内容

### 常用 JNI 访问 Java 对象方法
``MyJob.java``
```java
package com.example.myjniproject;

public class MyJob {

    public static String JOB_STRING = "my_job";
    private int jobId;

    public MyJob(int jobId) {
        this.jobId = jobId;
    }

    public int getJobId() {
        return jobId;
    }
}
```
``native-lib.cpp``
```c++
#include <jni.h>

extern "C"
JNIEXPORT jint JNICALL
Java_com_example_myjniproject_MainActivity_getJobId(JNIEnv *env, jobject thiz, jobject job) {

    // 根据实力获取 class 对象
    jclass jobClz = env->GetObjectClass(job);
    // 根据类名获取 class 对象
    jclass jobClz = env->FindClass("com/example/myjniproject/MyJob");

    // 获取属性 id
    jfieldID fieldId = env->GetFieldID(jobClz, "jobId", "I");
    // 获取静态属性 id
    jfieldID sFieldId = env->GetStaticFieldID(jobClz, "JOB_STRING", "Ljava/lang/String;");

    // 获取方法 id
    jmethodID methodId = env->GetMethodID(jobClz, "getJobId", "()I");
    // 获取构造方法 id
    jmethodID  initMethodId = env->GetMethodID(jobClz, "<init>", "(I)V");

    // 根据对象属性 id 获取该属性值
    jint id = env->GetIntField(job, fieldId);
    // 根据对象方法 id 调用该方法
    jint id = env->CallIntMethod(job, methodId);

    // 创建新的对象
    jobject newJob = env->NewObject(jobClz, initMethodId, 10);

    return id;
}
```

## NDK 开发流程
- 在 java 中声明 native 方法
```java
public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Log.d("MainActivity", stringFromJNI());
    }

    private native String stringFromJNI();
}
```

- 在 ``app/src/main`` 目录下新建 cpp 目录，新建相关 cpp 文件，实现相关方法（AS 可用快捷键快速生成）

``native-lib.cpp``
```
#include <jni.h>

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_myjniproject_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

>- 函数名的格式遵循遵循如下规则：Java_包名_类名_方法名。
>- extern "C" 指定采用 C 语言的命名风格来编译，否则由于 C 与 C++ 风格不同，导致链接时无法找到具体的函数
>- JNIEnv*：表示一个指向 JNI 环境的指针，可以通过他来访问 JNI 提供的接口方法
>- jobject：表示 java 对象中的 this
>- JNIEXPORT 和 JNICALL：JNI 所定义的宏，可以在 jni.h 头文件中查找到

- 通过 CMake 或者 ndk-build 构建动态库

## CMake 构建 NDK 项目
> CMake 是一个开源的跨平台工具系列，旨在构建，测试和打包软件，从 Android Studio 2.2 开始，Android Sudio 默认地使用 CMake 与 Gradle 搭配使用来构建原生库。

启动方式只需要在 ``app/build.gradle`` 中添加相关：
```groovy
android {
    ···
    defaultConfig {
        ···
        externalNativeBuild {
            cmake {
                cppFlags ""
            }
        }

        ndk {
            abiFilters 'arm64-v8a', 'armeabi-v7a'
        }
    }
    ···
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}
```

然后在对应目录新建一个 ``CMakeLists.txt`` 文件：
```txt
# 定义了所需 CMake 的最低版本
cmake_minimum_required(VERSION 3.4.1)

# add_library() 命令用来添加库
# native-lib 对应着生成的库的名字
# SHARED 代表为分享库
# src/main/cpp/native-lib.cpp 则是指明了源文件的路径。
add_library( # Sets the name of the library.
        native-lib

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        src/main/cpp/native-lib.cpp)

# find_library 命令添加到 CMake 构建脚本中以定位 NDK 库，并将其路径存储为一个变量。
# 可以使用此变量在构建脚本的其他部分引用 NDK 库
find_library( # Sets the name of the path variable.
        log-lib

        # Specifies the name of the NDK library that
        # you want CMake to locate.
        log)

# 预构建的 NDK 库已经存在于 Android 平台上，因此，无需再构建或将其打包到 APK 中。
# 由于 NDK 库已经是 CMake 搜索路径的一部分，只需要向 CMake 提供希望使用的库的名称，并将其关联到自己的原生库中

# 要将预构建库关联到自己的原生库
target_link_libraries( # Specifies the target library.
        native-lib

        # Links the target library to the log library
        # included in the NDK.
        ${log-lib})
···
```
- [CMake 命令详细信息文档](https://cmake.org/cmake/help/latest/manual/cmake-commands.7.html)

## 常用的 Android NDK 原生 API
| 支持 NDK 的 API 级别 | 关键原生 API | 包括
|--|--|--
| 3 | Java 原生接口 | 	#include <jni.h>
| 3 | Android 日志记录 API	| #include <android/log.h>
| 5 | OpenGL ES 2.0 | #include <GLES2/gl2.h><br>#include <GLES2/gl2ext.h>
| 8 | Android 位图 API | #include <android/bitmap.h>
| 9 | OpenSL ES | #include <SLES/OpenSLES.h><br>#include <SLES/OpenSLES_Platform.h><br>#include <SLES/OpenSLES_Android.h><br>#include <SLES/OpenSLES_AndroidConfiguration.h>
| 9 | 原生应用 API | #include <android/rect.h><br>#include <android/window.h><br>#include<android/native_activity.h><br>···
| 18 | OpenGL ES 3.0 | #include <GLES3/gl3.h><br>#include <GLES3/gl3ext.h>
| 21 | 原生媒体 API | #include <media/NdkMediaCodec.h><br>#include <media/NdkMediaCrypto.h><br>···
| 24 | 原生相机 API | #include <camera/NdkCameraCaptureSession.h><br>#include <camera/NdkCameraDevice.h><br>···
| ···

# 计算机网络基础
## 网络体系的分层结构
| 分层 | 说明
| -- | --
| 应用层（HTTP、FTP、DNS、SMTP 等）| 定义了如何包装和解析数据，应用层是 http 协议的话，则会按照协议规定包装数据，如按照请求行、请求头、请求体包装，包装好数据后将数据传至运输层
| 运输层（TCP、UDP 等） | 运输层有 TCP 和 UDP 两种，分别对应可靠和不可靠的运输。在这一层，一般都是和 Socket 打交道，Socket 是一组封装的编程调用接口，通过它，我们就能操作 TCP、UDP 进行连接的建立等。这一层指定了把数据送到对应的端口号
| 网络层（IP 等） | 这一层IP协议，以及一些路由选择协议等等，所以这一层的指定了数据要传输到哪个IP地址。中间涉及到一些最优线路，路由选择算法等
| 数据链路层（ARP）| 负责把 IP 地址解析为 MAC 地址，即硬件地址，这样就找到了对应的唯一的机器
| 物理层 | 提供二进制流传输服务，也就是真正开始通过传输介质（有线、无线）开始进行数据的传输

## Http 相关
### 请求报文与响应报文
- 请求报文
  
| 名称 | 组成
| -- | --
| 请求行 | 请求方法如 post/get、请求路径 url、协议版本等
| 请求头 | 即 header，里面包含了很多字段
| 请求体 | 发送的数据

- 响应报文

| 名称 | 组成
| -- | --
| 状态行 | 状态码如 200、协议版本等
| 响应头 | 即返回的 header
| 响应体 | 响应的正文数据 |

### 缓存机制
![](https://upload-images.jianshu.io/upload_images/1445840-c3465ef477e24416.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/930/format/webp)

- Cache-control 主要包含以下几个字段：

| 字段 | 说明
| -- | --
| private | 只有客户端可以缓存
| public | 客户端和代理服务器都可以缓存
| max-age | 缓存的过期时间
| no-cache | 需要使用对比缓存来验证缓存数据，如果服务端确认资源没有更新，则返回304，取本地缓存即可，如果有更新，则返回最新的资源。做对比缓存与 Etag 有关。
| no-store | 这个字段打开，则不会进行缓存，也不会取缓存

- Etag：当客户端发送第一次请求时服务端会下发当前请求资源的标识码 Etag ，下次再请求时，客户端则会通过 header 里的 If-None-Match 将这个标识码 Etag 带上，服务端将客户端传来的 Etag 与最新的资源 Etag 做对比，如果一样，则表示资源没有更新，返回304。

### Https
Https 保证了我们数据传输的安全，Https = Http + Ssl，之所以能保证安全主要的原理就是利用了非对称加密算法，平常用的对称加密算法之所以不安全，是因为双方是用统一的密匙进行加密解密的，只要双方任意一方泄漏了密匙，那么其他人就可以利用密匙解密数据。

### Http 2.0
Okhttp 支持配置使用 Http 2.0 协议，Http2.0 相对于 Http1.x 来说提升是巨大的，主要有以下几点：
- **二进制格式**：http1.x 是文本协议，而 http2.0 是二进制以帧为基本单位，是一个二进制协议，一帧中除了包含数据外同时还包含该帧的标识：Stream Identifier，即标识了该帧属于哪个 request,使得网络传输变得十分灵活。
- **多路复用**：多个请求共用一个TCP连接，多个请求可以同时在这个 TCP 连接上并发，一个是解决了建立多个 TCP 连接的消耗问题，一个也解决了效率的问题。
- **header 压缩**：主要是通过压缩 header 来减少请求的大小，减少流量消耗，提高效率。
- **支持服务端推送**

## TCP/IP
IP（Internet Protocol）协议提供了主机和主机间的通信，为了完成不同主机的通信，我们需要某种方式来唯一标识一台主机，这个标识，就是著名的 IP 地址。通过IP地址，IP 协议就能够帮我们把一个数据包发送给对方。

TCP 的全称是 Transmission Control Protocol，TCP 协议在 IP 协议提供的主机间通信功能的基础上，完成这两个主机上进程对进程的通信。

### 三次握手
所谓三次握手(Three-way Handshake)，是指建立一个 TCP 连接时，需要客户端和服务器总共发送3个包。

三次握手的目的是连接服务器指定端口，建立 TCP 连接，并同步连接双方的序列号和确认号，交换 TCP 窗口大小信息。在 socket 编程中，客户端执行 connect() 时。将触发三次握手。

![](https://raw.githubusercontent.com/HIT-Alibaba/interview/master/img/tcp-connection-made-three-way-handshake.png)

- 第一次握手(SYN=1, seq=x):

客户端发送一个 TCP 的 SYN 标志位置 1 的包，指明客户端打算连接的服务器的端口，以及初始序号 X，保存在包头的序列号 (Sequence Number) 字段里。

发送完毕后，客户端进入 ``SYN_SEND`` 状态。

- 第二次握手(SYN=1, ACK=1, seq=y, ACKnum=x+1):

服务器发回确认包(ACK)应答。即 SYN 标志位和 ACK 标志位均为 1。服务器端选择自己 ISN 序列号，放到 Seq 域里，同时将确认序号(Acknowledgement Number)设置为客户的 ISN 加1，即 X+1。 发送完毕后，服务器端进入 ``SYN_RCVD`` 状态。

- 第三次握手(ACK=1，ACKnum=y+1)

客户端再次发送确认包(ACK)，SYN 标志位为 0，ACK 标志位为 1，并且把服务器发来 ACK 的序号字段 +1，放在确定字段中发送给对方，并且在数据段放写 ISN 的 +1

发送完毕后，客户端进入 ESTABLISHED 状态，当服务器端接收到这个包时，也进入 ESTABLISHED 状态，TCP 握手结束。

### 四次挥手
TCP 的连接的拆除需要发送四个包，因此称为四次挥手(Four-way handshake)，也叫做改进的三次握手。客户端或服务器均可主动发起挥手动作，在 socket 编程中，任何一方执行 close() 操作即可产生挥手操作。

![](https://raw.githubusercontent.com/HIT-Alibaba/interview/master/img/tcp-connection-closed-four-way-handshake.png)

- 第一次挥手(FIN=1，seq=x)

假设客户端想要关闭连接，客户端发送一个 FIN 标志位置为1的包，表示自己已经没有数据可以发送了，但是仍然可以接受数据。

发送完毕后，客户端进入 FIN_WAIT_1 状态。

- 第二次挥手(ACK=1，ACKnum=x+1)

服务器端确认客户端的 FIN 包，发送一个确认包，表明自己接受到了客户端关闭连接的请求，但还没有准备好关闭连接。

发送完毕后，服务器端进入 CLOSE_WAIT 状态，客户端接收到这个确认包之后，进入 FIN_WAIT_2 状态，等待服务器端关闭连接。

- 第三次挥手(FIN=1，seq=y)

服务器端准备好关闭连接时，向客户端发送结束连接请求，FIN 置为1。

发送完毕后，服务器端进入 LAST_ACK 状态，等待来自客户端的最后一个ACK。

- 第四次挥手(ACK=1，ACKnum=y+1)

客户端接收到来自服务器端的关闭请求，发送一个确认包，并进入 TIME_WAIT状态，等待可能出现的要求重传的 ACK 包。

服务器端接收到这个确认包之后，关闭连接，进入 CLOSED 状态。

客户端等待了某个固定时间（两个最大段生命周期，2MSL，2 Maximum Segment Lifetime）之后，没有收到服务器端的 ACK ，认为服务器端已经正常关闭连接，于是自己也关闭连接，进入 CLOSED 状态。

### TCP 与 UDP 的区别
| 区别点    | TCP      | UDP    |
| -------- | -------- | ------ |
| 连接性   | 面向连接 | 无连接 |
| 可靠性   | 可靠     | 不可靠|
| 有序性   | 有序     | 无序   |
| 面向     | 字节流     | 报文（保留报文的边界） |
| 有界性   | 有界     | 无界   |
| 流量控制 | 有（滑动窗口） | 无     |
| 拥塞控制 | 有（慢开始、拥塞避免、快重传、快恢复）       | 无 |
| 传输速度 | 慢       | 快     |
| 量级     | 重量级   | 轻量级 |
| 双工性     | 全双工   | 一对一、一对多、多对一、多对多 |
| 头部 | 大（20-60 字节）       | 小（8 字节）     |
| 应用 | 文件传输、邮件传输、浏览器等 | 即时通讯、视频通话等     |

## Socket
Socket 是一组操作 TCP/UDP 的 API，像 HttpURLConnection 和 Okhttp 这种涉及到比较底层的网络请求发送的，最终当然也都是通过 Socket 来进行网络请求连接发送，而像 Volley、Retrofit 则是更上层的封装。

### 使用示例
使用 socket 的步骤如下：
- 创建 ServerSocket 并监听客户连接；
- 使用 Socket 连接服务端；
- 通过 Socket.getInputStream()/getOutputStream() 获取输入输出流进行通信。

```java
public class EchoClient {
 
    private final Socket mSocket;
 
    public EchoClient(String host, int port) throws IOException {
        // 创建 socket 并连接服务器
        mSocket = new Socket(host, port);
    }
 
    public void run() {
        // 和服务端进行通信
        Thread readerThread = new Thread(this::readResponse);
        readerThread.start();
 
        OutputStream out = mSocket.getOutputStream();
        byte[] buffer = new byte[1024];
        int n;
        while ((n = System.in.read(buffer)) > 0) {
            out.write(buffer, 0, n);
        }
    }

    private void readResponse() {
        try {
            InputStream in = mSocket.getInputStream();
            byte[] buffer = new byte[1024];
            int n;
            while ((n = in.read(buffer)) > 0) {
                System.out.write(buffer, 0, n);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
 
 
    public static void main(String[] argv) {
        try {
            // 由于服务端运行在同一主机，这里我们使用 localhost
            EchoClient client = new EchoClient("localhost", 9877);
            client.run();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

# 类加载器
![](https://img-blog.csdn.net/20161021101447117?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 双亲委托模式
某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。

因为这样可以避免重复加载，当父亲已经加载了该类的时候，就没有必要子 ClassLoader 再加载一次。如果不使用这种委托模式，那我们就可以随时使用自定义的类来动态替代一些核心的类，存在非常大的安全隐患。

## DexPathList
DexClassLoader 重载了 ``findClass`` 方法，在加载类时会调用其内部的 DexPathList 去加载。DexPathList 是在构造 DexClassLoader 时生成的，其内部包含了 DexFile。

``DexPathList.java``
```java
···
public Class findClass(String name) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    return null;
}
···
```

