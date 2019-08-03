# Hook
## 基本流程
1、根据需求确定 要hook的对象  
2、寻找要hook的对象的持有者，拿到要hook的对象  
3、定义“要hook的对象”的代理类，并且创建该类的对象  
4、使用上一步创建出来的对象，替换掉要hook的对象

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
