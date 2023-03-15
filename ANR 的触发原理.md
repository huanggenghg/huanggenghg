## ANR 的 触发原理

### 一、概述

触发场景：

- Service Timeout：比如台前服务在 20s 内未执行完成；
- BroadcastQueue Timeout：比如前台广播在 10s 内未执行完成；
- ContentProvider Timeout：内容提供者，在 publish 过超时 10s；
- InputDispatching Timeout：输入事件分发超时 5s，包括按键和触摸事件。

### 二、Service

Service Timeout 是位于 ActivityManager 线程中的`AMS.MainHandler`收到`SERVICE_TIMEOUT_MSG`消息时触发。

- 前台服务，SERVICE_TIMEOUT = 20s；
- 后台服务，SERVICE_BACKGROUND_TIMEOUT = 200s

#### 2.1 埋炸弹

Service 进程 attach 到 system_server 进程的过程中会调用`realStartServiceLocked()`埋下（发送 delay 消息 SERVICE_TIMEOUT_MSG）。

#### 2.2 拆炸弹

目标进程的主线程的`handleCrateService`过程，最终调用到 system_server 执行`serviceDoneExecuting`移除超时消息`SERVICE_TIMEOUT_MSG`。

#### 2.3 引爆炸弹

在 system_server 进程中有一个 Handler 线程，名为 ActivityManager，当倒计时结束便会向该 Handler 线程发送一条信息`SERVICE_TIMEOUT_MSG`。当存在 timeout 的 Service，执行`appNotResonding`。

### 三、BroadcastReceiver

Broadcast Timeout 是位于 ActivityManager 线程中的`BroadcastQueue.BroadcastHandler`收到`BROADCAST_TIMEOUT_MSG`消息时触发。

- 前台广播，BROADCAST_FG_TIMEOUT = 10s；
- 后台广播，BROADCAST_BG_TIMEOUT = 60s

#### 3.1 埋炸弹

`processNextBroadcast`中设置超时广播消息。

#### 3.2 拆炸弹

广播设置为结束`sendFinished`，（静态注册广播）需考虑 SP 有未同步到磁盘的工作，需等待其完成才能告知系统已完成该广播。

#### 3.3 引爆炸弹

1. mOrderedBroadcasts已处理完成，则不会anr;
2. 正在执行dexopt，则不会anr;
3. 系统还没有进入ready状态(mProcessesReady=false)，则不会anr;
4. 如果当前正在执行的receiver没有超时，则重新设置广播超时，不会anr;
5. `mHandler.post(new AppNotResponding(app, anrMessage))`

### 四、ContentProvider

ContentProvider Timeout 是位于 ActivityManager 线程中的AMS.MainHandler`收到`CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG`消息时触发。

ContentProvider 超时为CONTENT_PROVIDER_PUBLISH_TIMEOUT = 10s. 这个跟前面的Service和BroadcastQueue完全不同, 由Provider[进程启动](http://gityuan.com/2016/10/09/app-process-create-2/)过程相关.

#### 4.1 埋炸弹

埋炸弹的过程其实是在进程创建的过程，进程创建后会调用`attachApplicationLocked()`进入system_server 进程

```java
//app进程存在正在启动中的provider,则超时10s后发送CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG消息
    if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
        Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
        msg.obj = app;
        mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
    }
```



#### 4.2 拆炸弹

当 provider 成功 publish 之后（`AMS.publishContentProviders`）,便会拆除该炸弹。

```java
//成功pubish则移除该消息
               if (wasInLaunchingProviders) {
                   mHandler.removeMessages(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG, r);
               }
