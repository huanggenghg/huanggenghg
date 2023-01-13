## Android 进阶解密 读书笔记

### ch1 Android 系统架构

应用层、应用框架层、系统运行层、硬件抽象层和 Linux 内核层。

### ch2 Android 系统启动

#### init 进程启动过程

启动电源以及系统启动、引导程序 BootLoader、Linux 内核启动、init 进程启动。

init 的`main`函数，在开始的时候创建和挂载启动所需的文件目录，解析 init.rc 文件，解析 Service 类型语句。

init 启动 Zygote \ˈzaɪɡoʊt\ ，fork 创建子进程
启动属性服务

**总结**：
1. 创建和挂载启动所需的文件目录
2. 初始化和启动属性服务
3. 解析 init.rc 配置文件并启动 Zygote 进程

#### Zygote 进程启动过程

在 Android 系统中，DVM(Dalvik 虚拟机) 和 ART、应用程序进程以及运行系统的关键服务的 SystemServer 进程都是由 Zygote 进程来创建的，我们也将它称为孵化器。它通过 fock（复制进程）的形式来创建应用程序进程和 SystemServer 进程， Zygote 进程的名称并不是叫“zygote”，而是叫“app_process”，这个名称是在 Android.mk 中定义的。

Zygote 进程都是通过 fock 自身来创建子进程的，这样 Zygote 进程以及它的子进程都可以进入 app_main.cpp 的 main 函数。

这里为何要使用JNI呢？因为ZygoteInit的main方法是由Java语言编写的，当前的运行逻辑在Native中，这就需要通过JNI来调用Java。这样Zygote就从Native层进入了Java框架层。Zygote开创了Java框架层。

**ZygoteInit的main方法主要做了4件事**：
1. 创建一个Server端的Socket。`registerZygoteSocket`，在 Zygote 进程将 SystemServer 进程启动后，就会在这个服务端的 Socket 上等待 AMS 请求 Zygote 进程来创建新的应用进程。
2. 预加载类和资源。
3. 启动SystemServer进程。调用Zygote的forkSystemServer方法，其内部会调用nativeForkSystemServer 这个Native 方法，nativeForkSystemServer方法最终会通过fork函数在当前进程创建一个子进程。
4. 等待AMS请求创建新的应用程序进程。`runSelectLoop`，无限循环等待 AMS 请求。

**总结**：
1. 创建AppRuntime并调用其start方法，启动Zygote进程。
2. 创建Java虚拟机并为Java虚拟机注册JNI方法。
3. 通过JNI调用ZygoteInit的main函数进入Zygote的Java框架层。
4. 通过registerZygoteSocket方法创建服务器端Socket，并通过runSelectLoop方法等待AMS的请求来创建新的应用程序进程。
5. 启动SystemServer进程。

#### SystemServer 处理过程

SystemServer进程主要用于创建系统服务，我们熟知的AMS、WMS和PMS都是由它来创建的。

**总结**
SystemServer进程被创建后，主要做了如下工作：
1. 启动Binder线程池，这样就可以与其他进程进行通信。（Native 调用启动 Binder 线程池）
2. 创建SystemServiceManager，其用于对系统的服务进行创建、启动和生命周期管理。
3. 启动各种系统服务。（引导服务、核心服务、其他服务）

#### Launcher 启动过程

概述：
系统启动的最后一步是启动一个应用程序用来显示系统中已经安装的应用程序，这个应用程序就叫作Launcher。Launcher在启动过程中会请求PackageManagerService返回系统中已经安装的应用程序的信息，并将这些信息封装成一个快捷图标列表显示在系统屏幕上，这样用户可以通过点击这些快捷图标来启动相应的应用程序。

1. 作为Android系统的启动器，用于启动应用程序。
2. 作为Android系统的桌面，用于显示和管理应用程序的快捷图标或者其他桌面组件。

Launcher的AndroidManifest文件中的intent-filter标签匹配了Action为Intent.ACTION_MAIN，Category为Intent.CATEGORY_HOME。

Launcher 中应用图标显示过程：
Launcher 是用工作区的形式来显示系统安装的应用程序的快捷图标的，每一个工作区都是用来描述一个抽象桌面的，它由n个屏幕组成，每个屏幕又分为n个单元格，每个单元格用来显示一个应用程序的快捷图标。

#### Android 系统启动流程

![Android 系统启动流程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/c63cc8745f08441bb9b7e30bb13c9d750167f7cc/res/Android%20%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.png)

### 应用程序进程启动过程

要想启动一个应用程序，首先要保证这个应用程序所需要的应用程序进程已经启动。AMS在启动应用程序时会检查这个应用程序需要的应用程序进程是否存在，不存在就会请求Zygote进程启动需要的应用程序进程。在2.2节中，我们知道在Zygote的Java框架层中会创建一个Server端的Socket，这个Socket用来等待AMS请求Zygote来创建新的应用程序进程。Zygote进程通过fock自身创建应用程序进程，这样应用程序进程就会获得Zygote进程在启动时创建的虚拟机实例。当然，在应用程序进程创建过程中除了获取虚拟机实例外，还创建了Binder线程池和消息循环，这样运行在应用进程中的应用程序就可以方便地使用Binder进行进程间通信以及处理消息了。

AMS 发送启动应用进程请求：
![AMS 发送启动应用进程请求](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/AMS%20%E5%90%AF%E5%8A%A8%E5%BA%94%E7%94%A8%E8%BF%9B%E7%A8%8B.png)

Zygote 接收请求并创建应用程序进程
![Zygote 接收请求并创建应用程序进程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Zygote%E6%8E%A5%E6%94%B6%E8%AF%B7%E6%B1%82%E5%88%9B%E5%BB%BA%E5%BA%94%E7%94%A8%E8%BF%9B%E7%A8%8B.png)

通过registerZygoteSocket方法创建了一个Server端的Socket，这个name 为“zygote”的Socket用来等待AMS请求Zygote，以创建新的应用程序进程；`preload(bootTimingTraceLog)`预加载类和资源；启动SystemServer进程，这样系统的服务也会由SystemServer进程启动起来；调用ZygoteServer的runSelectLoop方法来等待AMS请求创建新的应用程序进程。

