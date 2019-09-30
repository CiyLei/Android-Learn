### 这张图要时时刻刻记住
![d2a08956897cb51fe6e2c9af8286c982.png](https://github.com/CiyLei/Android-Learn/blob/master/img/lifecycle1.png?raw=true)

可以继续回到 addObserver 
```Java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        // 给定 Observer 一个初始状态
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        // 防止重复添加 Observer
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
        if (previous != null) {
            return;
        }
        // 防止空指针
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            return;
        }
        // 判断现在是否在执行某个 Observer ，如果是，则值为 true，如果不是，则值为 false，后面就会触发 sync()。应该为多线程做处理
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        // 计算当前 Observer 要到达那个 state
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        // 每次只会进入下一个状态，所以循环判断是否达到了目标状态
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            // 先当前 Observer 的 state 压入 mParentStates 栈
            pushParentState(statefulObserver.mState);
            // 触发 Observer 的下一个事件和状态
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            // 出栈
            popParentState();
            // 重新计算目标状态
            targetState = calculateTargetState(observer);
        }

        if (!isReentrance) {
            // 将所有的 Observer 同步到一个状态
            sync();
        }
        mAddingObserverCounter--;
    }
```
> 其实上面可以看得出来，有针对多线程做处理，但能力有限，不是很能理解，特别是牵扯到 mParentStates 的 `calculateTargetState` 方法和 `pushParentState` 方法，而且我也试过用多线程试试看，发现很容易丢失处理某个 Observer ，所以我们接下来就不看他的多线程处理了。

先看 calculateTargetState 方法是如何计算目标 state 的
```Java
    private State calculateTargetState(LifecycleObserver observer) {
        // 获取上一个添加的 Observer
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);
        // 得到上一个添加的 Observer 的 state，上面也说了不考虑多线程的情况，那么这个 Observer 的 state 应该已经达到了目标状态，所以这个 siblingState 应该是等于 mState
        State siblingState = previous != null ? previous.getValue().mState : null;
        // 不考虑多线程情况，这里是就返回 null
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
                : null;
        // 不考虑多线程的情况，这里就是返回 mState
        return min(min(mState, siblingState), parentState);
    }
```
我们可以得出，在不考虑多线程的情况下，返回的就是 mState，mState 就是 Observer 的目标状态
而 mState 的状态是什么，怎么改变的已经在 章节1 里面提到过。

再看是如何触发 Observer 的下一个事件状态的
`statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));`
```Java
    private static Event upEvent(State state) {
        switch (state) {
            case INITIALIZED:
            case DESTROYED:
                return ON_CREATE;
            case CREATED:
                return ON_START;
            case STARTED:
                return ON_RESUME;
            case RESUMED:
                throw new IllegalArgumentException();
        }
        throw new IllegalArgumentException("Unexpected state value " + state);
    }
```
结合 **Lifecycle 的 state 和 event 的意义和为什么这么处理** 这个篇章里面说的和上面那张图，其实不难得出，在 addObserver 的时候，是不会出现迂回的情况的，那么是只考虑从左到右的情况了，所以这里返回的 event 是不会有 ON_PAUSE、ON_STOP、ON_DESTORY 的
```Java
        void dispatchEvent(LifecycleOwner owner, Event event) {
            // 得出该 event 的下一个 state 是什么
            State newState = getStateAfter(event);
            // 。。。以我有限的水平，看不出这行的意义
            mState = min(mState, newState);
            // 开始触发 Observer 的 event 了（即那些生命周期对应的方法了）
            mLifecycleObserver.onStateChanged(owner, event);
            // 更新 Observer 的 state
            mState = newState;
        }
```

再继续看 `mLifecycleObserver.onStateChanged(owner, event);` 
之前我们已经得知，这个 mLifecycleObserver 的实际类型是 ReflectiveGenericLifecycleObserver 
```Java
class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {
    ...
    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```
```Java
        void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
            // 触发这个 event 对应的所有方法
            invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
            // 触发 Lifecycle.Event.ON_ANY 的所有方法
            invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
                    target);
        }

        private static void invokeMethodsForEvent(List<MethodReference> handlers,
                LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
            if (handlers != null) {
                for (int i = handlers.size() - 1; i >= 0; i--) {
                    handlers.get(i).invokeCallback(source, event, mWrapped);
                }
            }
        }
```
到此，基本就差不多了，这里看不明白的看 **Lifecycle 源码分析（2-获取 LifecycleObserver 中对应的生命周期方法）** 篇章的总结，这个 mEventToHandlers 里面保存了所有已生命周期为key，对应的方法为value 的一个map。
`invokeCallback` 方法也很简单，`handlers.get(i)` 返回的对象是 MethodReference，里面有一个 mCallType 值对应着参数个数的类型，根据这个值调用反射调用方法就好了
```Java
        void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
            try {
                switch (mCallType) {
                    // 只有一个参数的类型
                    case CALL_TYPE_NO_ARG:
                        mMethod.invoke(target);
                        break;
                    // 只有两个参数的类型
                    case CALL_TYPE_PROVIDER:
                        mMethod.invoke(target, source);
                        break;
                    // 只有三个参数的类型
                    case CALL_TYPE_PROVIDER_WITH_EVENT:
                        mMethod.invoke(target, source, event);
                        break;
                }
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Failed to call observer method", e.getCause());
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        }
```

### 总结
在 addObserver 的时候，只考虑图中从左到右的情况，先计算 Observer 要到达的目标 state，不考虑多线程的情况下，就是 mState，即当前 activity 的 state，然后通过 while 一步步的执行下一个 event 和更新 state，直到与 mState 相等。