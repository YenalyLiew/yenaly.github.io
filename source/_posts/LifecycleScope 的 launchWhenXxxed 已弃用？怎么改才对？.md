---
title: LifecycleScope 的 launchWhenXxxed 已弃用？怎么改才对？
description: 部分图片显示不出来，需要的可以点击链接新开一个页面查看，因为有防盗链
---
# launchWhenX 给我弃用了，这可咋整？

> Android 别三日，当 deprecated 相待。

我当时写程序的时候有这种需求，希望某一段协程在**某一个生命周期开始**（下文假设生命周期为 onStart）进行，而且就**执行一次**。如果执行一次的话，单走一个`lifecycleScope.launch`其实也行，但并不安全。我去谷歌查阅了资料，又受到 IDE 提示的影响，了解到了“神奇”的`lifecycleScope.launchWhenStarted`，这个函数恰好满足我的需求。可是贴上去发现，弃用了！

![launchWhen1](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7a15f04750d4be1988b5644fb61a223~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

好家伙，让我去用`repeatOnLifecycle`？我心想这不对啊，我要执行一次，而这个函数的重点是`repeat`，还是不符合我要求。

我又转向了`LifecycleOwner::whenStarted`，该函数跟上面那个函数作用是一样的，只不过作用域不太一样。我想看看他是让你怎么做的。

![launchWhen2](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e8b0d495a234315bd77f58abae8bd24~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

好家伙，让我去用`withStarted`？但是`withStarted`里面只能放 non-suspending work（非挂起任务），这还是不符合我的要求啊。

其实如果你不怎么在乎警告，而且这种代码并没有给你的软件带来多大损失，你可以继续这么用着。但是在这个函数受众面这么广泛的情况下，为什么官方这么执意把它删除呢？

## repeatOnLifecycle 的问题

### 优势

我们可能都看过一张比较优势图：

![](https://miro.medium.com/v2/resize:fit:1400/0*pDKnvDJ9FzaCgXCd)

可以看到，`repeatOnLifecycle` 会在每次重复时都从头开始执行协程，并在生命周期状态降低到指定状态以下时取消。对于收集大多数流来说，它非常适合，因为在不需要时能完全取消流，从而节省了资源。

而 `launchWhenX` 系列函数不会取消协程并重新启动。它只会推迟启动时间，并在生命周期状态低于指定状态时挂起。

举一个简单的例子，

```kotlin
delay(5000)
Log.d("test", "test")
```

如果被`repeatOnLifecycle(Lifecycle.State.STARTED)`包裹起来，且 1s 后返回主界面，该挂起函数会立即**取消**，因为取消了，所以你再等 4s，也不会有任何反应。你返回程序再等 5s 后，log 信息会出现在控制台中，因为重新执行了。

如果被`launchWhenStarted`包裹起来，且 1s 后返回主界面，该挂起函数会立即**挂起**，因为挂起了，所以你再等 4s，也不会有任何反应，但是它还是一直在后台进行着 delay 的。当你过了 4s 多打开程序后，log 信息会立即出现在控制台中，因为挂起恢复了。

详细例子就不从这里细说了，可以参考一下这篇文章：[使用更为安全的方式收集 Android UI 数据流](https://zhuanlan.zhihu.com/p/389218426)

### 局限

我们从这篇文章也可以看到，这里面处理的是一个位置信息。位置信息，我们自然需要最新的，而且后台把该流取消掉，然后返回后重新创建也是可以接受的。

弃用`launchWhenX`方法，基本就是为了有效遏制挂起函数在后台还在运行，导致资源浪费甚至程序崩溃。

但是实际的需求是无穷无尽的，假如需要显示一个**一次性** SnackBar，或者打开一个**一次性** window，这些需求如果用`repeatOnLifecycle`，让我想起了`LiveData`当年的痛，有点类似处理粘性事件了，但又不太一样。

## 把 repeatOnLifecycle 用成 launchWhenX 的样子

官方在 [Alternative APIs for running one-time suspend code when lifecycle state is at least X?](https://issuetracker.google.com/issues/27004950) 中提供了三种解决方案，适用于不同需求

1. **让挂起的代码继续运行至结束**

   这样做的好处是整个代码块以原子方式运行，也就是说不会在中途被取消。

   ```kotlin
   lifecycleScope.launch {
     // 直到 STARTED 前一直挂起
     withStarted { }
     // 之前你写在 launchWhenStarted 中的代码写到这下面
     doYourOneTimeWork()
     // 注意：即使生命周期低于 STARTED 对应的 onStop，代码也将继续运行，
     // 这个和 launchWhenStarted 还不太一样！
   }
   ```

   这种类型的代码假设你的一次性任务不依赖于保持在某个生命周期状态以上，但这是一个库无法知道的情况。举个例子，如果在挂起任务之后运行了一个 FragmentTransaction，那么在该调用发生之前可能会保存状态。

2. **取消挂起的代码，并在返回到该状态以上时重新启动它**

   每次回到给定的生命周期状态以上时再重新运行代码块确实是 `repeatOnLifecycle` 的规定。然而，对于一个库来说，是无法知道是否可以安全地重新运行整个代码块，或者是否需要更频繁地进行检查点操作的。

   简单来说，如果可以多次运行整个代码块直到成功完成，那么只需要用一个 `isComplete` 来标记：

   ```kotlin
   lifecycleScope.launch {
     var isComplete = false
     repeatOnLifecycle(Lifecycle.State.STARTED) {
       if (!isComplete) {
         // 之前你写在 launchWhenStarted 中的代码写到这下面
         // 如果在运行时生命周期走向 STARTED 对应的 onStop 时会被直接取消
         doYourOneTimeWork()
         // 标记是否成功
         isComplete = true
       }
     }
   }
   ```

   当然不能保证如果中途被取消，重新运行该代码块就一定是安全的。

3. **取消挂起的代码，不重新启动**

   这个跟上一个类似，但是这个无论完没完成，都会在 finally 块中设置`isComplete`，让它彻底取消。

   ```kotlin
   lifecycleScope.launch {
     var isComplete = false
     repeatOnLifecycle(Lifecycle.State.STARTED) {
       if (!isComplete) {
         try {
           // 之前你写在 launchWhenStarted 中的代码写到这下面
           // 如果在运行时生命周期走向 STARTED 对应的 onStop 时会被直接取消
           doYourOneTimeWork()
         } finally {
           // 即使一次性任务运行了一半没运行完，也当它运行完了，
           // 并且在 finally 块中标记该任务已完成，所以该代码块不会重启。
           isComplete = true
         }
       }
     }
   }
   ```

   

## 五种方式怎么选？

> 我不太懂 PS，后面几个用画图软件 P 的，能看懂意思就行

首先遵循以下的 lifecycle 活动：

<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e7099f3934846d1be873b9559b80f2d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?" alt="View's lifecycle" style="zoom:25%;" />

### launchWhenStarted

> 不安全，且已被弃用，但可以后台加载，最重要的是**一次性**

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec1482ae9bc142d09115a180bd48616a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?" alt="launchWhenStarted" style="zoom: 25%;" />

### repeatOnLifecycle(Lifecycle.State.STARTED)

> 安全，但可能产生**粘性事件**

<img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4203c50abe324fa78f718bb709cbd967~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?" alt="repeat" style="zoom:25%;" />

### 解决方案1

> 不安全，谨慎使用
>
> 注意和`lifecycleScope.launch`的区别

<img src="https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d12ce31d67c04e598856374f2c39d121~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?" alt="solution1" style="zoom:25%;" />

### 解决方案2

> 安全，可以说是替代`launchWhenStarted`的较优方案。缺点是不能后台加载

<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93b58463a2674d77bb0c3713260f75ee~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?" alt="solution2" style="zoom:25%;" />

### 解决方案3

> 安全，但可能没处理完就结束了，不太适用于界面处理

<img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07db9d128dc84f3d9aebcb120937cd97~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?" alt="solution3" style="zoom:25%;" />