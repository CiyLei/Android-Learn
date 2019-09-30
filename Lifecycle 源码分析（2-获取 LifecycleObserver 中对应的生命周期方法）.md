假设我们通过 @OnLifecycleEvent 注解的方式来确定生命周期
```kotlin
class TestLifecycleObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onStart() {
        println("2222222222222222222222222 onStart")
    }
    ...
}
```

Lifecycle 是在 addObserver 的时候分析 LifecycleObserver 类的，分析出来结果是一个 ObserverWithState 类
```Java
    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ...
    }
```
```Java
    static class ObserverWithState {
        State mState;
        GenericLifecycleObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            mLifecycleObserver = Lifecycling.getCallback(observer);
            mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            State newState = getStateAfter(event);
            mState = min(mState, newState);
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```
可以看出来，重要的是 `Lifecycling.getCallback` 方法
```Java
    @NonNull
    static GenericLifecycleObserver getCallback(Object object) {
        // 假设我们的是 LifecycleObserver 的直接子类，所以不管这个
        if (object instanceof FullLifecycleObserver) {
            return new FullLifecycleObserverAdapter((FullLifecycleObserver) object);
        }
        // 假设我们的是 LifecycleObserver 的直接子类，所以不管这个
        if (object instanceof GenericLifecycleObserver) {
            return (GenericLifecycleObserver) object;
        }
        final Class<?> klass = object.getClass();
        // 判断我们该用反射去调用方法还是通过apt注解生成类的方式去调用方法
        int type = getObserverConstructorType(klass);
        // 如果是通过apt注解生成类的方式去调用方法的话（一般都是反射调用，不知道什么情况下这里会调用）
        if (type == GENERATED_CALLBACK) {
            List<Constructor<? extends GeneratedAdapter>> constructors =
                    sClassToAdapters.get(klass);
            if (constructors.size() == 1) {
                GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                        constructors.get(0), object);
                return new SingleGeneratedAdapterObserver(generatedAdapter);
            }
            GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
            for (int i = 0; i < constructors.size(); i++) {
                adapters[i] = createGeneratedAdapter(constructors.get(i), object);
            }
            return new CompositeGeneratedAdaptersObserver(adapters);
        }
        return new ReflectiveGenericLifecycleObserver(object);
    }
```
这个返回的 ReflectiveGenericLifecycleObserver 先不看，先看 `getObserverConstructorType()`

```Java
    private static int resolveObserverCallbackType(Class<?> klass) {
        // 如果是匿名类的话，则反射调用
        if (klass.getCanonicalName() == null) {
            return REFLECTIVE_CALLBACK;
        }
        // 如果找得到 XXXX_LifecycleAdapter 类的话，则这个生成类调用
        Constructor<? extends GeneratedAdapter> constructor = generatedConstructor(klass);
        if (constructor != null) {
            sClassToAdapters.put(klass, Collections
                    .<Constructor<? extends GeneratedAdapter>>singletonList(constructor));
            return GENERATED_CALLBACK;
        }
        // 如果类中有 @OnLifecycleEvent 注解的话，则反射调用并同时分析所有注解的方法
        boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
        if (hasLifecycleMethods) {
            return REFLECTIVE_CALLBACK;
        }
        // 递归调用查找父类的调用类型是什么
        Class<?> superclass = klass.getSuperclass();
        List<Constructor<? extends GeneratedAdapter>> adapterConstructors = null;
        if (isLifecycleParent(superclass)) {
            if (getObserverConstructorType(superclass) == REFLECTIVE_CALLBACK) {
                return REFLECTIVE_CALLBACK;
            }
            adapterConstructors = new ArrayList<>(sClassToAdapters.get(superclass));
        }
        // 递归调用查找所有接口的调用类型是什么
        for (Class<?> intrface : klass.getInterfaces()) {
            if (!isLifecycleParent(intrface)) {
                continue;
            }
            if (getObserverConstructorType(intrface) == REFLECTIVE_CALLBACK) {
                return REFLECTIVE_CALLBACK;
            }
            if (adapterConstructors == null) {
                adapterConstructors = new ArrayList<>();
            }
            adapterConstructors.addAll(sClassToAdapters.get(intrface));
        }
        // 如果父类获取继承的接口中有 apt生成类调用类型的，就返回这个类型
        if (adapterConstructors != null) {
            sClassToAdapters.put(klass, adapterConstructors);
            return GENERATED_CALLBACK;
        }
        // 默认返回反射调用类型
        return REFLECTIVE_CALLBACK;
    }
```
做个小总结：

| REFLECTIVE_CALLBACK | GENERATED_CALLBACK |
| --- | --- |
| 到时候我反射调用Observer的方法 | 到时候我通过apt生成类的方法去调用Observer中的方法 |

返回 GENERATED_CALLBACK 的几种情况：
1. 如果找得到 XXXX_LifecycleAdapter 类的话
2. 如果父类是 GENERATED_CALLBACK
3. 如果继承的接口类中有是 GENERATED_CALLBACK