`throw new Zygote.MethodAndArgsCaller(m, argv)`，这种抛出异常的处理会清除所有的设置过程需要的堆栈帧，并让ActivityThread的main方法看起来像是应用程序进程的入口方法。

#### Binder 线程池启动过程

Binder线程为一个PoolThread。PoolThread类继承了Thread类。调用IPCThreadState的joinThreadPool函数，将当前线程注册到Binder驱动程序中，这样我们创建的线程就加入了Binder线程池中，新创建的应用程序进程就支持Binder进程间通信了，我们只需要创建当前进程的Binder对象，并将它注册到ServiceManager中就可以实现Binder进程间通信，而不必关心进程间是如何通过Binder进行通信的。

#### 消息循环创建过程

```java
Looper.prepareMainLooper();

ActivityThreaed thread = new ActivityThread();

sMainThreadHandler = thread.getHandler();

Looper.loop();
```

### ch4 四大组件的工作过程

#### 4.1 根 Activity 的启动过程

![点击桌面图标](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%82%B9%E5%87%BB%E6%A1%8C%E9%9D%A2%E5%9B%BE%E6%A0%87.png)

`intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);`
调用Instrumentation的execStartActivity方法，Instrumentation 主要用来监控应用程序和系统的交互。

execStartActivity方法最终调用的是AMS 的startActivity方法。

![AMS 到 ApplicationThread 的调用过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/AMS%20%E5%88%B0%20ApplicationThread%20%E7%9A%84%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B.png)

在AMS的startActivity方法中返回了startActivityAsUser方法，可以发现startActivityAsUser方法比startActivity方法多了一个参数UserHandle.getCallingUserId（），这个方法会获得调用者的UserId，AMS根据这个UserId来确定调用者的权限。

ActivityStarter是Android 7.0中新加入的类，它是加载Activity的控制类，会收集所有的逻辑来决定如何将Intent和Flags转换为Activity，并将Activity和Task以及Stack相关联。

启动理由不能为空

```java
//...
if(caller != null) { // caller 指向的是 Launcher 所在的应用程序进程的 ApplicationThread 对象
	callerApp = mService.getRecordForAppLocked(caller); // 得到的是代表 Launcher 进程的 callerApp 对象，它是 ProcessRecord 类型，用于描述一个应用程序进程

	//...
	ActivityRecord r = new ActivityRecord(...); //ActivityRecord用于描述一个Activity，用来记录一个Activity 的所有信息。创建ActivityRecord，用于描述将要启动的Activity。
}
```

启动根Activity时会将Intent的Flag设置为FLAG_ACTIVITY_NEW_TASK；接着执行setTaskFromReuseOrCreateNewTask方法，其内部会创建一个新的TaskRecord，用来描述一个Activity任务栈，也就是说setTaskFromReuseOrCreateNewTask方法内部会创建一个新的Activity任务栈。

IApplicationThread，它的实现是ActivityThread 的内部类ApplicationThread，其中ApplicationThread继承了IApplicationThread.Stub。app指的是传入的要启动的Activity所在的应用程序进程，因此，这段代码指的就是要在目标应用程序进程启动Activity。当前代码逻辑运行在AMS 所在的进程（SystemServer 进程）中，通过ApplicationThread来与应用程序进程进行Binder通信，换句话说，**ApplicationThread是AMS所在进程（SystemServer进程）和应用程序进程的通信桥梁**。

![AMS 与应用程序进程通信](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/AMS%20%E4%B8%8E%E5%BA%94%E7%94%A8%E8%BF%9B%E7%A8%8B%E9%80%9A%E4%BF%A1.png)

![ActivityThrad 启动 Activity 的过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ActivityThread%20%E5%90%AF%E5%8A%A8%20Activity%20%E7%9A%84%E8%BF%87%E7%A8%8B.png)

scheduleLaunchActivity方法将启动Activity的参数封装成ActivityClientRecord，sendMessage方法向H类发送类型为LAUNCH_ACTIVITY的消息，并将ActivityClientRecord传递过去。

查看H的handleMessage方法中对LAUNCH_ACTIVITY的处理，在注释1处将传过来的msg的成员变量obj转换为ActivityClientRecord。在注释2处通过getPackageInfoNoCheck方法获得LoadedApk类型的对象并赋值给ActivityClientRecord 的成员变量packageInfo。应用程序进程要启动Activity时需要将该Activity所属的APK加载进来，而LoadedApk就是用来描述已加载的APK文件的。

1. 获取 ActivityInfo，用于存储代码以及AndroidManifes设置的Activity和Receiver节点信息，比如Activity的theme和launchMode。
2. 获取APK文件的描述类LoadedAp
3. 获取要启动的Activity的ComponentName类，在ComponentName 类中保存了该Activity的包名和类名
4. 创建要启动Activity的上下文环境
5. 根据ComponentName中存储的Activity类名，用类加载器来创建该Activity的实例
6. 创建Application，makeApplication 方法内部会调用Application的onCreate方法
7. 用Activity的attach方法初始化Activity，在attach方法中会创建Window对象（PhoneWindow）并与Activity自身进行关联
8. Instrumentation的callActivityOnCreate方法来启动Activity。

![根Activity启动过程中涉及的进程之间的关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E6%A0%B9Activity%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E4%B8%AD%E6%B6%89%E5%8F%8A%E7%9A%84%E8%BF%9B%E7%A8%8B%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.png)

