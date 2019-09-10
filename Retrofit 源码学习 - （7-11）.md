经过 1-6 的步骤，假设用户都是用 **Observable** 进行接收，那么用户得到的其实都是 **BodyObservable** 对象，当我们执行 **subscribe** 后面又发生了什么呢？
我们看看 **BodyObservable** 代码
```Java
  BodyObservable(Observable<Response<T>> upstream) {
    this.upstream = upstream;
  }

  @Override protected void subscribeActual(Observer<? super T> observer) {
    upstream.subscribe(new BodyObserver<T>(observer));
  }
```
我们通过之前得知，这里从构造方法传入的对象 **upstream** 是 **CallExecuteObservable** ，**subscribe** 操作的也是这个 **CallExecuteObservable**， 所以我们直接看这个对象的 **subscribeActual** 方法的关键代码
```Java
  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    。。。
    try {
      Response<T> response = call.execute();
      if (!disposable.isDisposed()) {
        observer.onNext(response);
      }
      if (!disposable.isDisposed()) {
        terminated = true;
        observer.onComplete();
      }
    } catch (Throwable t) {
      Exceptions.throwIfFatal(t);
      if (terminated) {
        RxJavaPlugins.onError(t);
      } else if (!disposable.isDisposed()) {
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
  }
```
其中这个 **call** 对象是 **HttpServiceMethod** 的 **invoke** 方法中所创建的
```Java
  @Override ReturnT invoke(Object[] args) {
    return callAdapter.adapt(
        new OkHttpCall<>(requestFactory, args, callFactory, responseConverter));
  }
```
所以我们看 **OkHttpCall** 的 **execute** 方法关键代码
```Java
  @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;
      。。。
      call = rawCall;
      if (call == null) {
        try {
          // 这是是创建call的重要代码
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException | Error e) {
          throwIfFatal(e); //  Do not assign a fatal error to creationFailure.
          creationFailure = e;
          throw e;
        }
      }
    }
    。。。
  }
```
继续看是如何创建一个 **call** 对象的，即 **createRawCall()**
```Java
  private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    。。。
    return call;
  }
```
重点来了，还记得步骤2中的 **parameterHandlers** 吗，每一个网络访问接口的数据都是抽象成一个 **RequestBuilder** 对象，而 **ParameterHandler** 则表示如果处理 **RequestBuilder** 属性的一个抽象，我们会在 **create** 方法中进行创建 **RequestBuilder** 对象。
我们进到 **create** 方法中去看关键代码
```Java
  okhttp3.Request create(Object[] args) throws IOException {
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;
    。。。
    // 这里只根据方法注释的信息，创建了一个 RequestBuilder 对象
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl,
        headers, contentType, hasBody, isFormEncoded, isMultipart);

    List<Object> argumentList = new ArrayList<>(argumentCount);
    for (int p = 0; p < argumentCount; p++) {
      argumentList.add(args[p]);
      // 遍历了每个参数，将参数和 RequestBuilder 对象传递给 ParameterHandler 对象进行处理
      handlers[p].apply(requestBuilder, args[p]);
    }

    // 最终 requestBuilder 将会是一个完整的访问网络接口的抽象，在这里将其装换成了 OkHttp的 Request 对象
    return requestBuilder.get()
        .tag(Invocation.class, new Invocation(method, argumentList))
        .build();
  }
```
我们在回到 **OkHttpCall** 的 **execute** 方法的最后面
```Java
    return parseResponse(call.execute());
```
调用 `call.execute()` 同步返回了请求结果，然后又传递到了 **parseResponse** 方法中去，在这个里面就做了数据模型的转化，点进去看看关键代码
```Java
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
    。。。
    try {
      T body = responseConverter.convert(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```
在之前我们就知道了， **responseConverter** 对象是 **GsonRequestBodyConverter**，而其中的 **convert** 方法就是调用了 **Gson** 框架进行模型的装换
```Java
  @Override public RequestBody convert(T value) throws IOException {
    Buffer buffer = new Buffer();
    Writer writer = new OutputStreamWriter(buffer.outputStream(), UTF_8);
    JsonWriter jsonWriter = gson.newJsonWriter(writer);
    adapter.write(jsonWriter, value);
    jsonWriter.close();
    return RequestBody.create(MEDIA_TYPE, buffer.readByteString());
  }
```
这时，我们再回到 **CallExecuteObservable** 的 **subscribeActual** 方法，没什么问题的话，我们就该执行
```Java
observer.onNext(response);
```
但是有一个疑问，这个 **response** 的数据模型是 **Response<网络接口返回模型>** ,我们的观察者是不需要这层 **Response** 的，而且实际使用也是没有的，这是怎么回事？
其实这个 **observer** 可不是用户的 **observer**，我们再看 **BodyObservable** 的 **subscribeActual**
```Java
  @Override protected void subscribeActual(Observer<? super T> observer) {
    upstream.subscribe(new BodyObserver<T>(observer));
  }
```
我们得知，这个 **observer** 是 **BodyObserver** 对象，用户的 **observer** 也被保存在了这个 **BodyObserver** 对象中，我看看 **BodyObserver** 的 **onNext** 方法
```Java
    @Override public void onNext(Response<R> response) {
      if (response.isSuccessful()) {
        observer.onNext(response.body());
      } else {
        terminated = true;
        Throwable t = new HttpException(response);
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
```
哦~，一切的明了了，最终 **BodyObserver** 会进行解包，手动调用客户的 **observer** 的 **onNext** ，并将之前的 **Response<网络接口返回模型>.body** 取出真正的 **网络接口返回模型**。

到此，整个流程我们都走完了，源码版本 2.5.0。