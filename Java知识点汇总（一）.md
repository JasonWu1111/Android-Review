# jvm
## jvm工作流程
![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a4906226?w=448&h=592&f=jpeg&s=44057)

## 运行时数据区（Runtime Data Area）
![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a499f6fe?w=868&h=497&f=webp&s=46378)

| 区域 | 说明                      
|----------|-----|
| 程序计数器 | 每条线程都需要有一个程序计数器，计数器记录的是正在执行的指令地址，如果正在执行的是Natvie 方法，这个计数器值为空（Undefined） |
| java虚拟机栈 | Java方法执行的内存模型，每个方法执行的时候，都会创建一个栈帧用于保存局部变量表，操作数栈，动态链接，方法出口信息等。一个方法调用的过程就是一个栈帧从VM栈入栈到出栈的过程 |
| 本地方法栈 | 与VM栈发挥的作用非常相似，VM栈执行Java方法（字节码）服务，Native方法栈执行的是Native方法服务。| 
| Java堆 | 此内存区域唯一的目的就是存放对象实例，几乎所有的对象都在这分配内存 |
| 方法区 | 方法区是各个内存所共享的内存空间，方法区中主要存放被JVM加载的类信息、常量、静态变量、即时编译后的代码等数据 | 

## 方法指令
| 指令 | 说明                      
|----------|-----|
| invokeinterface | 用以调用接口方法 |
| invokevirtual | 指令用于调用对象的实例方法 |
| invokestatic | 用以调用类/静态方法 |
| invokespecial | 用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法 | 

## 类加载器
| 类加载器 | 说明                      
|----------|-----|
| BootstrapClassLoader | Bootstrap类加载器负责加载rt.jar中的JDK类文件，它是所有类加载器的父加载器。Bootstrap类加载器没有任何父类加载器，如果你调用String.class.getClassLoader()，会返回null，任何基于此的代码会抛出NUllPointerException异常。Bootstrap加载器被称为初始类加载器 |
| ExtClasssLoader | 而Extension将加载类的请求先委托给它的父加载器，也就是Bootstrap，如果没有成功加载的话，再从jre/lib/ext目录下或者java.ext.dirs系统属性定义的目录下加载类。Extension加载器由sun.misc.Launcher$ExtClassLoader实现 |
| AppClassLoader | 第三种默认的加载器就是System类加载器（又叫作Application类加载器）了。它负责从classpath环境变量中加载某些应用相关的类，classpath环境变量通常由-classpath或-cp命令行选项来定义，或者是JAR中的Manifest的classpath属性。Application类加载器是Extension类加载器的子加载器 |
&nbsp;
| 工作原理 | 说明                      
|----------|------|
| 委托机制 | 加载任务委托交给父类加载器，如果不行就向下传递委托任务，由其子类加载器加载，保证java核心库的安全性 |
| 可见性机制 | 子类加载器可以看到父类加载器加载的类，而反之则不行 |
| 单一性机制 | 父加载器加载过的类不能被子加载器加载第二次 |

# static
- static关键字修饰的方法或者变量不需要依赖于对象来进行访问，只要类被加载了，就可以通过类名去进行访问。
- 静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初次加载时会被初始化。
- 能通过this访问静态成员变量吗?
所有的静态方法和静态变量都可以通过对象访问（只要访问权限足够）。
- static是不允许用来修饰局部变量

# final
- 可以声明成员变量、方法、类以及本地变量
- final成员变量必须在声明的时候初始化或者在构造器中初始化，否则就会报编译错误
- final变量是只读的
- final申明的方法不可以被子类的方法重写
- final类通常功能是完整的，不能被继承
- final变量可以安全的在多线程环境下进行共享，而不需要额外的同步开销
- final关键字提高了性能，JVM和Java应用都会缓存final变量，会对方法、变量及类进行优化
- 方法的内部类访问方法中的局部变量，但必须用final修饰才能访问

# String、StringBuffer、StringBuilder
- String是final类，不能被继承。对于已经存在的Stirng对象，修改它的值，就是重新创建一个对象
- StringBuffer是一个类似于String的字符串缓冲区，使用append()方法修改Stringbuffer的值，使用toString()方法转换为字符串，是线程安全的
- StringBuilder用来替代于StringBuffer，StringBuilder是非线程安全的，速度更快

