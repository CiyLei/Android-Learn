大致流程图：
![Retrofit.png](https://github.com/CiyLei/Android-Learn/blob/master/img/Retrofit.png?raw=true)

我们大致分为3个步骤
**（1-2）**：分析每个接口的方法和参数注解，并生成设置 **RequestBuilder** 对应的 **ParameterHandler** 处理类
**（3-6）**：根据返回值和 **CallAdapterFactory** 返回相应的对象
**（7-11）**： 生成 **RequestBuilder** 对象，用 **OkHttp** 访问网络并使用 **ConverterFactory** 进行转换对象回调。

接下来代码分析

**Retrofit** 采用 **Builder** 设计模式构建，**build** 完也只是设置了一个构造参数，所以直接看 **Retrofit** 的 **create** 方法
```Java
  public <T> T create(final Class<T> service) {
    // 首先这里做了验证，判断 service 是否是一个接口类，且这个接口类没有任何的父类接口，不符合要求直接异常
    Utils.validateServiceInterface(service);
    // 这个不知道干嘛的
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    // 这里直接返回的一个动态代理的对象
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // 如果方法是自Object基础过来的话（如toString），则调用自身的方法。
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            // 这里判断现在这个方法是不是default方法（Java8加的特性），android里没有实现相应的调用，这里就不做讨论
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 基本都是走到这里，会根据方法的各种注解信息调用OkHttp相应的方法进行请求
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
  }
```
**loadServiceMethod** 将相应的 **Method** 类型转化为 **ServiceMethod** 类型并缓存下来到 **serviceMethodCache** 里面
```Java
  ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        // 这里就是分析method并转化的方法
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
看 **parseAnnotations** 方法
```Java
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    // 这里面分析了所有的注解并保持在了RequestFactory对象里面
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    // 获取返回类型，并判断是不是不可分析，基本就是判断泛型的类型是不是都是确定的，不能是？或者没有指定具体类型的T
    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(method,
          "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    // 返回类型不可以是void
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }
    // 该进入2的步骤，生成了一个HttpServiceMethod对象
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```
这里主要的方法还是 **RequestFactory.parseAnnotations** ，他分析了一个方法的所有注解和方法里面所有的参数注解，进去看看
```Java
  static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
  }
```
```Java
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      // 获取这个方法上面所有的注解信息
      this.methodAnnotations = method.getAnnotations();
      // 获取这个方法里面所有参数的类型
      this.parameterTypes = method.getGenericParameterTypes();
      //  获取这个方法里面所有参数的所有注解，所以这个是一个二维数组
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
```
接下来的 **build** 方法有点多，一段一段看，首先他循环了 **methodAnnotations** ，即遍历了这个方法上面的所有注解，然后调用 **parseMethodAnnotation** 方法，现在我们只分析 **Get** 注解，所以删掉 **parseMethodAnnotation** 方法里面的其余代码
```Java
    RequestFactory build() {
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }
      。。。

```
```Java
    private void parseMethodAnnotation(Annotation annotation) {
      。。。
      if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } 
      。。。
    }
```
这了又调用了 **parseHttpMethodAndPath** 方法，分析了是否有在 **Get** 参数里面设置 **Path**
说明一下，这里的`((GET) annotation).value()` 会返回注解里面写的值，比如`@GET("test.do")` 就获取了`"test.do"` 
```Java
    private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      // 防止请求方式重复设置
      if (this.httpMethod != null) {
        throw methodError(method, "Only one HTTP method is allowed. Found: %s and %s.",
            this.httpMethod, httpMethod);
      }
      // 将这个值设置为 GET
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;

      if (value.isEmpty()) {
        return;
      }

      // 判断是否有Get参数
      int question = value.indexOf('?');
      if (question != -1 && question < value.length() - 1) {
        // 取出Get参数
        String queryParams = value.substring(question + 1);
        // 这里通过正则去匹配查看参数里面是否有{xxx}的情况，这种情况应该写在拼接url的时候，而不是写在参数里面
        Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
        if (queryParamMatcher.find()) {
          throw methodError(method, "URL query string \"%s\" must not have replace block. "
              + "For dynamic query parameters use @Query.", queryParams);
        }
      }

      this.relativeUrl = value;
      // 去匹配这个url的{xxx}情况，因为上面做了验证，此情况不可能出现在参数里面，只会在url中出现，所以都匹配出来保存在 relativeUrlParamNames 里面
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```
回到上面的 **build** 方法
```Java
      // 这个httpMethod上面设置成为Get
      if (httpMethod == null) {
        throw methodError(method, "HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }
      // 根据是否有 body 的情况判断是否有 Multipart FormEncoded 注解等情况
      if (!hasBody) {
        if (isMultipart) {
          throw methodError(method,
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError(method, "FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
        }
      }

      // 这里循环了这个方法的每个参数，开始分析参数注解了
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        parameterHandlers[p] = parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p]);
      }
      。。。
      return new RequestFactory(this);
    }
