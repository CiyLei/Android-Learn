### 这张图要时时刻刻记住
![d2a08956897cb51fe6e2c9af8286c982.png](https://github.com/CiyLei/Android-Learn/blob/master/img/lifecycle1.png?raw=true)

其实不难得出，Lifecycle 关于生命周期的监听是在当前 activity 中注入了一个 ReportFragment 来实现的。
注入的代码在 SupportActivity 的 onCreate 中
```Java
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
    }
```
```Java
public class ReportFragment extends Fragment {
    public static void injectIfNeededIn(Activity activity) {
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
    }
}
```

然后在生命周期改变时，拿到 activity 中的 LifecycleRegistry 对象，开始逻辑了
```Java
public class ReportFragment extends Fragment {
    ...
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        mProcessListener = null;
    }
    
    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
}
```

Lifecycle 中生命周期变化和状态的改变的逻辑都在 LifecycleRegistry 中实现，我们直接看 LifecycleRegistry 的 handleLifecycleEvent 方法

```Java
    
    public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
        State next = getStateAfter(event);
        moveToState(next);
    }
    
    static State getStateAfter(Event event) {
        switch (event) {
            case ON_CREATE:
            case ON_STOP:
                return CREATED;
            case ON_START:
            case ON_PAUSE:
                return STARTED;
            case ON_RESUME:
                return RESUMED;
            case ON_DESTROY:
                return DESTROYED;
            case ON_ANY:
                break;
        }
        throw new IllegalArgumentException("Unexpected event value " + event);
    }
    
    private void moveToState(State next) {
        ...
        mState = next;
        ...
    }
```
我注释掉了 **现在暂时** 无用的代码，可以看出，逻辑非常的简单
1. 先根据生命周期的时间（event）得出下一个是什么状态（state）
2. 将得出的下一个状态赋值给 mState

### 结论
activity 每更改一次生命周期，LifecycleRegistry 中的 mState 都会更换到对应的状态
| onCreate | onStart | onResume | onPause | onStop | onDestory |
| --- | --- | --- | --- | --- | --- |
| CREATED | STARTD | RESUMED | STARTD | CREATED | DESTORYD |

------------------
分析的源码版本：1.1.1