# 异常处理
- Exception、Error是Throwable类的子类
- Error类对象由Java虚拟机生成并抛出，不可捕捉  
- 不管有没有异常，finally中的代码都会执行
- 当try、catch中有return时，finally中的代码依然会继续执行


| 常见的Error | | |
|------|-----|-----|
| OutOfMemoryError | StackOverflowError | NoClassDeffoundError |


| 常见的Exception | | |
|------|-----|-----|
| 常见的非检查性异常 |  |
| ArithmeticException | ArrayIndexOutOfBoundsException | ClassCastException |
| IllegalArgumentException | IndexOutOfBoundsException | NullPointerException |
| NumberFormatException | SecurityException | UnsupportedOperationException |
| 常见的检查性异常 |  |
| IOException | CloneNotSupportedException | IllegalAccessException |
| NoSuchFieldException | NoSuchMethodException | FileNotFoundException

# 内部类
- 内部类提供了更好的封装，可以把内部类隐藏在外部类之内，不允许同一个包中的其他类访问该类。
- 内部类的方法可以直接访问外部类的所有数据，包括私有的数据。

# 多态
- 父类的引用可以指向子类的对象
- 创建子类对象时，调用的方法为子类重写的方法或者继承的方法
- 如果我们在子类中编写一个独有的方法，此时就不能通过父类的引用创建的子类对象来调用该方法

# 抽象和接口
- 抽象类不能有对象（不能用new此关键字来创建抽象类的对象）
- 抽象类中的抽象方法必须在子类中被重写
- 接口中的所有属性默认为：public static final ****；
- 接口中的所有方法默认为：public abstract ****；

# 集合
![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a86db5e6?w=643&h=611&f=gif&s=22445)
- List接口存储一组不唯一，有序（插入顺序）的对象, Set接口存储一组唯一，无序的对象。
- HashMap是非synchronized的，性能更好，HashMap可以接受为null的key和value，而Hashtable是线程安全的，比HashMap要慢，不接受null

# 反射
```java
try {
    Class cls = Class.forName("com.jasonwu.Test");
    //获取构造方法
    Constructor[] publicConstructors = cls.getConstructors();
    //获取全部构造方法
    Constructor[] declaredConstructors = cls.getDeclaredConstructors();
    //获取公开方法
    Method[] methods = cls.getMethods();
    //获取全部方法
    Method[] declaredMethods = cls.getDeclaredMethods();
    //获取公开属性
    Field[] publicFields = cls.getFields();
    //获取全部属性
    Field[] declaredFields = cls.getDeclaredFields();
    Object clsObject = cls.newInstance();
    Method method = cls.getDeclaredMethod("getModule1Functionality");
    Object object = method.invoke(null);
} catch (ClassNotFoundException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
}
```

# 单例
## 饿汉式
```java
public class CustomManager {
    private Context mContext;
    private static final Object mLock = new Object();
    private static CustomManager mInstance;

    public static CustomManager getInstance(Context context) {
        synchronized (mLock) {
            if (mInstance == null) {
                mInstance = new CustomManager(context);
            }

            return mInstance;
        }
    }

    private CustomManager(Context context) {
        this.mContext = context.getApplicationContext();
    }
}
```
## 双重检查模式
```java
public class CustomManager {
    private Context mContext;
    private volatile static CustomManager mInstance;

    public static CustomManager getInstance(Context context) {
        // 避免非必要加锁
        if (mInstance == null) {
            synchronized (CustomManger.class) {
                if (mInstance == null) {
                    mInstacne = new CustomManager(context);
                }
            }
        }

        return mInstacne;
    }

    private CustomManager(Context context) {
        this.mContext = context.getApplicationContext();
    }
}
```
## 静态内部类模式
```java
public class CustomManager{
    private CustomManager(){}
 
    private static class CustomManagerHolder {
        private static CustomManager INSTANCE = new CustomManager();
    }
 
    public static CustomManager getInstance() {
        return CustomManagerHolder.INSTANCE;
    } 
}
```
静态内部类的原理是：