继续回到上面 `ClassesInfoCache.sInstance.hasLifecycleMethods(klass)` 这个方法还是比较关键的
他会去分析类中是否有 @OnLifecycleEvent 注解过的方法，并把所有符合的方法全部分析保存下来
```Java
    boolean hasLifecycleMethods(Class klass) {
        // 如果缓存中结果了，直接返回
        if (mHasLifecycleMethods.containsKey(klass)) {
            return mHasLifecycleMethods.get(klass);
        }
        // 循环这个类中所有的方法
        Method[] methods = getDeclaredMethods(klass);
        for (Method method : methods) {
            OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
            if (annotation != null) {
                // 如果某个方法上面有 OnLifecycleEvent 的注解，则开始分析所有方法了（重点了）
                createInfo(klass, methods);
                return true;
            }
        }
        mHasLifecycleMethods.put(klass, false);
        return false;
    }
```
继续查看 `createInfo(klass, methods);` 方法（最后一步了）
```Java
    private CallbackInfo createInfo(Class klass, @Nullable Method[] declaredMethods) {
        Class superclass = klass.getSuperclass();
        // 临时保存某个方法对应的生命周期
        Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
        if (superclass != null) {
            // 现将父类中的方法分析完毕保存下来
            CallbackInfo superInfo = getInfo(superclass);
            if (superInfo != null) {
                handlerToEvent.putAll(superInfo.mHandlerToEvent);
            }
        }
        // 验证接口中的生命周期方法有没有在父类中已经有了（不能重复，不理解后面解释）
        Class[] interfaces = klass.getInterfaces();
        for (Class intrfc : interfaces) {
            for (Map.Entry<MethodReference, Lifecycle.Event> entry : getInfo(
                    intrfc).mHandlerToEvent.entrySet()) {
                verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
            }
        }
        // 循环所有方法
        Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
        boolean hasLifecycleMethods = false;
        for (Method method : methods) {
            OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
            // 如果方法没有被 OnLifecycleEvent 注释过，则跳过
            if (annotation == null) {
                continue;
            }
            hasLifecycleMethods = true;
            Class<?>[] params = method.getParameterTypes();
            //  callType 类型代表这个方法有几个参数，CALL_TYPE_NO_ARG 默认代表0个
            int callType = CALL_TYPE_NO_ARG;
            // CALL_TYPE_PROVIDER 代表参数有 1 个，并且这个参数的类型必须是 LifecycleOwner
            if (params.length > 0) {
                callType = CALL_TYPE_PROVIDER;
                if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                    throw new IllegalArgumentException(
                            "invalid parameter type. Must be one and instanceof LifecycleOwner");
                }
            }
            Lifecycle.Event event = annotation.value();
            // CALL_TYPE_PROVIDER_WITH_EVENT 代表参数有 2 个，并且第一个类型必须是 LifecycleOwner，第二个类型必须是 Lifecycle.Event，OnLifecycleEvent 注解中的 value 必须是 Lifecycle.Event.ON_ANY
            if (params.length > 1) {
                callType = CALL_TYPE_PROVIDER_WITH_EVENT;
                if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                    throw new IllegalArgumentException(
                            "invalid parameter type. second arg must be an event");
                }
                if (event != Lifecycle.Event.ON_ANY) {
                    throw new IllegalArgumentException(
                            "Second arg is supported only for ON_ANY value");
                }
            }
            // 参数的数量大于2个就报错
            if (params.length > 2) {
                throw new IllegalArgumentException("cannot have more than 2 params");
            }
            // 方法的封装类，保持方法自身对象和 callType 具体怎么调用的类别
            MethodReference methodReference = new MethodReference(callType, method);
            // 验证一个方法上面是否只有一个 OnLifecycleEvent 注解，不是就报错
            verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
        }
        // 将分析好的方法封装保持到 CallbackInfo 中，并用两个 map 保存对应关系
        // 一个 map 保存一个方法对应的一个生命周期
        // 另一个 map 保存一个生命周期对应的多个方法
        // 没错 生命周期与方法的对应关系是 一对多
        CallbackInfo info = new CallbackInfo(handlerToEvent);
        mCallbackMap.put(klass, info);
        mHasLifecycleMethods.put(klass, hasLifecycleMethods);
        return info;
    }
```

出现了很多的变量和对象，这里总结理一理：
* CallbackInfo：保存着某个类中所有的生命周期对应方法
* mCallbackMap：已分析的类为key，缓存着对应的 CallbackInfo （防止重复分析）
* mHasLifecycleMethods：已分析的类为key，缓存着这个类中是否有 OnLifecycleEvent 注解的方法


这时候我们在回到 `Lifecycling.getCallback` 的方法中，去看返回 ReflectiveGenericLifecycleObserver 类又有什么
```Java
class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```
```Java
    CallbackInfo getInfo(Class klass) {
        CallbackInfo existing = mCallbackMap.get(klass);
        if (existing != null) {
            return existing;
        }
        existing = createInfo(klass, null);
        return existing;
    }
```
终于可以缓缓了，因为上面讲过了，mCallbackMap 中是有缓存的 CallbackInfo，这里就直接可以获取到。

### 总结
每一个 LifecycleObserver 都会生成一个 ObserverWithState 封装类
每一个 ObserverWithState 都会有一个 ReflectiveGenericLifecycleObserver 类对象
每一个 ReflectiveGenericLifecycleObserver 中都保存着 CallbackInfo 类对象
每一个 CallbackInfo 中都会有两个 map 对象，一个 map 保存着所有方法对应的一个生命周期，一个map 保持着所有生命周期对应的多个方法
每一个方法的封装对象是 MethodReference，里面保存了方法自身以外还保存了一个 mCallType，对应着方法的参数个数的类型

生命周期与方法的对应关系是 一对多，即一个方法上面不可以有多个 OnLifecycleEvent 注解，但是多个方法可以注解同一个 OnLifecycleEvent