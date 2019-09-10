##### 先理清类
`MarsServiceNative` 是 `Service`
`MarsServiceStub` 是 `Binder`
`MarsServiceProxy` 是 `ServiceConnection`

##### 大致流程

1. 初始化 `MarsServiceProxy` 
首先会在 `SampleApplicaton` 中初始化 `MarsServiceProxy` ，`MarsServiceProxy` 中会创建一个 `Worker`  类，是线程类，然后每隔 50 毫秒死循环执行 `continueProcessTaskWrappers` 方法。

2. 初始化 `MarsServiceNative` 和 `MarsServiceNative` 
`continueProcessTaskWrappers` 方法中会判断 `MarsServiceNative`  IM的服务是否启动，如果没有，则会通过 `bindService` 方法启动IM服务即 `MarsServiceNative` ，然后在 `MarsServiceNative`  启动成功会返回 `MarsServiceNative$Stub`  对象，在 `MarsServiceProxy`  的 `onServiceConnected` 中会将 `MarsServiceNative$Stub`  转为  `MarsServiceNative$Stub$Proxy`   保存为 `server`

3. 阻塞读取要发送的消息
如果IM服务已经启动，则阻塞读取 `queue` 

4. 注册推送回调
打开 `ConversationActivity` 后，创建一个 `MainService` 类，然后将 cmdid 以 key，`MainService` 为 value的形式保存到 `MarsServiceProxy` 类的 `pushMessageHandlerHashMap` 中，其中接收消息的回调值为 `PUSHMSG_CMDID` 即 10001

5. 将要发送消息添加到队列中
将消息包装在 `TextMessageTask` 中，然后调用 `MarsServiceProxy`  的 `send` ，将消息添加到 `MarsServiceProxy`  的 `queue`  中去

6. 触发 3 步骤的阻塞，准备发送消息
然后调用 `MarsServiceNative$Stub$Proxy`  的 `send` 方法，其中会将 `TextMessageTask`  转换为 `MarsTaskWrapper$Stub$Proxy`  ，然后触发 `MarsServiceNative` 类中的 `send` 方法

7. 封装 `StnLogic.Task`  类，然后发送消息
根据 `TextMessageTask`  中的各种信息，封装一个 `StnLogic.Task`  类，然后将这个类和 `MarsTaskWrapper$Stub$Proxy`  以 key/value 的形式保存在一个 `TASK_ID_TO_WRAPPER` 字段中，调用 `StnLogic.startTask()` 发送信息

8. 在回调中对 request 进行转换为 byte[] 
在发送数据之前，首先要对数据进行编码，因为 `MarsServiceStub` 实现了 `StnLogic.ICallBack`  接口，所以回调在 `MarsServiceStub`  的 `req2Buf` 中， `MarsTaskWrapper$Stub$Proxy`  从 `TASK_ID_TO_WRAPPER`  中取出，然后调用其的  `req2Buf` 方法进行编码，最终触发到 `NanoMarsTaskWrapper` 的 `req2buf` 方法，将要发送的数据进行编码返回

9. 触发回执信息的回调
发送信息成功后，服务器返回信息，然后触发  `MarsServiceStub`  的 `buf2Resp` 方法，可以在其中做是否发送成功的回调，这个同样从 `TASK_ID_TO_WRAPPER`  中取出`MarsTaskWrapper$Stub$Proxy`  ，然后执行  `buf2Resp` 方法，最终执行到 `NanoMarsTaskWrapper` 的 `buf2resp` 方法，在其中会执行 `onPostDecode` 方法，而 `TextMessageTask` 中重写了这个方法，所以在里面有做是否成功的回调。

##### 接收消息大致流程
1. 在 `MarsServiceProxy`  的 `onServiceConnected` 方法中，我们有将一个 `MarsPushMessageFilter` 对象传入到 `MarsServiceStub`  中，而在 `MarsServiceProxy`  的 `onPush` 中是回调接收信息的，要用到这个对象。
2. 所以直接看  `MarsServiceProxy`  的 `MarsPushMessageFilter` 对象
3. 在上面的 4 步骤中有将推送的回调保存在 `MarsServiceProxy` 类的 `pushMessageHandlerHashMap` 中，这个根据 `cmdId` 取出相应的回调，然后将执行 `MainService` 的 `process` 方法
 4. `MainService` 中也维护了一个线程，然后将消息进行分发，主要有2个，一个是 `MessageHandler` 和 `StatisticHandler` 。
 在 `MessageHandler`  中会发送一个广播，然后由 `ChatDataCore`  类进行接收，这个类是观察者模式，聊天界面的 `Activity` 已经注册进来了，所以直接更新聊天界面的 `adapter` 去刷新聊天界面。
 在 `StatisticHandler` 中则保存了历史的回执消息，流量等信息的保存。