当SingleTon第一次被加载时，并不需要去加载SingleTonHoler，只有当getInstance()方法第一次被调用时，才会去初始化INSTANCE，这种方法不仅能确保线程安全，也能保证单例的唯一性，同时也延迟了单例的实例化。getInstance()方法并没有多次去new对象，取的都是同一个INSTANCE对象。

虚拟机会保证一个类的``<clinit>()``方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的``<clinit>()``方法，其他线程都需要阻塞等待，直到活动线程执行``<clinit>()``方法完毕

缺点在于无法传递参数，如Context等

# 线程
![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a48de8f9?w=700&h=480&f=jpeg&s=103460)

# valatile
当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，JVM 保证了每次读变量都从内存中读，跳过 CPU cache 这一步，因此在读取volatile类型的变量时总会返回最新写入的值。

![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4a48b216e?w=550&h=429&f=png&s=21448)

当一个变量定义为 volatile 之后，将具备以下特性：
- 保证此变量对所有的线程的可见性，不能保证它具有原子性（可见性，是指线程之间的可见性，一个线程修改的状态对另一个线程是可见的）
- 禁止指令重排序优化
- volatile 的读性能消耗与普通变量几乎相同，但是写操作稍慢，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行

AtomicInteger 中主要实现了整型的原子操作，防止并发情况下出现异常结果，其内部主要依靠JDK 中的unsafe 类操作内存中的数据来实现的。volatile 修饰符保证了value在内存中其他线程可以看到其值得改变。CAS操作保证了AtomicInteger 可以安全的修改value 的值。