```
看 **parseParameter** 方法
```Java
    private ParameterHandler<?> parseParameter(
        int p, Type parameterType, @Nullable Annotation[] annotations) {
      ParameterHandler<?> result = null;
      if (annotations != null) {
        for (Annotation annotation : annotations) {
          // 通过分析参数的注解，生成 ParameterHandler 对象
          ParameterHandler<?> annotationAction =
              parseParameterAnnotation(p, parameterType, annotations, annotation);

          if (annotationAction == null) {
            continue;
          }
          // 通过这里我们知道，框架控制了一个参数最多只能对应一个 ParameterHandler 对象，否则报错
          if (result != null) {
            throw parameterError(method, p,
                "Multiple Retrofit annotations found, only one allowed.");
          }

          result = annotationAction;
        }
      }

      if (result == null) {
        throw parameterError(method, p, "No Retrofit annotation found.");
      }

      return result;
    }
```
最关键的还是 **parseParameterAnnotation**  方法，它枚举了每个参数所有可能用到的注解，一共12种有：**Url** 、**Path**、**Query**、**QueryName**、**QueryMap**、**Header**、**HeaderMap**、**Field**、**FieldMap**、**Part**、**PartMap**、**Body**。
然后一一分析返回对应的 **ParameterHandler** 对象
这里太多了，我们就分析一个 **Query** 吧
```Java
      } else if (annotation instanceof Query) {
        validateResolvableType(p, type);
        Query query = (Query) annotation;
        String name = query.value();
        boolean encoded = query.encoded();

        // 获取这个参数的类型
        Class<?> rawParameterType = Utils.getRawType(type);
        gotQuery = true;
        // 如果这个参数时一个迭代的类型的话
        if (Iterable.class.isAssignableFrom(rawParameterType)) {
          // 如果参数的类型有泛型的话，不允许出现不确定的类型，即？什么的
          if (!(type instanceof ParameterizedType)) {
            throw parameterError(method, p, rawParameterType.getSimpleName()
                + " must include generic type (e.g., "
                + rawParameterType.getSimpleName()
                + "<String>)");
          }
          ParameterizedType parameterizedType = (ParameterizedType) type;
          // 获取第一个泛型的类型
          Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
          // 调用 addConverterFactory 添加的 ConverterFactory 转化类，在之后生成 RequestBuilder 的时候，会通过这个类将任意参数类型转化为字符串。
          Converter<?, String> converter =
              retrofit.stringConverter(iterableType, annotations);
          // 注意，最后面调用了iterable()，他会迭代执行 ParameterHandler.Query 对象的 apply 方法，将参数全部分析出来，当然，这也是在生成 RequestBuilder 的时候才会执行，现在执行生成 ParameterHandler 对象
          return new ParameterHandler.Query<>(name, converter, encoded).iterable();
        } else if (rawParameterType.isArray()) {
          // 如果是数组类型的话，返回数组里面是什么类型，如果是基础类型的话，就返回它的封装类，比如 boolean 转 Boolean
          Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
          Converter<?, String> converter =
              retrofit.stringConverter(arrayComponentType, annotations);
          // 数组的话for循环执行 ParameterHandler.Query 的 apply
          return new ParameterHandler.Query<>(name, converter, encoded).array();
        } else {
          Converter<?, String> converter =
              retrofit.stringConverter(type, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded);
        }

      }
```
**ParameterHandler** 对象有什么作用，这里在详细说明一下。
我们访问一次接口所用到的所有数据都封装成了一个 **RequestBuilder** 对象，但是分析参数的时候生成的是一个 **ParameterHandler** 对象，它跟 **RequestBuilder** 有什么联系呢？
其实 **ParameterHandler** 对象，准确的来讲是 **ParameterHandler** 的 **apply** 方法是对怎么设置 **RequestBuilder** 参数的一个抽象。
打个比方，我们看 **ParameterHandler.RelativeUrl()** 的 **apply** 方法
```Java
  static final class RelativeUrl extends ParameterHandler<Object> {
    @Override void apply(RequestBuilder builder, @Nullable Object value) {
      checkNotNull(value, "@Url parameter is null.");
      builder.setRelativeUrl(value);
    }
  }
```
**RequestBuilder** 对象的 **relativeUrl** 是请求的 **url** ，这里进行了设置。也就是在未来的某个环节，我会调用这个方法设置一个 **RequestBuilder** 对象的 **relativeUrl** ，我们在看看 **ParameterHandler.Query** 的 **apply** 方法
```Java
    @Override void apply(RequestBuilder builder, @Nullable T value) throws IOException {
      if (value == null) return; // Skip null values.

      String queryValue = valueConverter.convert(value);
      if (queryValue == null) return; // Skip converted but null values

      builder.addQueryParam(name, queryValue, encoded);
    }
```
这个方法就是设置添加 **RequestBuilder** 了 **QueryParam**，这里保存了所有 **Get** 请求的参数信息。
剧透一下，在未来我们的被观察者对象订阅的时候就会生成一个 **RequestBuilder** 对象并同意调用所有 **ParameterHandler** 的 **apply** 方法了。

好了，到这里我们就基本分析完了（1-2）步骤，根据每个参数生成了对应的 **ParameterHandler** 对象。
