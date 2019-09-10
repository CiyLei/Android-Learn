直接看 **HttpServiceMethod** 的 **parseAnnotations** 方法
```Java
  static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    // 1。取出 CallAdapterFactory 中的对象
    CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method);
    。。。
    // 2。取出 ConverterFactory 中的对象
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);
    // 3。取出 client 中的对象，默认对象是 OkHttpClient
    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    // 4。将上面3和对象都保存在 HttpServiceMethod 对象中
    return new HttpServiceMethod<>(requestFactory, callFactory, callAdapter, responseConverter);
  }
```
**CallAdapterFactory** 和 **ConverterFactory** 指的是创建 **Retrofit** 对象时，通过 **Builder** 设计模式传进去的对象。
但是无论是 **CallAdapterFactory** 还是 **ConverterFactory** 都是 **add** 进去的，那么就是说人家可能有多个对象，但是 **createCallAdapter** 和 **createResponseConverter** 方法都是返回了一个对象，返回的是哪个呢？
我们通过代码寻求答案，进过跟踪 **createCallAdapter**，最终到了 **Retrofit** 的 **nextCallAdapter** 方法，直接看其中的关键代码
```Java
  public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    。。。
    int start = callAdapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    。。。
```
非常简单，在 **callAdapterFactories** 中找到第一个适配的类，是根据 **returnType** 返回值的类型来判断该用哪个 **Adapter**，如果不为 **null** 就直接返回了，那 **createResponseConverter** 也是同一个思路吗？
直接看 **Retrofit** 的 **nextResponseBodyConverter** 方法关键
```Java
  public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    。。。
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
    。。。
```
跟 **createCallAdapter** 思路一样，找到第一个就返回了。
值得注意的是，人家并不是返回 **callAdapterFactories** 和 **converterFactories**中的对象，看
```Java
CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
```
```Java
Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
```
**adapter** 是 **callAdapterFactories** 中对象的 **get** 返回的
**converter** 是 **converterFactories** 中对象的 **responseBodyConverter** 返回的
假设我们添加的是 **RxJava2CallAdapterFactory** 和 **GsonConverterFactory**
那么 **adapter** 就是 **RxJava2CallAdapter** 对象
**converter** 就是 **GsonResponseBodyConverter** 对象

这时我们在看 **HttpServiceMethod** 对象里面保存了什么做个总结
```Java
  // 在步骤2中RequestFactory对象
  private final RequestFactory requestFactory;
  // 在创建 Retrofit.Builder 时所指定的 client
  private final okhttp3.Call.Factory callFactory;
  // 假设在创建 Retrofit.Builder 时    addCallAdapterFactory(RxJava2CallAdapterFactory.create())
  // 那么这个对象就是 RxJava2CallAdapter
  private final CallAdapter<ResponseT, ReturnT> callAdapter;
  // 假设在创建 Retrofit.Builder 时    addConverterFactory(GsonConverterFactory.create())
  // 那么这个对象就是 GsonResponseBodyConverter
  private final Converter<ResponseBody, ResponseT> responseConverter;

  private HttpServiceMethod(RequestFactory requestFactory, okhttp3.Call.Factory callFactory,
      CallAdapter<ResponseT, ReturnT> callAdapter,
      Converter<ResponseBody, ResponseT> responseConverter) {
    this.requestFactory = requestFactory;
    this.callFactory = callFactory;
    this.callAdapter = callAdapter;
    this.responseConverter = responseConverter;
  }

  @Override ReturnT invoke(Object[] args) {
    return callAdapter.adapt(
        new OkHttpCall<>(requestFactory, args, callFactory, responseConverter));
  }
```
在回到 **Retrofit** 的 **create** 部分的动态代理最后，我们看到最后还调用了 **invoke** 方法，也就是上面 **HttpServiceMethod** 对象的 **invoke** 方法。
所以我们直接看 **RxJava2CallAdapter** 对象的 **adapt** 方法
```Java
  @Override public Object adapt(Call<R> call) {
    // 因为我们是调用 RxJava2CallAdapterFactory.create() 所创建的，所以这里的这个 isAsync 是false
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

    Observable<?> observable;
    // isResult 和 isBody 的值是 RxJava2CallAdapterFactory 的 get 方法中决定的，我们默认用 OkHttpClient，这里 isResult 为false， isBody 为 true
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } else {
      observable = responseObservable;
    }

    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }
    // 我们就默认客户用 Observable 接受，那么下面都不用看了，都是 false
    if (isFlowable) {
      return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
      return observable.singleOrError();
    }
    if (isMaybe) {
      return observable.singleElement();
    }
    if (isCompletable) {
      return observable.ignoreElements();
    }
    return RxJavaPlugins.onAssembly(observable);
  }
```
所以最终返回都是 **BodyObservable** 对象，人家的构造接受一个对象，是 **CallExecuteObservable**， 剧透一下，访问网络的功能是写在这个对象里面的。
至此，3-6的步骤也好了