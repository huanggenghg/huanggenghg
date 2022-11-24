## Android 负一屏

### 一、IPC

[IPC 机制](https://github.com/huanggenghg/huanggenghg/blob/main/IPC%20%E6%9C%BA%E5%88%B6.md)

### 二、Google Feed 屏方案

Binder 的跨进程通信的过程中，有两个 Binder 对象，Google 提供了负一屏与桌面 Launcher      跨进程通信所需的两个 aidi 接口

```java
// com.google.android.libraries.launcherclient
interface ILauncherOverlay {
    oneway void startScroll(); // Launcher 主动开始滑动，调此方法通知 Overlay
    oneway void onScroll(in float progress); // Launcher 滑动的进度
    oneway void endScroll(); // Launcher 停止滑动，调此方法通知 Overlay
    oneway void windowAttached(in LayoutParams lp, in ILauncherOverlayCallback cb, in int flags);
    oneway void windowDetached(in boolean isChangingConfigurations);
    oneway void closeOverlay(in int flags);
    oneway void onPause();
    oneway void onResume();
    oneway void openOverlay(in int flags);
    oneway void requestVoiceDetection(in boolean start);
    String getVoiceSearchLanguage();
    boolean isVoiceDetectionRunning();
    boolean hasOverlayContent();
    oneway void windowAttached2(in Bundle bundle, in ILauncherOverlayCallback cb);
    oneway void unusedMethod();
    oneway void setActivityState(in int flags);
    boolean startSearch(in byte[] data, in Bundle bundle);
}

interface ILauncherOverlayCallback {
    oneway void overlayScrollChanged(float progress); // Overlay 主动滑动的进度，Overlay 通过此接口回调给 Launcher
    oneway void overlayStatusChanged(int status); // Overlay 的滑动状态，比如开始滑动等，Overlay 通过此接口回调给 Launcher
}
```

![负一屏与Launcher之间的通讯模型](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/16/1721e2ea465bcfe5~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

能在桌面显示：

![Window 显示 View](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/16/1721e345d22ee262~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

日常开发一个 Activity 页面的时候，都是通过各类的 View 来实现，而这些 View 都在应用进程里，对应到系统进程里（WMS），只是在一个 Window 里。Window 是有分层，多个 Window 之间有 Z 轴顺序的，WindowManagerSeervice 就是来管理这些 Window 的，处于上层的 Window 会覆盖下层的 Window，而后由 SurfaceFlinger 负责绘制。

```java
// 负一屏
WindowManager.LayoutParams mLayoutParams;

public void windowAttached(WindowManager.LayoutParams lp, ILauncherOverlayCallback cb, int flags) {
    mLayoutParams.width = ViewGroup.LayoutParams.MATCH_PARENT;
    mLayoutParams.height = ViewGroup.LayoutParams.MATCH_PARENT;
    mLayoutParams.gravity = Gravity.START;
    // 负一屏的 Window 层级比 Launcher 的大就可以
    mLayoutParams.type = lp.type + 1; 
    mLayoutParams.token = lp.token;
    mLayoutParams.flags = WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS | 
        WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE | 
        WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS | 
        WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS |
        WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION;
    mLayoutParams.x = -screenWidth; // 避免遮盖桌面
    mLayoutParams.format = PixelFormat.TRANSLUCENT;
    
    mWindowManager.addView(mOverlayDecorView, mLayoutParams);
    
    if (cb != null) {
        cb.overlayStatusChanged(FLAG_SUCCESS);
    }
}
```

![负一屏滑动过程](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/17/1721e4352b16ed7d~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

在开始滑动显示负一屏的时候，负一屏 Window 的 LayoutParms.x 设为 0，直接覆盖在桌面 Window 上方，但负一屏 Window 中 View 的 scrollX 一开始需要设为 screenWidth，即内容不可见，随着桌面的滑动，改变此 View 的 scrollX 参数使得负一屏的内容逐步显示出来

### 反射方案

反射方案，指的是负一屏提供一个固定的类、固定的接口给 Launcher。

```java
public class AssistantCtrl {
    public View createView() {
        // ... ...
    }
}
```

此接口会返回负一屏的页面 View，Launcher 通过反射获取到 AssistantCtrl 对象，然后反射调用 createView() 方法获取到负一屏页面 View，并添加到自己的 View 里。

```java
Context foreignContext = context.createPackageContext("负一屏应用包名", flag); // 获取负一屏 context，拿到相关资源索引
Class cls = foreignContext.getClassLoader().loadClass("Assistant 类全路径名");
```

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/5/17/1721e5f6e7a23d14~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

Launcher 通过反射获取到负一屏 View 的实例，所以负一屏 View 页面是运行在 Launcher 进程里，而 View 中的数据需要从负一屏获取，所以这里也需要做一层跨进程通信，同样也是用到 Binder 实现。

```java
public abstract Context createPackageContext (String packageName, 
                int flags)
/*
return a new Context object for the given application name. This Context is the same as what the named application gets when it is launched, containing the same resources and class loader. Each call to this method returns a new instance of a Context object; Context objects are not shared, however they share common state (Resources, ClassLoader, etc) so the Context instance itself is fairly lightweight.

Throws PackageManager.NameNotFoundException if there is no application with the given package name.

Throws SecurityException if the Context requested can not be loaded into the caller's process for security reasons (see CONTEXT_INCLUDE_CODE for more information}.
*/
```

### 对比

Google Feed 屏方案
 优点：负一屏和 Luancher 的解耦；负一屏页面是加载在负一屏进程。
 缺点：Launcher 和负一屏之前的滑动切换不好处理，在滑动过程不断通过跨进程通信来达到两者间的滑动同步，没有反射方案那么流畅。

反射方案
 优点：负一屏的 View 直接加载在 Launcher 进程里，所以 Launcher 和负一屏页面间的切换流畅易实现。
 缺点：负一屏的 View 直接加载在 Launcher 里，占用 Launcher 的内存；负一屏和 Launcher 耦合度高，不便于拓展；需要 Launcher 提供其他的能力，如权限，比如负一屏的图片来源于网络，这就是要桌面有请求网络的能力。

*以上摘录自[负一屏和桌面交互实现原理](https://juejin.cn/post/6876749370734493709)*

