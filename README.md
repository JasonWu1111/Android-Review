# Android复习资料
接触 Android 开发也有一段时间了，前段时间便开始想抽空整理一些知识点，通过笔记整理的方式减少自己重复学习的时间成本和提高自身的效率。

整理的知识点会有Java、Android SDK、Android 源码、其他的一些计算机基础以及常见的面试题等几个部分，往后的一个月时间里会陆续补充更新。

目录：  
* [jvm](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#jvm)
   * [jvm工作流程](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#jvm工作流程)
   * [运行时数据区（Runtime Data Area）](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#运行时数据区runtime-data-area)
   * [方法指令](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#方法指令)
   * [类加载器](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#类加载器)
* [static](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#static)
* [final](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#final)
* [String、StringBuffer、StringBuilder](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#stringstringbufferstringbuilder)
* [异常处理](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#异常处理)
* [内部类](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#内部类)
* [多态](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#多态)
* [抽象和接口](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#抽象和接口)
* [集合](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#集合)
* [反射](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#反射)
* [单例](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#单例)
   * [饿汉式](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#饿汉式)
   * [双重检查模式](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#双重检查模式)
   * [静态内部类模式](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#静态内部类模式)
* [线程](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#线程)
* [valatile](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#valatile)
* [HashMap](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#hashmap)
   * [HashMap的工作原理](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#hashmap的工作原理)
* [synchronized](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#synchronized)
   * [根据获取的锁分类](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#根据获取的锁分类)
   * [原理](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#原理)
* [Lock](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#lock)
   * [悲观锁、乐观锁](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#悲观锁乐观锁)
   * [自旋锁、适应性自旋锁](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#自旋锁适应性自旋锁)
   * [死锁](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#死锁)
* [引用类型](https://github.com/JasonWu1111/Android-Review/blob/master/Java%20知识点汇总.md#引用类型)

- [Android 知识点汇总](https://github.com/JasonWu1111/Android-Review/blob/master/Android%20%E7%9F%A5%E8%AF%86%E7%82%B9%E6%B1%87%E6%80%BB.md)

- [常见面试算法题汇总](https://github.com/JasonWu1111/Android-Review/blob/master/%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E7%AE%97%E6%B3%95%E9%A2%98%E6%B1%87%E6%80%BB.md)
