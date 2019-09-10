### HolderFragment 什么时候用到

每个 ViewModel 的对象都被保存在 ViewModelStore 对象里面，而 sdk 26 级以上的 FragmentActivity 和 Fragment 已经做了适配，里面有 ViewModelStore 的对象实例，而 26 以下的会在当前页嵌入一个 HolderFragment ，主要是去监听 onDestory 事件，进行 ViewModel 的 onCleared 方法。
```Java
    public static ViewModelStore of(@NonNull FragmentActivity activity) {
        if (activity instanceof ViewModelStoreOwner) {
            return ((ViewModelStoreOwner) activity).getViewModelStore();
        }
        return holderFragmentFor(activity).getViewModelStore();
    }
```
如果配置发生变化，那么这个 HolderFragment 也会执行到 onDestory ，那么 ViewModel 不就被清了吗？在 HolderFragment 的构造方法里面设置了 `setRetainInstance(true);` ，作用是配置发生变化时不会销毁 Fragment。

### 在 FragmentActivity 中贯彻生命周期

看 FragmentActivity 的 onRetainNonConfigurationInstance 方法，这里将 ViewModelStore 进行了缓存，然后在 getViewModelStore 中先从缓存中读取，所以 ViewModel 不会随着 Activity 的销毁而销毁
```Java
    public final Object onRetainNonConfigurationInstance() {
        Object custom = this.onRetainCustomNonConfigurationInstance();
        FragmentManagerNonConfig fragments = this.mFragments.retainNestedNonConfig();
        if (fragments == null && this.mViewModelStore == null && custom == null) {
            return null;
        } else {
            FragmentActivity.NonConfigurationInstances nci = new FragmentActivity.NonConfigurationInstances();
            nci.custom = custom;
            nci.viewModelStore = this.mViewModelStore;
            nci.fragments = fragments;
            return nci;
        }
    }
    
    @NonNull
    public ViewModelStore getViewModelStore() {
        if (this.getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the Application instance. You can't request ViewModel before onCreate call.");
        } else {
            if (this.mViewModelStore == null) {
                FragmentActivity.NonConfigurationInstances nc = (FragmentActivity.NonConfigurationInstances)this.getLastNonConfigurationInstance();
                if (nc != null) {
                    this.mViewModelStore = nc.viewModelStore;
                }

                if (this.mViewModelStore == null) {
                    this.mViewModelStore = new ViewModelStore();
                }
            }

            return this.mViewModelStore;
        }
    }
```

### 在 Fragment 中贯彻生命周期

看 fragment 是的 ViewModel 是如何进行缓存的，我们先看 FragmentActivity 的 onSaveInstanceState 方法
```Java
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        this.markFragmentsCreated();
        Parcelable p = this.mFragments.saveAllState();
        ...
    }
```
这里的 mFragments 是 FragmentController 类型，fragment 的控制类，执行了 saveAllState 方法最终会生成一个 FragmentManagerNonConfig 类对象

```Java
    Parcelable saveAllState() {
        ...
          this.saveNonConfig();
        ...
    }
    
    void saveNonConfig() {
        ...

        if (fragments == null && childFragments == null && viewModelStores == null) {
            this.mSavedNonConfig = null;
        } else {
            this.mSavedNonConfig = new FragmentManagerNonConfig(fragments, childFragments, viewModelStores);
        }

    }
```

这个生成的 FragmentManagerNonConfig 类里面就保持着 Fragment 的 ViewModel 类，然后会随着 Activity 的 ViewModel 一起缓存，回到 FragmentActivity 的 onRetainNonConfigurationInstance 方法看看
```Java
    public final Object onRetainNonConfigurationInstance() {
        Object custom = this.onRetainCustomNonConfigurationInstance();
        // 读取 FragmentManagerNonConfig
        FragmentManagerNonConfig fragments = this.mFragments.retainNestedNonConfig();
        if (fragments == null && this.mViewModelStore == null && custom == null) {
            return null;
        } else {
            FragmentActivity.NonConfigurationInstances nci = new FragmentActivity.NonConfigurationInstances();
            nci.custom = custom;
            nci.viewModelStore = this.mViewModelStore;
            // 进行缓存
            nci.fragments = fragments;
            return nci;
        }
    }
```

然后会在 FragmentActivity 重新创建的时候恢复 Fragment 的状态
```Java
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        this.mFragments.attachHost((Fragment)null);
        super.onCreate(savedInstanceState);
        FragmentActivity.NonConfigurationInstances nc = (FragmentActivity.NonConfigurationInstances)this.getLastNonConfigurationInstance();
        ...
        this.mFragments.restoreAllState(p, nc != null ? nc.fragments : null);
        ...
    }
```

### 什么时候执行 onCleared

看 FragmentActivity 和 Fragment 的 onDestory
```Java
// FragmentActivity
    protected void onDestroy() {
        super.onDestroy();
        if (this.mViewModelStore != null && !this.isChangingConfigurations()) {
            this.mViewModelStore.clear();
        }

        this.mFragments.dispatchDestroy();
    }
// Fragment
    public void onDestroy() {
        this.mCalled = true;
        FragmentActivity activity = this.getActivity();
        boolean isChangingConfigurations = activity != null && activity.isChangingConfigurations();
        if (this.mViewModelStore != null && !isChangingConfigurations) {
            this.mViewModelStore.clear();
        }
    }
```
看到有判断是否是配置发生变化的销毁，只有在不是配置发生变化的情况下（即 finish 退出）销毁的才会执行 ViewModel 对象的 onCleared 方法。

源码版本 1.1.1