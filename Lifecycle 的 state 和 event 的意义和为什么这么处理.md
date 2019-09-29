![d2a08956897cb51fe6e2c9af8286c982.png](https://github.com/CiyLei/Android-Learn/blob/master/img/lifecycle1.png?raw=true)

### state 和 event 存在的意义

在我看源码之前我是这么猜想 `Lifecycle`  的大致原理的：
> 在触发生命周期的时候，就触发 `LifecycleObserver` 中相对应的方法（没错，想的就是这么的单纯）

但是这样做是无法做到这样一个需求的：
> 在 onStart 中 addObserver ，那在 LifecycleObserver 中就无法触发 onCreate 对应的方法了

所以说，为了解决这个问题，必须把每次 **生命周期的走向(event)** 和 **生命周期的状态(state)** 都保存下来
每个 `LifecycleObserver` 创建的时候的 `state` 都是 `INITIALIZED` ，之后会一步步走向当前生命周期的状态，之后生命周期发生了变化，也是通过 `event` 改变 `state`

### 为什么流程这么走？（在 RESUME 来了个回头杀）

在想明白上面的逻辑的时候，我脑子又脑补了一个 `Lifecycle`  的大致原理的：
> 既然需要把每次的走向和状态都保存下来，那么  `event` 和 `state` 的对应状态应该是如下
![c7adce42671e89fadf01c166eaa94bec.png](https://github.com/CiyLei/Android-Learn/blob/master/img/lifecycle2.png?raw=true)




然而我还是想的太年轻了，人家根本没有 `STOP` 和 `DESTROY` 的状态

这又是为什么？这一点我想了很久。。。
可以想象这么一个场景（假设在 LifecycleObserver 写了所有的生命周期对应的方法）
> 我们把 addObserver 的代码放到 onStop 的生命周期里面执行，那么我的思路和 Lifecycle 思路在执行 LifecycleObserver 里面对应方法时的区别是什么？
> * 我的思路
> `ON_CREATE` -> `ON_START` -> `ON_RESUME` -> `ON_PAUSE` -> `ON_STOP`
> * Lifecycle 的思路
> `ON_CREATE` 
> 因为执行了 `ON_CREATE` 后，状态是停在了 `CREATED` 上，所以只会执行 `ON_CREATE` 

我的思路就不做赘述了，Lifecycle 为什么会这么走呢？
因为 activity 的几个生命周期对应的几个实际意义的状态是
* RESUMED（onResume - onPause）：代表 activity 在前台并且可以交互
* STARTD（onStart - onStop）：代表 activity 可见但不可交互
* CREATED（onCreate - onDestory）：代表 activity 还没销毁
* DESTORYD：代表 activity 销毁

**假设执行了 onPause，那么 activity 就处于可见但不可交互的状态，那么从意义上来讲，跟执行 onStart 是一样的，因为执行了 onStart 也代表 activity 处于可见但不可交互的状态。**

所以 Lifecycle 是从本质上考虑问题，不知道 activity 的源码里面有没有类似于的这样的状态
> 但是有一说一，上面的情况是将 addObserver 放到了 onPause、onStop、onDestory 生命周期的情况下所出现的特殊情况，如果是放到 onCreate、onStart、onResume 生命周期里那是按照我的思路走的。
