## 输入系统和 IMS

### 一、输入事件传递流程

输入设备可用时，Linux 内核会在 /dev/input 中创建对应的设备节点。

输入事件产生的原始信息会被 Linux 内核中的输入子系统采集，原始信息由 Kernel space 的驱动层一直传递到 User space 的设备节点。

![输入系统事件传递流程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E8%BE%93%E5%85%A5%E7%B3%BB%E7%BB%9F%E4%BA%8B%E4%BB%B6%E4%BC%A0%E9%80%92%E6%B5%81%E7%A8%8B.drawio.png)

![输入系统部分](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E8%BE%93%E5%85%A5%E7%B3%BB%E7%BB%9F%E9%83%A8%E5%88%86.drawio.png)

![WMS 的职责](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/WMS%20%E7%9A%84%E8%81%8C%E8%B4%A3.drawio.png)

### 二、IMS 的创建

与 AMS、WMS、PMS 一样，IMS 是在 SystemServer 进程中创建的，SystemServer 进程用来创建系统服务。IMS 是属于其他服务。

```java
startOtherServices();
```

```java
private vooid startOtherServices() {
    ...
        inputManager = new InputManagerService(context);
    ...
        wm = WindowManagerService.main(context, inputManager, ...); // WMS 是输入系统的中转站，内部包含 IMS 引用
    ServiceManager.addService(Context.INPUT_SERVICE, inputManager);
    ...
}
```

IMS 的构造方法中，创建了 InputManagerHandler，InputManagerHandler 运行在 android.display 线程中，android.display 线程是系统共享的单例前台线程，这个线程内部执行了 WMS 的创建。

在构造方法中，通过`nativeInit`创建了 NativeInputManager，在其中创建了 EventHub 和 InputManager，EventHub 通过 Linux内核的 INotify 与 Epoll 机制监听设备节点，通过 EventHub 的 getEvent 函数读取设备节点的增删事件和原始输入事件。

在 InputManager 构造函数中创建了 InputReader 和 InputDispatcher，InputReader 会不断循环读取 EventHub 中的原始输入事件，将这些原始输入事件进行加工后交由 InputDispatcher 处理，InputDispatcher 中保存了 WMS 的所有窗口信息，这样就可以将输入事件派发给合适的窗口。

![IMS 架构简图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/IMS%20%E6%9E%B6%E6%9E%84%E7%AE%80%E5%9B%BE.drawio.png)

> INotify 是 Linux 内核 2.6.13 以后支持的一种特性，功能是监视文件系统的变化，在监视到文件系统发生变化
> 以后，会向相应的应用程序发送变化事件。
>
> Epoll 是一种 I/O 时间通信机制，是 Linux 内核实现 IO 多路复用的一种方式。一次监听多个 fd 的可读/可写状态，当 fd 关联的内核缓冲区非空时，发出可读信号；当缓冲区不满时，发出可写信号。

### 三、IMS 的启动过程

![IMS 的启动过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/IMS%20%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.drawio.png)

### 四、InputDispatcher 的启动过程

```c++
// InputDispatcher.h
class InputDispatcherThread : public Thread {
    public:
    explicit InputDispatherThread(const sp<InputDispatcherInterface>& dispatcher);
    ~InputDispathcerThread();
    private:
    virtual bool threadLoop();
    sp<InputDispatcherInterface> mDispatcher;
};
```