![根Activity启动过程中进程调用时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E6%A0%B9Activity%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E4%B8%AD%E8%BF%9B%E7%A8%8B%E8%B0%83%E7%94%A8%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

#### 4.2 Service 的启动过程

Activity 启动会创建上下文对象 appContentx，并传入Activity的attach方法中，将Activity与上下文对象appContext关联起来。

![ContextImpl到AMS的调用过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ContextImpl%E5%88%B0AMS%E7%9A%84%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B.png)

![ActivityThread启动Service的时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ActivityThread%E5%90%AF%E5%8A%A8Service%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

`handleCreateService`:

1. 获取要启动Service的应用程序的LoadedApk，LoadedApk是一个APK文件的描述类。
2. 通过调用LoadedApk的getClassLoader方法来获取类加载器
3. 根据CreateServiceData对象中存储的Service信息，创建Service实例
4. 创建Service的上下文环境ContextImpl对象
5. 通过Service的attach方法来初始化Service
6. 调用Service的onCreate方法，这样Service就启动了
7. 将启动的Service加入到ActivityThread的成员变量mServices中，其中mServices是ArrayMap类型

#### 4.3 Service 的绑定过程

![bindService ContextImpl到AMS的调用过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/bindService%20ContextImpl%E5%88%B0AMS%E7%9A%84%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B.png)

> - ServiceRecord：用于描述一个Service。
> - ProcessRecord：一个进程的信息。
> - ConnectionRecord：用于描述应用程序进程和Service建立的一次通信。
> - AppBindRecord：应用程序进程通过Intent绑定Service时，会通过AppBindRecord来维护Service与应用程序进程之间的关联。其内部存储了谁绑定的Service （ProcessRecord）、被绑定的Service （AppBindRecord）、绑定Service的Intent （IntentBindRecord）和所有绑定通信记录的信息（ArraySet＜ConnectionRecord＞）。
> - IntentBindRecord：用于描述绑定Service的Intent。

`handleBindService`:
1. 获取要绑定的Service。
2. BindServiceData的成员变量rebind的值为false，这样会调用Service的onBind方法，到这里Service处于绑定状态了。
3. 如果rebind的值为true就会调用Service的onRebind方法，这一点结合前文的bindServiceLocked方法的注释5处，得出的结论就是：如果当前应用程序进程第一个与Service进行绑定，并且Service已经调用过onUnBind方法，则会调用Service的onRebind方法。
4. 未绑定的情况，实际上是调用AMS的publishService方法。

![Service的绑定过程前半部分调用关系时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Service%E7%9A%84%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E5%89%8D%E5%8D%8A%E9%83%A8%E5%88%86%E8%B0%83%E7%94%A8%E5%85%B3%E7%B3%BB%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

调用了ServiceConnection 类型的对象mConnection 的onServiceConnected方法，这样在客户端实现了ServiceConnection接口类的onServiceConnected方法就会被执行。

![Service的绑定过程剩余部分的代码时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Service%E7%9A%84%E7%BB%91%E5%AE%9A%E8%BF%87%E7%A8%8B%E5%89%A9%E4%BD%99%E9%83%A8%E5%88%86%E7%9A%84%E4%BB%A3%E7%A0%81%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

#### 4.4 广播的注册、发送和接收过程

广播的注册分为两种，分别是静态注册和动态注册，**静态注册在应用安装时由PackageManagerService来完成注册过程**。

![广播的动态注册过程时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E5%B9%BF%E6%92%AD%E7%9A%84%E5%8A%A8%E6%80%81%E6%B3%A8%E5%86%8C%E8%BF%87%E7%A8%8B%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

IIntentReceiver是一个Binder接口，用于广播的跨进程的通信，它在LoadedApk.ReceiverDispatcher.InnerReceiver中实现。

最终会调用AMS的registerReceiver方法，并将IIntentReceiver类型的rd传进去，这里之所以不直接传入BroadcastReceiver而是传入IIntentReceiver，是因为注册广播是一个跨进程过程，需要具有跨进程的通信功能的IIntentReceiver。

```java
// ActivityManagerService.java #registerReceiver
//...
callerApp = getRecordForAppLocked(caller);// 1.用于描述请求 AMS 注册广播接收者的 Activity 所在的应用程序进程
//...
Iterator<String> actoins = filter.actioinsInterator();// 2.根据actions列表和userIds（userIds可以理解为应用程序的uid）得到所有的粘性广播的intent
//...
stickyIntents.addAll(intents);// 3.传入到stickyIntents中。接下来从stickyIntents中找到匹配传入的参数filter的粘性广播的intent

//...
allSticky.add(intent); //4. 将这些intent存入到allSticky列表中，从这里可以看出**粘性广播是存储在AMS中的**。
```

```java
// ActivityManagerService.java #registerReceiver
//...
ReceiverList rl = mRegisteredReceivers.get(reveiver.asBinder()); // 1.
//...
BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage, permission, callingUid, userId); // BroadcastFilter用来描述注册的广播接收者

//...
rl.add(bf); // 将自身添加到ReceiverList中

//...
mReceiverResolver.addFilter(bf); // 将BroadcastFilter添加到IntentResolver类型的mReceiverResolver中，这样当AMS接收到广播时就可以从mReceiverResolver中找到对应的广播接收者了，从而达到了注册广播的目的。
```

广播的发送和接收过程：

![ContextImpl到AMS的调用过程的时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ContextImpl%E5%88%B0AMS%E7%9A%84%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

AMS `verifyBroadcastLocked`验证广播是否合法。

AMS `broadcastIntentLocked`前面的工作主要是将动态注册的广播接收者和静态注册的广播接收者按照优先级高低不同存储在不同的列表中，再将这两个列表合并到receivers列表中，这样receivers列表包含了所有的广播接收者（无序广播和有序广播）。接着调用BroadcastQueue的scheduleBroadcastsLocked方法。 

![AMS到BroadcastReceiver的调用过程的时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/AMS%E5%88%B0BroadcastReceiver%E7%9A%84%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

```java
static final class ReceiverDispatcher {
	findl static class InnerReceiver extends IIntentReceiver.Stub {
		...
		@Override
		public void performReceive(Intent intent, ...) {

		}
	}
}
```

IIntentReceiver和IActivityManager 一样，都使用了AIDL来实现进程间通信。InnerReceiver继承自IIntentReceiver.Stub，是Binder通信的服务器端，IIntentReceiver则是Binder 通信的客户端、InnerReceiver 在本地的代理，它的具体的实现就是InnerReceiver。

#### ContentProvider 的启动过程

![query方法到AMS的调用过程的时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/query%E6%96%B9%E6%B3%95%E5%88%B0AMS%E7%9A%84%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

![AMS启动Content Provider的过程的时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/AMS%E5%90%AF%E5%8A%A8Content%20Provider%E7%9A%84%E8%BF%87%E7%A8%8B%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

### ch5 理解上下文 Context

#### 5.1 Context 的关联类

Activity、Service和Application都间接地继承自Context，因此我们可以计算出一个应用程序进程中有多少个Context，这个数量等于Activity和Service的总个数加1，1指的是Application的数量。

![Context 关联类](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Context%20%E5%85%B3%E8%81%94%E7%B1%BB.png)

ContextImpl 提供了很多功能，但是外界需要使用并拓展ContextImpl的功能，因此设计上使用了装饰模式，ContextWrapper是装饰类，它对ContextImpl进行包装，ContextWrapper主要是起了方法传递的作用，ContextWrapper中几乎所有的方法都是调用ContextImpl的相应方法来实现的。ContextThemeWrapper、Service和Application都继承自ContextWrapper，这样它们都可以通过mBase来使用Context的方法，同时它们也是装饰类，在ContextWrapper的基础上又添加了不同的功能。ContextThemeWrapper中包含和主题相关的方法（比如getTheme方法），因此，需要主题的Activity继承ContextThemeWrapper，而不需要主题的Service继承ContextWrapper。

#### 5.2 Application Context 的创建过程

![Application Context的创建过程的时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Application%20Context%E7%9A%84%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

```java
//LoadedApk.java
ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
appContext.setOuterContext(app);
mApplication = app; // Application 类型
```

这个base一路传递过来指的是ContextImpl，它是Context的实现类，将ContextImpl赋值给ContextWrapper的Context类型的成员变量mBase，这样在ContextWrapper中就可以使用Context的方法，而Application继承自ContextWrapper，同样可以使用Context的方法。**Application的attach方法的作用就是使Application可以使用Context的方法，这样Application才可以用来代表Application Context**。

#### 5.3 Application Context 的获取过程

```java
//ContextImpl.java
@Override
public Context getApplicationContext() {
	return (mPackageInfo != null) ? mPackageInfo.getApplication() : mMainThraed.getApplicatioin();
}
```

#### 5.4 Activity的Context创建过程

ActivityThread是应用程序进程的主线程管理类，它的内部类ApplicationThread会调用scheduleLaunchActivity方法来启动Activity

![Activity的Context创建过程的时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Activity%E7%9A%84Context%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

```java
//ActivityThread.java
private Activity performLaunchActivity(ActivityRecord r, Intent customIntent) {
	...
	ContextImpl appContext = createBaseContextForActivity(r); // 创建Activity的ContextImpl
	...
	activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
	...
	appContext.setOuterContext(activity);
	activity.attach(appContext, ...);

}
```

总结一下，在启动Activity的过程中创建ContextImpl，并赋值给ContextWrapper的成员变量mBase。Activity继承自ContextWrapper的子类ContextThemeWrapper，这样在Activity中就可以使用Context中定义的方法了。

#### 5.5 Service的Context创建过程

时序图可参考 Activity 的 Context 的创建过程

### ch6 理解 ActivityManagerService

#### 6.1 AMS 家族

![Android 7.0 AMS家族](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Android%207.0%20AMS%E5%AE%B6%E6%97%8F.png)

AMP就是AMS的代理类。其中IActivityManager是一个接口，AMN和AMP都实现了这个接口，用于实现代理模式和Binder通信。

AMP是AMN的内部类，它们都实现了IActivityManager接口，这样它们就可以实现代理模式，具体来讲是远程代理：AMP和AMN是运行在两个进程中的，AMP是Client端，AMN则是Server端，而Server端中具体的功能都是由AMN的子类AMS来实现的，因此，AMP就是AMS在Client端的代理类。AMN又实现了Binder类，这样AMP和AMS就可以通过Binder来进行进程间通信。ActivityManager通过AMN的getDefault方法得到AMP，通过AMP就可以和AMS进行通信。除ActivityManager以外，有些想要与AMS进行通信的类也需要通过AMP。

Android 8.0 AMS, 采用的是AIDL，IActivityManager.java类是由AIDL工具在编译时自动生成的。要实现进程间通信，服务器端也就是AMS只需要继承IActivityManager.Stub类并实现相应的方法就可以了。采用AIDL后就不需要使用AMS的代理类AMP了，因此Android 8.0去掉了AMP，代替它的是IActivityManager，它是AMS在本地的代理。

![Android 8.0 AMS家族](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Android%208.0%20AMS%E5%AE%B6%E6%97%8F.png)

#### 6.2 AMS 的启动过程

AMS的启动是在SystemServer进程中启动的。

SystemServer的run方法会首先加载了动态库libandroid_servers.so。接下来创建SystemServiceManager，它会对系统的服务进行创建、启动和生命周期管理。

分别是引导服务、核心服务和其他服务，其中其他服务是一些非紧要和不需要立即启动的服务。AMS 是在引导服务中启动的

mSystemServiceManager.startService（ActivityManagerService.Lifecycle.class）.getService（）实际得到的就是AMS实例。

#### 6.3 AMS 与应用程序进程

- 启动应用程序时AMS会检查这个应用程序需要的应用程序进程是否存在。
- 如果需要的应用程序进程不存在，AMS就会请求Zygote进程创建需要的应用程序进程。

#### 6.4 AMS 重要的数据结构
...

#### 6.5 Activity 栈管理

![Activity任务栈模型](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Activity%E4%BB%BB%E5%8A%A1%E6%A0%88%E6%A8%A1%E5%9E%8B.png)

ActivityRecord 用来记录一个Activity 的所有信息，TaskRecord 中包含了一个或多个ActivityRecord，TaskRecord用来表示Activity的任务栈，用来管理栈中的ActivityRecord，ActivityStack又包含了一个或多个TaskRecord，它是TaskRecord的管理者。

 singleInstance：和singleTask基本类似，不同的是启动Activity时，首先要创建一个新栈，然后创建该Activity实例并压入新栈中，新栈中只会存在这一个Activity实例。

taskAffinity与FLAG_ACTIVITY_NEW_TASK或者singleTask配合。如果新启动Activity的taskAffinity和栈的taskAffinity相同则加入到该栈中；如果不同，就会创建新栈。

taskAffinity与allowTaskReparenting配合。如果allowTaskReparenting为true，说明Activity 具有转移的能力。

### ch7 理解 WindowManager

![Window、WindowManager和WMS的关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Window%E3%80%81WindowManager%E5%92%8CWMS%E7%9A%84%E5%85%B3%E7%B3%BB.png)

![WindowManager的关联类](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/WindowManager%E7%9A%84%E5%85%B3%E8%81%94%E7%B1%BB.png)

Context的getSystemService方法得到的是WindowManagerImpl实例；WindowManagerImpl虽然是WindowManager的实现类，但是没有实现什么功能，而是将功能实现委托给了WindowManagerGlobal，这里用到的是桥接模式。

PhoneWindow继承自Window，Window通过setWindowManager方法与WindowManager发生关联。WindowManager 继承自接口ViewManager，WindowManagerImpl是WindowManager 接口的实现类，但是具体的功能都会委托给WindowManagerGlobal来实现。

Window 的属性，它们分别是Type（Window的类型）、Flag（Window的标志）和SoftInputMode（软键盘相关模式）。

![Window的操作](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Window%E7%9A%84%E6%93%8D%E4%BD%9C.png)

WindowManagerGlobal中维护的和Window操作相关的3个列表，在窗口的添加、更新和删除过程中都会涉及这3个列表，它们分别是View 列表（ArrayList＜View＞ mViews）、布局参数列表（ArrayList＜WindowManager.LayoutParams＞mParams）和ViewRootImpl列表（ArrayList＜ViewRootImpl＞mRoots）

`addView`:
1. 参数检查
2. 若为子窗口会根据父窗口对子窗口的WindowManager.LayoutParams 类型的wparams 对象进行相应调整
3. 将添加的View保存到View列表中
4. 将窗口的参数保存到布局参数列表中
5. 创建了ViewRootImp并赋值给root，紧接着将root存入到ViewRootImpl列表中
6. 将窗口和窗口的参数通过setView方法设置到ViewRootImpl中

ViewRootImpl身负了很多职责，主要有以下几点：· View树的根并管理View树。· 触发View的测量、布局和绘制。· 输入事件的中转站。· 管理Surface。· 负责与WMS进行进程间通信。

![ViewRootImpl与WMS通信](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ViewRootImpl%E4%B8%8EWMS%E9%80%9A%E4%BF%A1.png)

在addToDisplay方法中调用了WMS的addWindow方法，并将自身也就是Session作为参数传了进去，每个应用程序进程都会对应一个Session，WMS会用ArrayList来保存这些Session，这就是为什么WMS包含Session的原因。这样剩下的工作就交给WMS来处理，**在WMS中会为这个添加的窗口分配Surface，并确定窗口显示次序，可见负责显示界面的是画布Surface，而不是窗口本身。WMS 会将它所管理的Surface 交由SurfaceFlinger处理，SurfaceFlinger会将这些Surface混合并绘制到屏幕上**。

![系统窗口StatusBar的添加过程的时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%B3%BB%E7%BB%9F%E7%AA%97%E5%8F%A3StatusBar%E7%9A%84%E6%B7%BB%E5%8A%A0%E8%BF%87%E7%A8%8B%E7%9A%84%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

Activity 的添加过程：
无论是哪种窗口，它的添加过程在WMS 处理部分中基本是类似的，只不过会在权限和窗口显示次序等方面会有些不同。

Window 的更新过程：
`ViewManager#updateViewLayout` ->...-> `ViewRootImpl#scheduleTraversals`

```java
//ViewRootImpl.java
void scheduleTraversals() {
	...
	mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
}
```
Choreographer译为“舞蹈指导”，用于接收显示系统的VSync信号，在下一个帧渲染时控制执行一些操作。Choreographer的postCallback方法用于发起添加回调，这个添加的回调将在下一帧被渲染时执行。

-> `doTraversal` -> `performTraversals` ViewTree 开始 View 的工作流程。

### ch8 理解 WindowManagerService

#### 8.1 WMS 的职责

1. 窗口管理：窗口管理的核心成员有DisplayContent、WindowToken和WindowState
2. 窗口动画：动画子系统的管理者为WindowAnimator
3. 输入系统的中转站：InputManagerService（IMS）会对触摸事件进行处理，它会寻找一个最合适的窗口来处理触摸反馈信息
4. Surface管理：窗口并不具备绘制的功能，因此每个窗口都需要有一块Surface来供自己绘制

![WMS 职责](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/WMS%20%E8%81%8C%E8%B4%A3.png)

#### 8.2 WMS 的创建过程

WMS是在SystemServer进程中创建的。

官方把系统服务分为了三种类型，分别是引导服务、核心服务和其他服务，其中其他服务是一些非紧要和不需要立即启动的服务，**WMS就是其他服务的一种**。

初始化，Watchdog 用来监控系统一些关键服务的运行状况。

WMS的main 方法是运行在SystemServer的run方法中的，换句话说就是运行在“system_server”线程中。

```java
//WindowManagerService.java
public static WindowManagerService main(final Context context, InputManagerService im, final boolean haveInputMethods, final boolean showBootMsgs, final boolean onlyCore, WindowManagerPolicy policy) {
	DisplayThread.getHandler().runWithScissots(() -> // 切换线程，WMS 的创建是运行在 android.display 线程中的
		sInstance = new WindowManagerService(context, im, haveInputMethods, showBootMsgs, onlyCore, policy), 0);
	return sInstance;
}
```

DisplayThread是一个单例的前台线程，这个线程用来处理需要低延时显示的相关操作，并只能由WindowManager、DisplayManager和InputManager实时执行快速操作。

内部会根据每个线程只有一个Looper 的原理来判断当前的线程（system_server线程）是否是Handler所指向的线程（android.display线程），如果是则直接执行Runnable的run方法，如果不是则调用BlockingRunnable的postAndWait方法，并将当前线程的Runnable作为参数传进去。

system_server线程等待的就是android.display线程，一直到android.display线程执行完毕再执行system_server线程，这是因为android.display线程内部执行了WMS的创建，而WMS的创建优先级要更高。

PWM的init方法运行在android.ui线程中，它的优先级要高于initPolicy方法所在的android.display线程，因此android.display 线程要等PWM的init方法执行完毕后，处于等待状态的android.display线程才会被唤醒从而继续执行下面的代码。

![三个线程之间的关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E4%B8%89%E4%B8%AA%E7%BA%BF%E7%A8%8B%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.png)

1. 首先在system_server 线程中执行了SystemServer的startOtherServices 方法，在startOtherServices方法中会调用WMS的main方法，main方法会创建WMS，创建的过程在android.display线程中实现，创建WMS的优先级更高，因此system_server线程要等WMS创建完成后，处于等待状态的system_server线程才会被唤醒从而继续执行下面的代码。
2. 在WMS的构造方法中会调用WMS的initPolicy方法，在initPolicy方法中又会调用PWM 的init 方法，PWM的init方法在android.ui线程中运行，它的优先级要高于android.display线程，因此“android.display”线程要等PWM的init方法执行完毕后，处于等待状态的android.display线程才会被唤醒从而继续执行下面的代码。
3. PWM的init方法执行完毕后，android.display线程就完成了WMS的创建，等待的system_server线程被唤醒后继续执行WMS的main 方法后的代码逻辑，比如WMS的displayReady方法用来初始化屏幕显示信息。

#### 8.3 WMS 的重要成员

#### 8.4 Window 的添加过程（WMS 处理部分）

`WMS#addWindow`：

- part1：根据 Window 属性，检查权限；通过displayId 来获得窗口要添加到哪个DisplayContent 上；根据attrs.token 作为key 值从mWindowMap中得到该子窗口的父窗口。接着对父窗口进行判断。返回的是`addWindow`的各种状态。

- part2：通过displayContent的getWindowToken方法得到WindowToken；rootType 赋值（父窗口或自身 type）；隐式创建 WindowToken；转化为专门针对应用程序窗口的 AppWindowToken，后续会根据其值进行判断。

- part3：创建了WindowState（存有窗口的所有的状态信息，在WMS中它代表一个窗口）；判断请求添加窗口的客户端是否已经死亡、窗口的DisplayContent是否为null；根据窗口的type对窗口的LayoutParams的一些成员变量进行修改；WindowState 添加到对应数据结构及 WindowToken 中。

**总结**：
addWindow方法分了3个部分来进行讲解，主要就是做了下面4件事：

1. 对所要添加的窗口进行检查，如果窗口不满足一些条件，就不会再执行下面的代码逻辑。
2. WindowToken相关的处理，比如有的窗口类型需要提供WindowToken，没有提供的话就不会执行下面的代码逻辑，有的窗口类型则需要由WMS 隐式创建WindowToken。
3. WindowState的创建和相关处理，将WindowToken和WindowState相关联。
4. 创建和配置DisplayContent，完成窗口添加到系统前的准备工作。

#### 8.5 Window 的删除过程

`WindowManagerGlobal#removeView` -> `WindowManagerGlobal#removeViewLocked`：InputMethodManager 实例不为null处理结束 View 的输入法相关逻辑 -> `ViewRootImpl#die` -> `ViewRootImpl#doDie`：判断执行线程的正确性（需为创建 View 的原始线程）、防止重复调用、`dispatchDetachedFromWindow`方法来销毁View > 如果V有子View并且不是第一次被添加 -> `WindowManagerGlobal#doRemoveView`：doRemoveView方法会从这三个列表中清除V对应的元素。在注释1处找到V对应的ViewRootImpl在ViewRootImpl列表中的索引，接着根据这个索引从ViewRootImpl列表、布局参数列表和View列表中删除与V对应的元素。

`dispatchDetachedFromWindow`方法来销毁View -> `IWindowSession#remove` -> 跨进程 -> `Session#remove` -> `WindowManagerService#removeWindow`：获取Window对应的WindowState，WindowState用于保存窗口的信息，在WMS中它用来描述一个窗口。 -> `WindowState#removeIfPossible`：是否需要延迟删除操作的条件判断过滤 -> `WindowState#removeImmediately`：是否正在删除 Window 操作，避免重复删除；删除对应控制器，将V对应的Session从WMS的ArraySet＜Session＞ mSessions 中删除并清除Session 对应的SurfaceSession 资源（SurfaceSession是SurfaceFlinger的一个连接，通过这个连接可以创建1个或者多个Surface并渲染到屏幕上），接着调用了WMS的postWindowRemoveCleanupLocked方法用于对V进行一些集中的清理工作。

**总结**：
1. 检查删除线程的正确性，如果不正确就抛出异常。
2. 从ViewRootImpl列表、布局参数列表和View列表中删除与V对应的元素。
3. 判断是否可以直接执行删除操作，如果不能就推迟删除操作。
4. 执行删除操作，清理和释放与V相关的一切资源。


### ch10 Java 虚拟机

#### 10.1 概述

Java 虚拟机家族：HotSpot VM、J9 VM、Zing VM

Java 虚拟机执行流程：

![Java 虚拟机执行流程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Java%20%E8%99%9A%E6%8B%9F%E6%9C%BA%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png)

#### 10.2 Java 虚拟机结构

![Java虚拟机结构](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%BB%93%E6%9E%84.png)

Java虚拟机结构包括运行时数据区域、执行引擎、本地库接口和本地方法库，其中类加载子系统并不属于Java虚拟机的内部结构。

![类的生命周期](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%B1%BB%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

1. 加载：查找并加载Class文件。

2. 链接：包括验证、准备和解析。
- 验证：确保被导入类型的正确性。
- 准备：为类的静态字段分配字段，并用默认值初始化这些字段.
- 解析：虚拟机将常量池内的符号引用替换为直接引用。

3. 初始化：将类变量初始化为正确初始值

系统加载器：Bootstrap ClassLoader（引导类加载器）、Extensions ClassLoader（拓展类加载器）、Application ClassLoader（应用程序类加载器）

运行时数据区域：
这些数据区域分别为程序计数器、Java虚拟机栈、本地方法栈、Java堆和方法区。

1. 程序计数器：程序计数器是**线程私有**的。如果线程执行的方法不是Native方法，则程序计数器保存正在执行的字节码指令地址，如果是Native 方法则程序计数器的值为空（Undefined）。程序计数器是Java虚拟机规范中**唯一没有规定任何OutOfMemoryError情况的数据区域**。

2. Java 虚拟机栈：**每一条Java 虚拟机线程都有一个线程私有的Java 虚拟机栈**（Java Virtual Machine Stacks）。它的生命周期与线程相同，与线程是同时创建的。存储的是线程中Java方法调用的状态，包括局部变量、参数、返回值以及运算的中间结果等。一个Java虚拟机栈包含了多个栈帧，一个栈帧用来存储局部变量表、操作数栈、动态链接、方法出口等信息。

- 如果线程请求分配的栈容量超过Java虚拟机所允许的最大容量，Java虚拟机会抛出StackOverflowError。
- 如果Java虚拟机栈可以动态扩展（大部分Java虚拟机都可以动态扩展），但是扩展时无法申请到足够的内存，或者在创建新的线程时没有足够的内存去创建对应的Java虚拟机栈，则会抛出OutOfMemoryError异常。

3. 本地方法栈：与Java虚拟机栈类似，本地方法栈也会抛出StackOverflowError和OutOfMemoryError异常。

4. Java 堆：Java堆（Java Heap）是被**所有线程**共享的运行时内存区域。Java堆用来存放对象实例，几乎所有的对象实例都在这里分配内存。如果在堆中没有足够的内存来完成实例分配，并且堆也无法进行扩展时，则会抛出OutOfMemoryError异常。

5. 方法区：方法区（Method Area）是被所有线程共享的运行时内存区域，用来存储已经被Java虚拟机加载的类的结构信息，包括运行时常量池、字段和方法信息、静态变量等数据。**方法区是 Java 堆的逻辑组成部分**。如果方法区的内存空间不满足内存分配需求时，Java虚拟机会抛出OutOfMemoryError异常。

6. 运行时常量池：运行时常量池（Runtime Constant Pool）**并不是运行时数据区域的其中一份子，而是方法区的一部分**。当创建类或接口时，如果构造运行时常量池所需的内存超过了方法区所能提供的最大值，Java虚拟机会抛出OutOfMemoryError异常。

#### 10.6 垃圾标记算法

Java 中的引用：

- 强引用
- 软引用：如果一个对象只具有软引用，当内存不够时，会回收这些对象的内存
- 弱引用：比起软引用具有更短的生命周期，垃圾收集器一旦发现了只具有弱引用的对象，不管当前内存是否足够，都会回收它的内存。
- 虚引用：一个只具有虚引用的对象，被垃圾收集器回收时会收到一个系统通知，这也是虚引用的主要作用。

引用计数算法：循环引用问题

根搜索算法：

可以作为GC Roots的对象主要有以下几种：

- Java栈中引用的对象。
- 本地方法栈中JNI引用的对象。
- 方法区中运行时常量池引用的对象。
- 方法区中静态属性引用的对象。
- 运行中的线程。
- 由引导类加载器加载的对象。
- GC控制的对象。

Java 对象在虚拟机中的生命周期：

1. 创建阶段（Created）
2. 应用阶段（In Use）
3. 不可见阶段（Invisible）
4. 不可达阶段（Unreachable）
5. 收集阶段（Collected）
6. 终结阶段（Finalized）
7. 对象空间重新分配阶段（Deallocated）

垃圾回收算法：

- 标记—清除算法
- 复制算法
- 标记—压缩算法
- 分代收集算法

### ch11 Dalvik 和 ART

DVM 与 JVM 的区别

1. 基于的架构不同：DVM是基于寄存器的，它没有基于栈的虚拟机在复制数据时而使用的大量的出入栈指令，同时指令更紧凑、更简洁。但是由于显式指定了操作数，所以基于寄存器的指令会比基于栈的指令要大，但是由于指令数量的减少，总的代码数不会增加多少。
2. 执行的字节码不同：DVM会用dx工具将所有的.class文件转换为一个.dex文件，然后DVM会从该.dex文件读取指令和数据。执行顺序为.java文件→.class文件→.dex文件。
3. DVM允许在有限的内存中同时运行多个进程
4. DVM允许在有限的内存中同时运行多个进程：是一个DVM进程，同时也用来创建和初始化DVM实例。每当系统需要创建一个应用程序时，Zygote就会fock自身，快速地创建和初始化一个DVM实例，用于应用程序的运行。
5. DVM有共享机制：DVM拥有预加载—共享的机制，不同应用之间在运行时可以共享相同的类，拥有更高的效率。
6. DVM早期没有使用JIT编译器：从Android 2.2版本开始DVM使用了JIT 编译器，它会对多次运行的代码（热点代码）进行编译，生成相当精简的本地机器码（Native Code），这样在下次执行到相同逻辑的时候，直接使用编译之后的本地机器码，而不是每次都需要编译

![DVM 架构](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/DVM%20%E6%9E%B6%E6%9E%84.png)

DVM 的运行时堆：DVM的运行时堆使用标记—清除（Mark-Sweep）算法进行GC，它由两个Space以及多个辅助数据结构组成，两个Space分别是Zygote Space（Zygote Heap）和Allocation Space （Active Heap）。

ART 与 DVM 的区别：

1. 而在ART中，系统在安装应用程序时会进行一次AOT（ahead of time compilation，预编译），将字节码预先编译成机器码并存储在本地，这样应用程序每次运行时就不需要执行编译了，运行效率会大大提升，设备的耗电量也会降低。采用AOT也会有缺点，主要有两个：第一个是AOT会使得应用程序的安装时间变长，尤其是一些复杂的应用；第二个是字节码预先编译成机器码，机器码需要的存储空间会多一些。Android 7.0 版本中的 ART 加入了即时编译 JIT 作为补充解决上述缺点。
2. DVM是为32位CPU设计的，而ART支持64位并兼容32位CPU，这也是DVM被淘汰的主要原因之一
3. ART对垃圾回收机制进行了改进，比如更频繁地执行并行垃圾收集，将GC暂停由2次减少为1次等。
4. ART的运行时堆空间划分和DVM不同

**DVM和ART是在Zygote进程中诞生的，这样Zygote进程就持有了DVM或者ART的实例，此后Zygote进程fork自身创建应用程序进程时，应用程序进程也得到了DVM或者ART的实例，这样就不需要每次启动应用程序进程都要创建DVM或者ART，从而加快了应用程序进程的启动速度**。

### ch12 理解 ClassLoader

Java中的类加载器主要有两种类型，即系统类加载器和自定义类加载器。其中系统类加载器包括3种，分别是Bootstrap ClassLoader、Extensions ClassLoader和Application ClassLoader。

![ClassLoader的继承关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ClassLoader%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)

双亲委托模式：所谓双亲委托模式就是首先判断该Class是否已经加载，如果没有则不是自身去查找而是委托给父加载器进行查找，这样依次进行递归，直到委托到最顶层的Bootstrap ClassLoader，如果Bootstrap ClassLoader找到了该Class，就会直接返回，如果没找到，则继续依次向下查找，如果还没找到则最后会交由自身去查找。

双亲委托模式好处：

- 避免重复加载
- 更加安全：如果不使用双亲委托模式，就可以自定义一个String 类来替代系统的String类，这显然会造成安全隐患，采用双亲委托模式会使得系统的String类在Java虚拟机启动时就被加载，也就无法自定义String类来替代系统的String类，除非我们修改类加载器搜索类的默认算法。还有一点，只有两个类名一致并且被同一个类加载器加载的类，Java虚拟机才会认为它们是同一个类，想要骗过Java虚拟机显然不会那么容易

Android 中的 ClassLoader：加载的不再是Class文件，而是dex 文件，这就需要重新设计ClassLoader 相关类。也分为两种类型，分别是系统类加载器和自定义加载器。其中系统类加载器主要包括3种，分别是BootClassLoader、PathClassLoader和DexClassLoader。

![Android8.0中ClassLoader的继承关系](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Android8.0%E4%B8%ADClassLoader%E7%9A%84%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)

![ClassLoader查找流程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/ClassLoader%E6%9F%A5%E6%89%BE%E6%B5%81%E7%A8%8B.png)

Android中没有ExtClassLoader 和AppClassLoader，替代它们的是PathClassLoader和DexClassLoader。

### ch13 热修复原理

代码修复、资源修复和动态链接库修复。

Instant Run中的资源热修复可以简单地总结为两个步骤：

1. 创建新的AssetManager，通过反射调用addAssetPath方法加载外部的资源，这样新创建的AssetManager就含有了外部资源。
2. 将AssetManager类型的mAssets字段的引用全部替换为新创建的AssetManager。

代码修复方案：底层替换方案、类加载方案和Instant Run方案。

类加载方案：根据类的查找流程，我们将有Bug的类Key.class进行修改，再将Key.class打包成包含dex的补丁包Patch.jar，放在Element数组dexElements的第一个元素，这样会首先找到Patch.dex中的Key.class去替换之前存在Bug的Key.class，排在数组后面的dex文件中存在Bug的Key.class根据ClassLoader的双亲委托模式就不会被加载，这就是类加载方案

![类加载方案](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%96%B9%E6%A1%88.png)

类加载方案需要重启App 后让ClassLoader 重新加载新的类，为什么需要重启呢？这是因为类是无法被卸载的，要想重新加载新的类就需要重启App，因此采用类加载方案的热修复框架是不能即时生效的。

底层替换方案：底层替换方案不会再次加载新类，而是直接在Native层修改原有类，由于在原有类进行修改限制会比较多，且不能增减原有类的方法和字段，如果我们增加了方法数，那么方法索引数也会增加，这样访问方法时会无法通过索引找到正确的方法，同样的字段也是类似的情况。

Instant Run 方案：ASM

动态链接库的修复：热修复框架的so的修复的主要是更新so，换句话说就是重新加载so，因此so的修复的基础原理就是加载so：

1. 将so补丁插入到NativeLibraryElement数组的前部，让so补丁的路径先被返回和加载。
2. 调用System的load方法来接管so的加载入口。

### ch15 Activity 插件化

Activity插件化主要有3种实现方式，分别是反射实现、接口实现和Hook技术实现。

![根Activity启动过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E6%A0%B9Activity%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.png)

![普通Activity启动过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E6%99%AE%E9%80%9AActivity%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.png)

Hook IActivityManager 方案实现：具体做法就是先使用一个在AndroidManifest.xml中注册的Activity来进行占坑，用来通过AMS的校验。接着之后用插件Activity替换占坑的Activity。

主要的方案就是先用一个在AndroidManifest.xml中注册的Activity来进行占坑，用来通过AMS的校验，接着在合适的时机用插件Activity替换占坑的Activity。为了更好地讲解启动插件Activity的原理，本节省略了插件Activity的加载逻辑，直接创建一个TargetActivity来代表已经加载进来的插件Activity。同时这一节使我们更好地理解了Activity的启动过程。

### ch16 绘制优化

绘制原理：要想画面保持在60fps，需要屏幕在1秒内刷新60次，也就是每16.6667ms刷新一次（绘制时长在16ms以内）。

卡顿原因：

- 布局Layout过于复杂，无法在16ms内完成渲染。
- 同一时间动画执行的次数过多，导致CPU或GPU负载过重。
- View过度绘制，导致某些像素在同一帧时间内被绘制多次。
- 在UI线程中做了稍微耗时的操作。
- GC回收时暂停时间过长或者频繁的GC产生大量的暂停时间。

布局优化：

工具：Hierarchy Viewer、Android Lint

优化方法：合理运用布局、Include、Merge和ViewStub

### ch17 内存优化

内存泄漏产生的原因，主要分为三大类：

- 由开发人员自己编码造成的泄漏。
- 第三方框架造成的泄漏。
- 由Android系统或者第三方ROM造成的泄漏。

内存泄漏的场景：

- 非静态内部类的静态实例
- 多线程相关的匿名内部类/非静态内部类
- Handler内存泄漏：解决方案有两个，一个是使用一个静态的Handler内部类，Handler持有的对象要使用弱引用；还有一个解决方案是在Activity的Destroy方法中移除MessageQueue中的消息。
- 未正确使用Context
- 静态View
- WebView
- 资源对象未关闭
- 集合中对象没清理
- Bitmap对象
- 监听器未关闭

内存抖动：一般指在很短的时间内发生了多次内存分配和释放，严重的内存抖动还会导致应用程序卡顿。内存抖动出现的原因主要是短时间频繁地创建对象。

内存分析工具 MAT：

- Shallow Heap：对象自身占用的内存大小，不包括它引用的对象
- Retained Heap：一个对象的Retained Set包含对象所占内存的总大小。换句话说，Retained Heap就是当前对象被GC后，从Heap上总共能释放掉的内存。

LeakCanary：

[LeakCanary 原理](https://github.com/huanggenghg/huanggenghg/blob/main/LeakCanary%20%E5%8E%9F%E7%90%86.md)


























