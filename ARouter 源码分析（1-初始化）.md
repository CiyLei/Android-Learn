### 大致流程
1. 分析app所有的dex文件，得到 `com.alibaba.android.arouter.routes` 包下面所有的类
2. 将所有生成的类进行分类保存到 `Warehouse` 中

### 1. 分析app所有的dex文件，得到 `com.alibaba.android.arouter.routes` 包下面所有的类
从入口开始看
```Java
ARouter.init(Application application)
```
追踪下来，可以得出，最重要的部分还是在 `LogisticsCenter` 的 `init` 方法中
```Java
public synchronized static void init(Context context, ThreadPoolExecutor 
tpe) throws HandlerException {
    ...
    // 记录所有生成的类
    Set<String> routerMap;
    // 如果是debug状态或者app的版本更新了的话，就重新分析dex中所有的类
    if (ARouter.debuggable() || PackageUtils.isNewVersion(context)) {
        // 获取dex中所有生成的类
        routerMap = ClassUtils.getFileNameByPackageName(mContext, ROUTE_ROOT_PAKCAGE);
        // 将获取到的类列表进行缓存
        if (!routerMap.isEmpty()) {
            context.getSharedPreferences(AROUTER_SP_CACHE_KEY, 
                Context.MODE_PRIVATE).edit().putStringSet(AROUTER_SP_KEY_MAP, 
                routerMap).apply();
        }
        // 记录版本
        PackageUtils.updateVersion(context);
    } else {
        // 从缓存中读取所有生成的类
        routerMap = new HashSet<>(context.getSharedPreferences(AROUTER_SP_CACHE_KEY, 
            Context.MODE_PRIVATE).getStringSet(AROUTER_SP_KEY_MAP, new HashSet<String>()));
    }
    ...
}
```
以上为精简部分
`PackageUtils.isNewVersion` 会判断是否有记录版本，则是否是第一次初始化ARouter，如果是的话，也返回True，所以第一次是会运行此处。
可以看出，`ClassUtils.getFileNameByPackageName` 这个方法是关键，它分析了这个app的所有dex，得出所有的类，那点进去看。
```Java
    public static Set<String> getFileNameByPackageName(Context context, final String packageName) throws PackageManager.NameNotFoundException, IOException, InterruptedException {
        // 缓存所有生成的类
        final Set<String> classNames = new HashSet<>();
        // 获取所有源码路径（即dex）
        List<String> paths = getSourcePaths(context);
        // 一个计数器闭锁，当值为0时解锁，处理多线程并发
        final CountDownLatch parserCtl = new CountDownLatch(paths.size());
        // 这开了一个多线程进行加载源码（dex），并分析
        for (final String path : paths) {
            DefaultPoolExecutor.getInstance().execute(new Runnable() {
                @Override
                public void run() {
                    DexFile dexfile = null;
                    try {
                        // 判断是不是.zip结尾（居然还有这种的）
                        if (path.endsWith(EXTRACTED_SUFFIX)) {
                            dexfile = DexFile.loadDex(path, path + ".tmp", 0);
                        } else {
                            dexfile = new DexFile(path);
                        }
                        // 遍历所有类，取出所有以 com.alibaba.android.arouter.routes 打头的类
                        Enumeration<String> dexEntries = dexfile.entries();
                        while (dexEntries.hasMoreElements()) {
                            String className = dexEntries.nextElement();
                            if (className.startsWith(packageName)) {
                                classNames.add(className);
                            }
                        }
                    } catch (Throwable ignore) {
                        Log.e("ARouter", "Scan map file in dex files made error.", ignore);
                    } finally {
                        if (null != dexfile) {
                            try {
                                dexfile.close();
                            } catch (Throwable ignore) {
                            }
                        }
                        // 计数器减一
                        parserCtl.countDown();
                    }
                }
            });
        }
        // 等待所有源码分析完毕
        parserCtl.await();
        Log.d(Consts.TAG, "Filter " + classNames.size() + " classes by packageName <" + packageName + ">");
        return classNames;
    }
```
上面的逻辑还是比较清晰的，主要 `getSourcePaths` 比较关键，是获取app所有的源码文件的，但是这并不是ARouter的重点，所以有兴趣自己看。


### 2. 将所有生成的类进行分类保存到 `Warehouse` 中

经过上面的步骤，已经获取到了所有 `com.alibaba.android.arouter.routes` 包下的类，此包下就是保存了所有ARouter提前生成和注解生成的类。
开始 `Warehouse` 之前，我们先来理解一下注解生成类的逻辑，我们分为2大类，一种为生成在 `com.alibaba.android.arouter.routes` 包下面的路由生成类，一种是生成在原包名下面的注入类.

1. 生成在 `com.alibaba.android.arouter.routes` 包下面的路由生成类

    | 生成规则（XXX为路由根节点，即Group） | 说明 |
    | --- | --- |
    | ARouter$$Group$$XXX | 以路由路径为key，RouteMeta为value保存着XXX group下面的所有路由对象信息 |
    | ARouter$$Root$$XXX | 以group为key，ARouter$$Group$$XXX为value保存着XXX group下面的group信息 |
    | ARouter$$Providers$$XXX | 以路由路径为key，RouteMeta为value保存着XXX group下面的的 Provider 信息 |
    
2. 生成在原包名下面的注入类（最后用得到）

    | 生成规则（XXX为被注解的类） | 说明 |
    | --- | --- |
    | XXX$$ARouter$$Autowired | 自动注入的生成类 |

在 `Warehouse` 中有4个列表，我们先提前说明这4个列表是什么（不讲拦截器）。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| groupsIndex | `Map<String, Class<? extends IRouteGroup>>` | 使用 `ARouter$$Root$$XXX`，缓存了所有的 `ARouter$$Group$$`  信息（key为group） |
| routes | `Map<String, RouteMeta>` | 缓存了所有的路由信息（key为路由路径） |
| providersIndex | `Map<String, RouteMeta>` | 使用 `ARouter$$Providers$$XXX`，缓存了所有的 `Provider` 信息（key为类名） |
| providers | `Map<Class, IProvider>` | 缓存了所有的 `Provider` 信息（key为类对象，value为 `Provider`  对象） |

providersIndex 和 providers 区别：
因为ARouter是只能通过路由进行获取对象的，但是表面的api是可以通过类对象获取返回对象，这是怎么做到的呢？
只能初始化的时候就将 provider 和路由抽象对应起来，providersIndex里面存的就是这个

现在回到 `LogisticsCenter` 的 `init` 方法中
```Java
                ...
                for (String className : routerMap) {
                    // 解析 ARouter$$Root
                    if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_ROOT)) {
                        ((IRouteRoot) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.groupsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_INTERCEPTORS)) {
                    // 解析 ARouter$$Interceptors
                        ((IInterceptorGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.interceptorsIndex);
                    } else if (className.startsWith(ROUTE_ROOT_PAKCAGE + DOT + SDK_NAME + SEPARATOR + SUFFIX_PROVIDERS)) {
                    // 解析 ARouter$$Providers
                        ((IProviderGroup) (Class.forName(className).getConstructor().newInstance())).loadInto(Warehouse.providersIndex);
                    }
                }
```
就看解析ARouter$$Root和ARouter$$Providers的情况，很简单，就是反射初始化，然后调用生成类里面的 `loadInto` 将对象保存到 `Warehouse` 中，具体保存在哪里有什么意思，参考上面的表格。

### 结束
至此，已将所有注解生成和ARouter提前生成的类都分类保存进了`Warehouse` 中。
