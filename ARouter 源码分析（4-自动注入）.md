### inject
回到`_ARouter` 的 `inject` 方法
```Java
    static void inject(Object thiz) {
        AutowiredService autowiredService = ((AutowiredService) ARouter.getInstance().build("/arouter/service/autowired").navigation());
        if (null != autowiredService) {
            autowiredService.autowire(thiz);
        }
    }
```
我们可以知道，autowiredService 可以是某个继承了 AutowiredService 接口并且加了 Router 注解的类，那就是 AutowiredServiceImpl 类了
```Java
@Route(path = "/arouter/service/autowired")
public class AutowiredServiceImpl implements AutowiredService {
    private LruCache<String, ISyringe> classCache;
    private List<String> blackList;

    @Override
    public void init(Context context) {
        classCache = new LruCache<>(66);
        blackList = new ArrayList<>();
    }

    @Override
    public void autowire(Object instance) {
        String className = instance.getClass().getName();
        try {
            if (!blackList.contains(className)) {
                ISyringe autowiredHelper = classCache.get(className);
                if (null == autowiredHelper) {  
                    // SUFFIX_AUTOWIRED 是 $$ARouter$$Autowired
                    autowiredHelper = (ISyringe) Class.forName(instance.getClass().getName() + SUFFIX_AUTOWIRED).getConstructor().newInstance();
                }
                autowiredHelper.inject(instance);
                classCache.put(className, autowiredHelper);
            }
        } catch (Exception ex) {
            blackList.add(className);    // This instance need not autowired.
        }
    }
}
```
上面的逻辑也很简单，注解生成的类在章节1中有说明
反射获取注解生成的类，然后调用 `inject` 方法
我们可以看看某个 XXX$$ARouter$$Autowired 类的 `inject` 方法

```Java
public class MainActivity$$ARouter$$Autowired implements ISyringe {
  private SerializationService serializationService;

  @Override
  public void inject(Object target) {
    serializationService = ARouter.getInstance().navigation(SerializationService.class);
    MainActivity substitute = (MainActivity)target;
    substitute.liveService = ARouter.getInstance().navigation(LiveService.class);
    substitute.imService = ARouter.getInstance().navigation(IMService.class);
  }
}
```

这里是通过类对象获取目标对象，我们在章节1中说过，ARouter初始化的时候会将所有 IProvider 等信息已类名为key，路由抽象信息为value保存到 Warehouse.providersIndex 中，这里会重新读取路由信息的抽象，就可以走路由正常的返回目标对象了。具体可以看 `LogisticsCenter.buildProvider` ，非常简单。
```Java
    public static Postcard buildProvider(String serviceName) {
        //  Warehouse.providersIndex 缓存中读取
        RouteMeta meta = Warehouse.providersIndex.get(serviceName);

        if (null == meta) {
            return null;
        } else {
            return new Postcard(meta.getPath(), meta.getGroup());
        }
    }
```