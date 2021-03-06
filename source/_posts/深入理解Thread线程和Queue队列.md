---
title: 深入理解Thread线程和Queue队列
date: 2018-05-09 08:42:44
categories: iOS
tags: iOS
---

## 思考一段代码

我们先来看一段代码，猜猜一下代码的的运行结果：

```objc
    // 主队列
    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    // 给主队列设置一个标记
    dispatch_queue_set_specific(mainQueue, "key", "main", NULL);

    // 定义一个block任务
    dispatch_block_t log = ^{
        // 判断是否是主线程
        NSLog(@"main thread: %d", [NSThread isMainThread]);
        // 判断是否是主队列
        void *value = dispatch_get_specific("key");
        NSLog(@"main queue: %d", value != NULL);
    };

    // 全局队列
    dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);
    // 异步加入全局队列里
    dispatch_async(globalQueue, ^{
        // 异步加入主队列里
        dispatch_async(dispatch_get_main_queue(), log);
    });

    NSLog(@"before dispatch_main");
    dispatch_main();
    NSLog(@"after dispatch_main");
```

运行结果：

```objc
2018-05-08 15:08:05.557398+0800 TestRunLoop[28206:767410] before dispatch_main
2018-05-08 15:08:05.557682+0800 TestRunLoop[28206:767462] main thread: 0  //不是主线程
2018-05-08 15:08:05.557814+0800 TestRunLoop[28206:767462] main queue: 1 //是主队列
```

什么情况？派发给主队列的任务不是在主线程上运行，跟我们平常用的和理解的完全不一样。

不要激动，导致这种原因最关键的是这行代码`dispatch_main()` ,就是`这货`让主队列的任务在非主线程运行。

这个方法苹果官方文档这样解释的：

```objc
/*!
 * @function dispatch_main
 *
 * @abstract
 * Execute blocks submitted to the main queue.
 * 执行提交给主队列的任务blocks
 *
 * @discussion
 * This function "parks" the main thread and waits for blocks to be submitted
 * 
 * to the main queue. This function never returns.
 * 这个函数会阻塞主线程并且等待提交给主队列的任务blocks完成，这个函数永远不会返回
 * 
 * Applications that call NSApplicationMain() or CFRunLoopRun() on the
 * main thread do not need to call dispatch_main().
 *
 */
API_AVAILABLE(macos(10.6), ios(4.0))
DISPATCH_EXPORT DISPATCH_NOTHROW DISPATCH_NORETURN
void
dispatch_main(void);
```

意思是这个方法会阻塞主线程，然后在其它线程中执行主队列中的任务，这个方法永远不会返回（意思会卡住主线程）

如果去掉`dispatch_main()`这行代码，就会正常在主线程里执行任务

```objc
2018-05-08 15:20:16.358742+0800 TestRunLoop[28367:779939] main thread: 1 //主线程
2018-05-08 15:20:16.359066+0800 TestRunLoop[28367:779939] main queue: 1 //主队列
```

所以在主队列的任务通常是在主线程里执行，但是不一定，我们可以主动去执行被添加到主队列MainQueue的任务task（也就是说我们可以主动来调用添加到主线程队列的blocks）。可以使用以下任一个来实现：`dispatch_main()`、`UIApplicationMain()` 、`CFRunLoopRun()`

---

那我们再思考一下，主线程是否可以运行非主队列的任务blocks吗？答案是可以的，比如下面的代码：

```objc
    // 同步加入全局队列里
    dispatch_sync(globalQueue, ^{
        // 判断是否是主线程
        NSLog(@"main thread: %d", [NSThread isMainThread]);
        // 判断是否是主队列
        void *value = dispatch_get_specific("key");
        NSLog(@"main queue: %d", value != NULL);
    });
```

执行结果：

```objc
2018-05-08 15:27:31.215279+0800 TestRunLoop[28442:785851] main thread: 1 // 主线程
2018-05-08 15:27:33.519456+0800 TestRunLoop[28442:785851] main queue: 0 // 全局队列
```

所以通过`dispatch_sync()`执行的block不会开辟新的线程，而是在当前的线程（即主线程）中同步执行block

## runloop和queue的区别
* runloop和queue的区别

`runloop`和`queue`各自维护着自己的一个任务队列，在`runloop`的每个周期里面，会检测自身的任务队列里面是否存在待执行的`task`并且执行。但主线程的情况比较特殊，在`main runloop`的每个周期，会去检测`main queue`是否存在待执行任务，如果存在，那么`copy`到自身的任务队列中执行

* async的实现不同

在非主线程之外，`runloop`和`queue`的任务队列是互不干扰的，因此两者处理任务的机制也是完全不同的。当`async`任务到队列时，`GCD`会尝试寻找一个线程来执行任务。由于串行队列同时只能与一个线程挂钩，因此`GCD`会让该线程执行完已有任务后，才执行`async`到队列中的任务。

## 多线程的实现有以下几种方式
|    线程类型    | 简介 | 语言 | 线程生命周期 | 使用频率 |
| ---------- | --- | --- | --- | --- |
| pthread | * 一套通用的多线程API </br> * 适用于Unix\Linux\Windows\OSX等系统 </br> * 跨平台\可移植 </br> * 使用难度大| C | 程序员管理 | 几乎不用 |
| NSThread | * 使用更加面向对象 </br> * 简单易用 </br> * 可直接操作线程对象| OC | 程序员管理 | 偶尔使用 |
| GCD | * 旨在替代NSThread等线程技术 </br> * 充分利用设备的多核| C | 自动管理 | 经常使用 |
| NSOperation | * 基于GCD（底层是GCD） </br> * 比GCD多了一些更简单实用的功能 </br> * 使用更加面向对象| OC | 自动管理 | 经常使用 |


## 串行与并发
|    类型    | 全局并发队列 | 手动创建串行队列 | 主队列 |
| ---------- | --- | --- | --- | --- |
| 同步(sync) | * `没有`开启新线程 <br> * `串行`执行任务 | * `没有`开启新线程 <br> * `串行`执行任务  | * `没有`开启新线程 <br> * `串行`执行任务  |
| 异步(async) | * `有`开启新线程 <br> * `并发`执行任务 | * `有`开启新线程 <br> * `串行`执行任务  | * `没有`开启新线程 <br> * `串行`执行任务  |

## 队列和线程
![](GCD-POOL.png)

## 参考：

* [深入理解GCD](https://bestswifter.com/deep-gcd/)
* [被遗弃的线程](https://juejin.im/post/5ad17544f265da23793c9606)
* [奇怪的GCD](http://sindrilin.com/note/2018/03/03/weird_thread.html)
* [iOS 多线程详解](https://juejin.im/entry/58aacac08ac247006e625b8c)
* [玩转GCD](https://github.com/ShowJoy-com/showjoy-blog/issues/14)