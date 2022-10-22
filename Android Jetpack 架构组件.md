## Android Jetpack 架构组件

### 一、Lifecycle

![Lifecycle 观察生命周期](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Lifecycle%20%E8%A7%82%E5%AF%9F%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E8%B0%83%E7%94%A8%E9%93%BE.drawio.png)

![Lifecycle 关联类](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/Lifecycle%20%E5%85%B3%E8%81%94%E7%B1%BB.drawio.png)

### 二、LiveData

更改 LiveData 中的数据

`Transformations.map`

`Transformations.switchMap`

`MediatorLiveData`：合并多个 LiveData 数据源

```java
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        assertMainThread("observe");
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);// 存储到 SageIterableMap<Observer<? super T>>
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper); // 完成 Lifecycle 观察者的添加
    }
```

```java
    class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
        @NonNull
        final LifecycleOwner mOwner;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
            super(observer);
            mOwner = owner;
        }

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED); // 包括 STARTED 和 RESUMED 状态
        }

        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
                removeObserver(mObserver); // 观察组件移除
                return;
            }
            activeStateChanged(shouldBeActive());
        }

        @Override
        boolean isAttachedTo(LifecycleOwner owner) {
            return mOwner == owner;
        }

        @Override
        void detachObserver() {
            mOwner.getLifecycle().removeObserver(this);
        }
    }
```

```java
    private abstract class ObserverWrapper {
        final Observer<? super T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;

        ObserverWrapper(Observer<? super T> observer) {
            mObserver = observer;
        }

        abstract boolean shouldBeActive();

        boolean isAttachedTo(LifecycleOwner owner) {
            return false;
        }

        void detachObserver() {
        }

        void activeStateChanged(boolean newActive) {
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            // 根据 Active 状态和处于 Active 状态的组件的数量，对 onActive 方法和 onInactive 方法进行回调，这两个方法用于拓展 LiveData 对象。
            mActive = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += mActive ? 1 : -1;
            if (wasInactive && mActive) {
                onActive();
            }
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
            if (mActive) {
                dispatchingValue(this);
            }
        }
    }
```

```java
    void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) { // 正处在分发状态中
            mDispatchInvalidated = true; // 分发无效
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false; // 分发有效
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false; // 分发状态重置
    }
```

```java
    private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false); // 其方法内部会再次判断是否执行 onActive 和 onInactive 方法回调
            return;
        }
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        observer.mObserver.onChanged((T) mData);
    }
```

```java
    private final Runnable mPostValueRunnable = new Runnable() {
        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            setValue((T) newValue);
        }
    };
...
    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
    // 切换线程
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }
...
    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }
```

无论 LiveData 的 observe 方法还是 LiveData 的 postValue/setValue 都会调用 dispatchingValue。

![LiveData 粘性事件](https://pic3.zhimg.com/80/v2-e2c9c2051b16fe7030a97f2d65420172_720w.webp)

[LiveData粘性事件处理](https://zhuanlan.zhihu.com/p/480324760)

![LiveData 关联类 UML](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/LiveData%20%E5%85%B3%E8%81%94%E7%B1%BB%20UML.drawio.png)