# HashMap
![](https://user-gold-cdn.xitu.io/2019/6/23/16b833f4ac8f44fd?w=1636&h=742&f=png&s=88323)
## HashMap的工作原理
HashMap基于hashing原理，我们通过put()和get()方法储存和获取对象。当我们将键值对传递给put()方法时，它调用键对象的hashCode()方法来计算hashcode，让后找到bucket位置来储存Entry对象。当两个对象的hashcode相同时，它们的bucket位置相同，‘碰撞’会发生。因为HashMap使用链表存储对象，这个Entry会存储在链表中，当获取对象时，通过键对象的equals()方法找到正确的键值对，然后返回值对象。

**如果HashMap的大小超过了负载因子(load factor)定义的容量，怎么办？**  
默认的负载因子大小为0.75，也就是说，当一个map填满了75%的bucket时候，和其它集合类(如ArrayList等)一样，将会创建原来HashMap大小的两倍的bucket数组，来重新调整map的大小，并将原来的对象放入新的bucket数组中。这个过程叫作rehashing，因为它调用hash方法找到新的bucket位置。

**为什么String, Interger这样的wrapper类适合作为键?**  
因为String是不可变的，也是final的，而且已经重写了equals()和hashCode()方法了。其他的wrapper类也有这个特点。不可变性是必要的，因为为了要计算hashCode()，就要防止键值改变，如果键值在放入时和获取时返回不同的hashcode的话，那么就不能从HashMap中找到你想要的对象。不可变性还有其他的优点如线程安全。如果你可以仅仅通过将某个field声明成final就能保证hashCode是不变的，那么请这么做吧。因为获取对象的时候要用到equals()和hashCode()方法，那么键对象正确的重写这两个方法是非常重要的。如果两个不相等的对象返回不同的hashcode的话，那么碰撞的几率就会小些，这样就能提高HashMap的性能。

# synchronized
当它用来修饰一个方法或者一个代码块的时候，能够保证在同一时刻最多只有一个线程执行该段代码。

在 Java 中，每个对象都会有一个 monitor 对象，这个对象其实就是 Java 对象的锁，通常会被称为“内置锁”或“对象锁”。类的对象可以有多个，所以每个对象有其独立的对象锁，互不干扰。针对每个类也有一个锁，可以称为“类锁”，类锁实际上是通过对象锁实现的，即类的 Class 对象锁。每个类只有一个 Class 对象，所以每个类只有一个类锁。

Monitor是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联，同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。Monitor是依赖于底层的操作系统的Mutex Lock（互斥锁）来实现的线程同步。

## 根据获取的锁分类
**获取对象锁**
- synchronized(this|object) {}  
- 修饰非静态方法  

**获取类锁**
- synchronized(类.class) {}  
- 修饰静态方法

## 原理
**同步代码块：**
- monitorenter和monitorexit指令实现的

**同步方法**
- 方法修饰符上的ACC_SYNCHRONIZED实现

# Lock
![](https://user-gold-cdn.xitu.io/2019/6/18/16b69b50c9d340a5?w=1372&h=1206&f=png&s=142754)

## 悲观锁、乐观锁
悲观锁认为自己在使用数据的时候一定有别的线程来修改数据，因此在获取数据的时候会先加锁，确保数据不会被别的线程修改。Java中，synchronized关键字和Lock的实现类都是悲观锁。悲观锁适合写操作多的场景，先加锁可以保证写操作时数据正确。

而乐观锁认为自己在使用数据时不会有别的线程修改数据，所以不会添加锁，只是在更新数据的时候去判断之前有没有别的线程更新了这个数据。如果这个数据没有被更新，当前线程将自己修改的数据成功写入。如果数据已经被其他线程更新，则根据不同的实现方式执行不同的操作（例如报错或者自动重试）。乐观锁在Java中是通过使用无锁编程来实现，最常采用的是CAS算法，Java原子类中的递增操作就通过CAS自旋实现。乐观锁适合读操作多的场景，不加锁的特点能够使其读操作的性能大幅提升。

## 自旋锁、适应性自旋锁
阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态转换需要耗费处理器时间。如果同步代码块中的内容过于简单，状态转换消耗的时间有可能比用户代码执行的时间还要长。

在许多场景中，同步资源的锁定时间很短，为了这一小段时间去切换线程，线程挂起和恢复现场的花费可能会让系统得不偿失。如果物理机器有多个处理器，能够让两个或以上的线程同时并行执行，我们就可以让后面那个请求锁的线程不放弃CPU的执行时间，看看持有锁的线程是否很快就会释放锁。

而为了让当前线程“稍等一下”，我们需让当前线程进行自旋，如果在自旋完成后前面锁定同步资源的线程已经释放了锁，那么当前线程就可以不必阻塞而是直接获取同步资源，从而避免切换线程的开销。这就是自旋锁。

自旋锁本身是有缺点的，它不能代替阻塞。自旋等待虽然避免了线程切换的开销，但它要占用处理器时间。如果锁被占用的时间很短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白浪费处理器资源。所以，自旋等待的时间必须要有一定的限度，如果自旋超过了限定次数（默认是10次，可以使用-XX:PreBlockSpin来更改）没有成功获得锁，就应当挂起线程。

自旋锁的实现原理同样也是CAS，AtomicInteger中调用unsafe进行自增操作的源码中的do-while循环就是一个自旋操作，如果修改数值失败则通过循环来执行自旋，直至修改成功。

## 死锁
当前线程拥有其他线程需要的资源，当前线程等待其他线程已拥有的资源，都不放弃自己拥有的资源。

# 引用类型
强引用 > 软引用 > 弱引用 
| 引用类型 | 说明
|------|------
| StrongReferenc（强引用）| 当一个对象具有强引用，那么垃圾回收器是绝对不会的回收和销毁它的，**非静态内部类会在其整个生命周期中持有对它外部类的强引用**
| WeakReference （弱引用）| 在垃圾回收器运行的时候，如果对一个对象的所有引用都是弱引用的话，该对象会被回收 |
| SoftReference（软引用）| 如果一个对象只具有软引用，若内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，才会回收这些对象的内存
| PhantomReference（虚引用） | 一个只被虚引用持有的对象可能会在任何时候被GC回收。虚引用对对象的生存周期完全没有影响，也无法通过虚引用来获取对象实例，仅仅能在对象被回收时，得到一个系统通知（只能通过是否被加入到ReferenceQueue来判断是否被GC，这也是唯一判断对象是否被GC的途径）。|