InputDispatcher.h 中定义了 threadLoop 纯虚函数，InputDispather 继承了 Thread。native 的 Thread 内部有一个循环，当线程运行时，会调用上面代码的 threadLoop 函数，如果返回 true 并没有调用 requestExit 函数，就会接着循环调用 threadLoop 函数。（可参考文章[Android Framework中的线程Thread及它的threadLoop方法](https://blog.csdn.net/briblue/article/details/51104230)）

![InputDispatcher 的启动过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/InputDispatcher%20%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.drawio.png)

### 五、InputReader 处理事件的过程

键盘输入事件的调用时序图

![键盘输入事件的调用时序图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/%E9%94%AE%E7%9B%98%E8%BE%93%E5%85%A5%E4%BA%8B%E4%BB%B6.drawio.png)

```c++
void InputReader::loopOnce() {
    ...
        // 获取事件信息并存放在 mEventBuffer 中
        size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);
    ...
        if(count) {
            // 对事件进行加工处理
            processEventsLocked(mEventBuffer, count)
        }
}
```

在`processEventsLocked`中，对事件进行遍历分类出原始输入事件和设备事件，同一设备的输入事件交由`processEventsForDeviceLocked`处理。真正加工原始输入事件的是 InputMapper 对象，如 KeyboardInputMapper 或 TouchInputMapper，处理完事件封装成对应 NotifyKeyArgs 并告知给 InputListenerInterface，其实现也就是 InputDispatcher（InputDispatcher 继承了 InputDispatcherInterface，而 InputDispatcherInterface 继承了 InputListenerInterface）。

分发的最后，会根据 KeyEntry 来判断是否需要将睡眠中的 InputDispatcherThread 唤醒。

输入事件的处理总结：

1. IMS 启动了 InputDispatcherThread 和 InputReaderThread，它们分别运行 InputDispatcher 和 InputReader；
2. InputDispatcher 先于 InputReader 被创建，InputDispatcher 的`dispatchOnceInnerLocked`用来将事件分发给合适的窗口。InputDispatcher 没有输入事件处理时会进入睡眠状态，等待 InputReader 来通知唤醒；
3. InputReader 通过 EventHub 的`getEvents`来获取事件信息，如果是原始输入事件，则将这些事件交由不同的 InputMapper 来处理，最终交由 InputDispatcher 来进行分发；
4. 在 InputDispatcher 的`notifyKey`中会根据按键数据来判断`InputDispatcher`是否需要被唤醒，若被唤醒后，会重新唤醒进行分发事件。

![IMS 架构图](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/IMS%20%E6%9E%B6%E6%9E%84%E5%9B%BE.drawio.png)

### 六 、InputDispatcher  的分发过程

不同的事件类型有不同的分发过程，其中 Switch 事件的处理时没有分发过程的，在 InputDispatcher 的 notifySwitch 函数中会将 Switch 事件交由 InputDispatcherPolicy 来处理。以下为 Motion 事件的分发过程。

InputDispatcher 的`notifyMotion`来唤醒 InputDispatcherThread

```c++
void InputDispatcher::notifyMotion(const NotifyMotionArgs* args) {
#if DEBUG_INBOUND_EVENT_DETAILS
    ...
#endif
    if (!validateMotionEvent(args->action, args->actionButton, args->pointerCount,
                             args->pointerProperties)) { // 事件参数是否有效，内部会交叉触控点及其 id
        return;
    }

    uint32_t policyFlags = args->policyFlags;
    policyFlags |= POLICY_FLAG_TRUSTED;

    android::base::Timer t;
    mPolicy->interceptMotionBeforeQueueing(args->displayId, args->eventTime, /*byref*/ policyFlags);
    if (t.duration() > SLOW_INTERCEPTION_THRESHOLD) {
        ALOGW("Excessive delay in interceptMotionBeforeQueueing; took %s ms",
              std::to_string(t.duration().count()).c_str());
    }

    bool needWake;
    { // acquire lock
        mLock.lock();

        if (shouldSendMotionToInputFilterLocked(args)) { // Motion 事件是否需要交由 InputFilter 过滤
            mLock.unlock();

            MotionEvent event; // NotifyMotionArgs 参数构造一个 MotionEvent
            event.initialize(args->id, args->deviceId, args->source, args->displayId, INVALID_HMAC,
                             args->action, args->actionButton, args->flags, args->edgeFlags,
                             args->metaState, args->buttonState, args->classification, 1 /*xScale*/,
                             1 /*yScale*/, 0 /* xOffset */, 0 /* yOffset */, args->xPrecision,
                             args->yPrecision, args->xCursorPosition, args->yCursorPosition,
                             args->downTime, args->eventTime, args->pointerCount,
                             args->pointerProperties, args->pointerCoords);

            policyFlags |= POLICY_FLAG_FILTERED;
            if (!mPolicy->filterInputEvent(&event, policyFlags)) {
                return; // event was consumed by the filter
            }

            mLock.lock();
        }

        // Just enqueue a new motion event.
        MotionEntry* newEntry =
                new MotionEntry(args->id, args->eventTime, args->deviceId, args->source,
                                args->displayId, policyFlags, args->action, args->actionButton,
                                args->flags, args->metaState, args->buttonState,
                                args->classification, args->edgeFlags, args->xPrecision,
                                args->yPrecision, args->xCursorPosition, args->yCursorPosition,
                                args->downTime, args->pointerCount, args->pointerProperties,
                                args->pointerCoords, 0, 0);

        needWake = enqueueInboundEventLocked(newEntry); // 内部会将 MotionEntry 添加到 InputDispatcher 的 mInboundQueue 队列的队尾
        mLock.unlock();
    } // release lock

    if (needWake) {
        mLooper->wake();
    }
}
```

```c++
void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    ...

    // If dispatching is frozen, do not process timeouts or try to deliver any new events.
    if (mDispatchFrozen) {
        if (DEBUG_FOCUS) {
            ALOGD("Dispatch frozen.  Waiting some more.");
        }
        return;
    }

    // Optimize latency of app switches.
    // Essentially we start a short timeout when an app switch key (HOME / ENDCALL) has
    // been pressed.  When it expires, we preempt dispatch and drop all other pending events.
    bool isAppSwitchDue = mAppSwitchDueTime <= currentTime; // isAppSwitchDue 为 true，则说明没有及时响应 Home 等操作
    if (mAppSwitchDueTime < *nextWakeupTime) {
        *nextWakeupTime = mAppSwitchDueTime;
    }

    // Ready to start a new event.
    // If we don't already have a pending event, go grab one.
    if (!mPendingEvent) {
        if (mInboundQueue.empty()) {
            ...
            // Nothing to do if there is no pending event.
            if (!mPendingEvent) {
                return;
            }
        } else {
            // Inbound queue has at least one entry.
            mPendingEvent = mInboundQueue.front();
            mInboundQueue.pop_front();
            traceInboundQueueLengthLocked();
        }

        // Poke user activity for this event.
        if (mPendingEvent->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            pokeUserActivityLocked(*mPendingEvent);
        }
    }

    // Now we have an event to dispatch.
    // All events are eventually dequeued and processed this way, even if we intend to drop them.
    ALOG_ASSERT(mPendingEvent != nullptr);
    bool done = false;
    DropReason dropReason = DropReason::NOT_DROPPED;
    ...

    switch (mPendingEvent->type) {
        case EventEntry::Type::CONFIGURATION_CHANGED: {
            ConfigurationChangedEntry* typedEntry =
                    static_cast<ConfigurationChangedEntry*>(mPendingEvent);
            done = dispatchConfigurationChangedLocked(currentTime, typedEntry);
            dropReason = DropReason::NOT_DROPPED; // configuration changes are never dropped
            break;
        }
		...
        case EventEntry::Type::MOTION: {
            MotionEntry* typedEntry = static_cast<MotionEntry*>(mPendingEvent);
            if (dropReason == DropReason::NOT_DROPPED && isAppSwitchDue) { // 没有及时响应窗口切换操作
                dropReason = DropReason::APP_SWITCH;
            }
            if (dropReason == DropReason::NOT_DROPPED && isStaleEvent(currentTime, *typedEntry)) { // 事件过期
                dropReason = DropReason::STALE;
            }
            if (dropReason == DropReason::NOT_DROPPED && mNextUnblockedEvent) { // 阻碍其他窗口获取事件
                dropReason = DropReason::BLOCKED;
            }
            done = dispatchMotionLocked(currentTime, typedEntry, &dropReason, nextWakeupTime); // 为 Motion 事件寻找合适的窗口
            break;
        }
    }

    if (done) {
        if (dropReason != DropReason::NOT_DROPPED) {
            dropInboundEventLocked(*mPendingEvent, dropReason);
        }
        mLastDropReason = dropReason;

        releasePendingEventLocked(); // 释放本次事件处理的对象
        *nextWakeupTime = LONG_LONG_MIN; // force next poll to wake up immediately
    }
}
```

```c++
bool InputDispatcher::dispatchMotionLocked(nsecs_t currentTime, MotionEntry* entry,
                                           DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ATRACE_CALL();
    // Preprocessing.
    if (!entry->dispatchInProgress) {
        entry->dispatchInProgress = true;

        logOutboundMotionDetails("dispatchMotion - ", *entry);
    }

    // Clean up if dropping the event.
    if (*dropReason != DropReason::NOT_DROPPED) {
        setInjectionResult(entry,
                           *dropReason == DropReason::POLICY ? INPUT_EVENT_INJECTION_SUCCEEDED
                                                             : INPUT_EVENT_INJECTION_FAILED);
        return true; // 返回 true, 丢弃此次事件，清除
    }

    bool isPointerEvent = entry->source & AINPUT_SOURCE_CLASS_POINTER;

    // Identify targets.
    std::vector<InputTarget> inputTargets;

    bool conflictingPointerActions = false;
    int32_t injectionResult;
    if (isPointerEvent) {
        // Pointer event.  (eg. touchscreen)
        injectionResult =
                findTouchedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime,
                                               &conflictingPointerActions);
    } else {
        // Non touch event.  (eg. trackball)
        injectionResult =
                findFocusedWindowTargetsLocked(currentTime, *entry, inputTargets, nextWakeupTime);
    }
    if (injectionResult == INPUT_EVENT_INJECTION_PENDING) { // 输入事件被挂起，说明找到了窗口并且窗口无反应
        return false;
    }

    setInjectionResult(entry, injectionResult);
    if (injectionResult == INPUT_EVENT_INJECTION_PERMISSION_DENIED) {
        ALOGW("Permission denied, dropping the motion (isPointer=%s)", toString(isPointerEvent));
        return true;
    }
    if (injectionResult != INPUT_EVENT_INJECTION_SUCCEEDED) {
        CancelationOptions::Mode mode(isPointerEvent
                                              ? CancelationOptions::CANCEL_POINTER_EVENTS
                                              : CancelationOptions::CANCEL_NON_POINTER_EVENTS);
        CancelationOptions options(mode, "input event injection failed");
        synthesizeCancelationEventsForMonitorsLocked(options);
        return true;
    }

    // Add monitor channels from event's or focused display.
    // 将分发的目标添加到 inputTargets 列表中
    addGlobalMonitoringTargetsLocked(inputTargets, getTargetDisplayId(*entry));

    if (isPointerEvent) {
        std::unordered_map<int32_t, TouchState>::iterator it =
                mTouchStatesByDisplay.find(entry->displayId);
        if (it != mTouchStatesByDisplay.end()) {
            const TouchState& state = it->second;
            if (!state.portalWindows.empty()) {
                // The event has gone through these portal windows, so we add monitoring targets of
                // the corresponding displays as well.
                for (size_t i = 0; i < state.portalWindows.size(); i++) {
                    const InputWindowInfo* windowInfo = state.portalWindows[i]->getInfo();
                    addGlobalMonitoringTargetsLocked(inputTargets, windowInfo->portalToDisplayId,
                                                     -windowInfo->frameLeft, -windowInfo->frameTop);
                }
            }
        }
    }

    // Dispatch the motion.
    if (conflictingPointerActions) {
        CancelationOptions options(CancelationOptions::CANCEL_POINTER_EVENTS,
                                   "conflicting pointer actions");
        synthesizeCancelationEventsForAllConnectionsLocked(options);
    }
    dispatchEventLocked(currentTime, entry, inputTargets); // 将事件分发给 inputTargets 列表中的目标
    return true;
}
```

```c++
/*
 * An input target specifies how an input event is to be dispatched to a particular window
 * including the window's input channel, control flags, a timeout, and an X / Y offset to
 * be added to input event coordinates to compensate for the absolute position of the
 * window area.
 */
struct InputTarget {
    ...
}
```

InputTarget 定义了一个 Input 事件怎么分发到指定窗口，包括窗口的 input channel，控制相关的 flags、相关触碰点位置等信息。

处理点击形式的事件，也即是 InputDispatcher 的`findTouchedWindowTargetsLocked`，详情读者可自行查看代码了解。以下是 Motion 事件分发过程图。

![Motion 事件分发过程](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Motion%20%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E8%BF%87%E7%A8%8B.drawio.png)

总结：

1. 在 InputReaderThread 线程的 InputReader 中进行加工，判断是否唤醒 InputDispatcherThread，若唤醒则不断用 InputDispatcher 来分发 Motion 事件；
2. InputFilter 过滤；
3. Motion 事件加工后数据结构 NotifyMotionArgs，`notifyMotion`，依 NotifyMotionArgs 构造 MotionEntry，再添加到队尾 mInboundQueue；
4. mPendingEvent = mInboundQueue 队头 EventEntry；
5. mPendingEvent  事件丢弃处理；
6. 找到目标窗口，生成对应 inputTarget；
7. 通过 InputTarget 获取一个 Connection，并依赖 Connection 将输入事件发送给目标窗口。

 