```



#### 4.3 引爆炸弹

在system_server进程中有一个Handler线程, 名叫”ActivityManager”.当倒计时结束便会向该Handler线程发送 一条信息`CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG`。

### 五、appNotResponding 处理流程

```java
final void appNotResponding(ProcessRecord app, ActivityRecord activity, ActivityRecord parent, boolean aboveSystem, final String annotation) {
    ...
    updateCpuStatsNow(); //第一次 更新cpu统计信息
    synchronized (this) {
      //PowerManager.reboot() 会阻塞很长时间，因此忽略关机时的ANR
      if (mShuttingDown) {
          return;
      } else if (app.notResponding) {
          return;
      } else if (app.crashing) {
          return;
      }
      //记录ANR到EventLog
      EventLog.writeEvent(EventLogTags.AM_ANR, app.userId, app.pid,
              app.processName, app.info.flags, annotation);
              
      // 将当前进程添加到firstPids
      firstPids.add(app.pid);
      int parentPid = app.pid;
      
      //将system_server进程添加到firstPids
      if (MY_PID != app.pid && MY_PID != parentPid) firstPids.add(MY_PID);
      
      for (int i = mLruProcesses.size() - 1; i >= 0; i--) {
          ProcessRecord r = mLruProcesses.get(i);
          if (r != null && r.thread != null) {
              int pid = r.pid;
              if (pid > 0 && pid != app.pid && pid != parentPid && pid != MY_PID) {
                  if (r.persistent) {
                      firstPids.add(pid); //将persistent进程添加到firstPids
                  } else {
                      lastPids.put(pid, Boolean.TRUE); //其他进程添加到lastPids
                  }
              }
          }
      }
    }
    
    // 记录ANR输出到main log
    StringBuilder info = new StringBuilder();
    info.setLength(0);
    info.append("ANR in ").append(app.processName);
    if (activity != null && activity.shortComponentName != null) {
        info.append(" (").append(activity.shortComponentName).append(")");
    }
    info.append("\n");
    info.append("PID: ").append(app.pid).append("\n");
    if (annotation != null) {
        info.append("Reason: ").append(annotation).append("\n");
    }
    if (parent != null && parent != activity) {
        info.append("Parent: ").append(parent.shortComponentName).append("\n");
    }
    
    //创建CPU tracker对象
    final ProcessCpuTracker processCpuTracker = new ProcessCpuTracker(true);
    //输出traces信息【见小节2】
    File tracesFile = dumpStackTraces(true, firstPids, processCpuTracker, 
            lastPids, NATIVE_STACKS_OF_INTEREST);
            
    updateCpuStatsNow(); //第二次更新cpu统计信息
    //记录当前各个进程的CPU使用情况
    synchronized (mProcessCpuTracker) {
        cpuInfo = mProcessCpuTracker.printCurrentState(anrTime);
    }
    //记录当前CPU负载情况
    info.append(processCpuTracker.printCurrentLoad());
    info.append(cpuInfo);
    //记录从anr时间开始的Cpu使用情况
    info.append(processCpuTracker.printCurrentState(anrTime));
    //输出当前ANR的reason，以及CPU使用率、负载信息
    Slog.e(TAG, info.toString()); 
    
    //将traces文件 和 CPU使用率信息保存到dropbox，即data/system/dropbox目录
    addErrorToDropBox("anr", app, app.processName, activity, parent, annotation,
            cpuInfo, tracesFile, null);

    synchronized (this) {
        ...
        //后台ANR的情况, 则直接杀掉
        if (!showBackground && !app.isInterestingToUserLocked() && app.pid != MY_PID) {
            app.kill("bg anr", true);
            return;
        }

        //设置app的ANR状态，病查询错误报告receiver
        makeAppNotRespondingLocked(app,
                activity != null ? activity.shortComponentName : null,
                annotation != null ? "ANR " + annotation : "ANR",
                info.toString());

        //重命名trace文件
        String tracesPath = SystemProperties.get("dalvik.vm.stack-trace-file", null);
        if (tracesPath != null && tracesPath.length() != 0) {
            //traceRenameFile = "/data/anr/traces.txt"
            File traceRenameFile = new File(tracesPath);
            String newTracesPath;
            int lpos = tracesPath.lastIndexOf (".");
            if (-1 != lpos)
                // 新的traces文件= /data/anr/traces_进程名_当前日期.txt
                newTracesPath = tracesPath.substring (0, lpos) + "_" + app.processName + "_" + mTraceDateFormat.format(new Date()) + tracesPath.substring (lpos);
            else
                newTracesPath = tracesPath + "_" + app.processName;

            traceRenameFile.renameTo(new File(newTracesPath));
        }
                
        //弹出ANR对话框
        Message msg = Message.obtain();
        HashMap<String, Object> map = new HashMap<String, Object>();
        msg.what = SHOW_NOT_RESPONDING_MSG;
        msg.obj = map;
        msg.arg1 = aboveSystem ? 1 : 0;
        map.put("app", app);
        if (activity != null) {
            map.put("activity", activity);
        }
        
        //向ui线程发送，内容为SHOW_NOT_RESPONDING_MSG的消息
        mUiHandler.sendMessage(msg);
    }
    
}
```

当发生ANR时, 会按顺序依次执行:

1. 输出ANR Reason信息到EventLog. 也就是说ANR触发的时间点最接近的就是EventLog中输出的am_anr信息;
2. 收集并输出重要进程列表中的各个线程的**traces信息**，该方法较耗时; 
3. 输出当前各个进程的**CPU使用情况**以及CPU负载情况;
4. 将traces文件和 CPU使用情况信息**保存到dropbox**，即data/system/dropbox目录
5. 根据进程类型,来决定直接后台杀掉,还是弹框告知用户.

### 六、其他

*以上是[理解Android ANR的触发原理](http://gityuan.com/2016/07/02/android-anr/)、[理解Android ANR的信息收集过程](http://gityuan.com/2016/12/02/app-not-response/)的摘录。*