### 大致流程
接下来就是调用 `Postcard` 的 `navigation` 方法了，这个方法我们拆成2个步骤来说
1. LogisticsCenter.completion
2. navigation

### 1. LogisticsCenter.completion
调用 `Postcard` 的 `navigation` 方法，会执行到 `_ARouter` 的 `navigation` 方法中
```Java
    protected Object navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        ...
        LogisticsCenter.completion(postcard);
        ...
    }
```
其他的先不用看，直接看这行关键代码，点进去开始分析。
```Java
    public synchronized static void completion(Postcard postcard) {
        if (null == postcard) {
            throw new NoRouteFoundException(TAG + "No postcard!");
        }
        // 先从 Warehouse.routes 查找，是否有缓存
        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {    
            // 再从 Warehouse.groupsIndex 查找，是否有group的信息
            Class<? extends IRouteGroup> groupMeta = Warehouse.groupsIndex.get(postcard.getGroup());  
            if (null == groupMeta) {
                // 如果都没有找到，则就是查找一个不存在的路由
                throw new NoRouteFoundException(TAG + "There is no route match the path [" + postcard.getPath() + "], in group [" + postcard.getGroup() + "]");
            } else {
                try {
                    ...
                    // 如果在 Warehouse.groupsIndex 中查找到了，就将这个group下面所有的路由信息缓存到 Warehouse.routes 中
                    IRouteGroup iGroupInstance = groupMeta.getConstructor().newInstance();
                    iGroupInstance.loadInto(Warehouse.routes);
                    // 删除此 IRouteGroup 防止死递归
                    Warehouse.groupsIndex.remove(postcard.getGroup());
                    ...
                } catch (Exception e) {
                    throw new HandlerException(TAG + "Fatal exception when loading group meta. [" + e.getMessage() + "]");
                }
                // 调用自己 递归重新查找
                completion(postcard); 
            }
        } else {
            ...
        }
    }
```
从上面的逻辑可以得出，路由的加载是一组一组加载的，如果某个路由没有被加载进来，则会加入整个组的路由信息，这些组的路由信息由某个IRouteGroup维护，具体看 章节1

从下递归的话，在 Warehouse.routes 中就会查找到，那么就会进入 else 的分支
```Java
    public synchronized static void completion(Postcard postcard) {
        if (null == postcard) {
            throw new NoRouteFoundException(TAG + "No postcard!");
        }
        RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
        if (null == routeMeta) {
            ...
        } else {
            //  设置已缓存的字段
            postcard.setDestination(routeMeta.getDestination());
            postcard.setType(routeMeta.getType());
            postcard.setPriority(routeMeta.getPriority());
            postcard.setExtra(routeMeta.getExtra());
            ...
            switch (routeMeta.getType()) {
                // 如果是 IProvider的话
                case PROVIDER:  
                    //  获取目标对象
                    Class<? extends IProvider> providerMeta = (Class<? extends IProvider>) routeMeta.getDestination();
                    // 先从 Warehouse.providers 缓存中读取
                    IProvider instance = Warehouse.providers.get(providerMeta);
                    if (null == instance) { 
                        IProvider provider;
                        try {
                            // 如果缓存中没有的话，反射创建一个，然后缓存到 Warehouse.providers 中
                            provider = providerMeta.getConstructor().newInstance();
                            provider.init(mContext);
                            Warehouse.providers.put(providerMeta, provider);
                            instance = provider;
                        } catch (Exception e) {
                            throw new HandlerException("Init provider failed! " + e.getMessage());
                        }
                    }
                    // 将获取的 IProvider 设置到 Postcard 中
                    postcard.setProvider(instance);
                    // 设置绿色通道，即不经过拦截器
                    postcard.greenChannel();    
                    break;
                case FRAGMENT:
                    // fragment也设置绿色通道
                    postcard.greenChannel();    
                default:
                    break;
            }
        }
    }
```

### 总结
1. 因为 Warehouse.routes 中缓存了所有的路由对象信息，如果这个里面没有找到的话，就是还没有加载此路由对象，所以根据group在Warehouse.groupsIndex中取出了IRouteGroup对象，加载这个group下所有的路由信息。
2. 然后在 Warehouse.providers 中查找是否已经缓存过目标对象了，没有反射初始化然后缓存到 Warehouse.providers 中

### 2. navigation
回到  `_ARouter` 的 `navigation` ，下面执行到了  `_navigation`  方法
```Java
    private Object _navigation(final Context context, final Postcard postcard, final int requestCode, final NavigationCallback callback) {
        final Context currentContext = null == context ? mContext : context;

        switch (postcard.getType()) {
            // 如果是目标类型是 Activity
            case ACTIVITY:
                // Build intent
                final Intent intent = new Intent(currentContext, postcard.getDestination());
                intent.putExtras(postcard.getExtras());

                // Set flags.
                int flags = postcard.getFlags();
                if (-1 != flags) {
                    intent.setFlags(flags);
                } else if (!(currentContext instanceof Activity)) {    // Non activity, need less one flag.
                    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }

                // Set Actions
                String action = postcard.getAction();
                if (!TextUtils.isEmpty(action)) {
                    intent.setAction(action);
                }

                // Navigation in main looper.
                runInMainThread(new Runnable() {
                    @Override
                    public void run() {
                        startActivity(requestCode, currentContext, intent, postcard, callback);
                    }
                });

                break;
            // 如果是目标类型是 IProvider
            case PROVIDER:
                // 上面的 LogisticsCenter.completion 已经将生成的目标对象保存了，这里进行返回就进行
                return postcard.getProvider();
            ...
        }

        return null;
    }
```