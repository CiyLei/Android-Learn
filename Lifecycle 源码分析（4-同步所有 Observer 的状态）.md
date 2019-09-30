### 这张图要时时刻刻记住
![d2a08956897cb51fe6e2c9af8286c982.png](https://github.com/CiyLei/Android-Learn/blob/master/img/lifecycle1.png?raw=true)

重新在回到 addObserver 的代码，假设有多线程的情况，上一个 Observer 还没处理完毕，刚执行完 pushParentState，在 mParentStates 中压入了一个 INITIALIZED 的情况
```Java
@Overridepublic void addObserver(@NonNull LifecycleObserver observer) {
        // 那么这里的 mAddingObserverCounter 就不对等于 0，所以 isReentrance 就是 true
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        // 这个时候 targetState 就等于上一个 Observer 的 state，就是 INITIALIZED
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        // 那 statefulObserver.mState 的初始 state 是 INITIALIZED，targetState 也是 INITIALIZED，那就不会执行 while 了
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }
        // 这是也不会触发到这个 sync，将会由第一个还没处理完 Observer 的线程处理，应该那时候这个 isReentrance 还是 false
        if (!isReentrance) {
            sync();
        }
        mAddingObserverCounter--;
    }
```
```Java
    private State calculateTargetState(LifecycleObserver observer) {
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);
        State siblingState = previous != null ? previous.getValue().mState : null;
        // 这时候读出来的就是 INITIALIZED
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1) : null;
        // 自然 INITIALIZED 最小，返回的就是这个
        return min(min(mState, siblingState), parentState);
    }
```

### 小总结
上面多线程的情况，第一个线程中的 Observer 能正常进入 while 执行 event 和更新 state，然后触发 sycn，之后线程中除了将 Observer 添加进 mObserverMap，基本什么事情也没干。

-----------------------------
然后我们看 sync
```Java
    private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                    + "new events from it.");
            return;
        }
        // 先判断所有的 Observer 的 state 都相等吗，如果不相等，就同步他们的 state
        while (!isSynced()) {
            mNewEventOccurred = false;
            // 后退状态，对应图中从右到左的情况
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            // 前进状态，对应图中从左到右的情况
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
```
```Java
    private boolean isSynced() {
        // 没有 Observer 自然就不需要同步
        if (mObserverMap.size() == 0) {
            return true;
        }
        // 因为每次同步状态会将所有的 Observer 的 state 更新掉，所以只要判断首尾的 Observer 是否相等就好了，因为首尾不相等，所有 Observer 的 state 肯定没有同步过，因为同步过，所有的 Observer 的 state 是相等的，自然也包括首尾。
        State eldestObserverState = mObserverMap.eldest().getValue().mState;
        State newestObserverState = mObserverMap.newest().getValue().mState;
        return eldestObserverState == newestObserverState && mState == newestObserverState;
    }
```

### 小总结
首先先判断了 mObserverMap 中所有的 Observer 的 state 是否已经同步统一，如果没有的话，判断是该前进状态了还是后退状态了
> 不是说 addObserver 没有从右到左的情况吗？那哪来的后退状态？
> 因为sync不止在 addObserver 中调用，当时如果在 addObserver 中调用的，的确不会出现后退状态的情况（即backwardPass）

------------------------------

因为我们现在将的还是 addObserver 的情况，所有就看 `forwardPass` 方法吧
```Java
    private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        // 循环出所有 Observer 对象
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            // 如果 Observer 的 state 小于 mState，就前进一步状态
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }
```
 这上面的代码跟 addObserver 的更新 state 的代码很像，功能也差不多，只不过这个会更新所有的 Observer。
 上面也讲了，多线程重入的情况下，后面线程的 Observer 是没有更新 state 的，在这里，由第一个线程统一更新的 state。
 
 ----------------------------
 
 到此，我们讲了在没有 Observer 的情况下，Lifecycle 是如何监听生命周期的，有讲了添加 Observer 的时候又发生了什么，但是添加完毕之后，所有的 Observer 的 state 能更新完毕了之后，生命周期发生了变化又发生了什么呢？
其实如果上面你都看懂了，那么就非常非常简单了。在第一章说到了，生命周期发生变化时会触发到 LifecycleRegistry 的 handleLifecycleEvent 方法的
```Java
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }

    private void moveToState(State next) {
        if (mState == next) {
            return;
        }
        mState = next;
        if (mHandlingEvent || mAddingObserverCounter != 0) {
            mNewEventOccurred = true;
            return;
        }
        mHandlingEvent = true;
        sync();
        mHandlingEvent = false;
    }
```
更新了 mState 之后就触发了 sync，因为这里是生命周期发生变化的时候调用的，所以会有 ON_PAUSE、ON_STOP、ON_DESTORY的event 的，在 sync 中也是会处理后退状态的，上面也讲了，会触发所有的 Observer 的 event 和 更新 state 。