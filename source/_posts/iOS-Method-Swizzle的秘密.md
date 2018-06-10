---
title: iOS Method Swizzle的秘密
date: 2018-05-07 20:28:09
categories: iOS
tags: iOS
---

## 什么是Method Swizzling
方法交换(Method Swizzling)，顾名思义就是将两个方法的实现交换，即由原来的SEL(A)－IMP(A)、SEL(B)－IMP(B)对应关系变成了SEL(A)－IMP(B)、SEL(B)－IMP(A)，如下图：

![](方法替换Befor和After.png)

## Method类型
Method类型是一个`objc_method`结构体指针，而结构体`objc_method`有三个成员，方法交换(Method Swizzling)的本质就是更改两个成员`method_types`和`method_imp`

[runtime.h源码](https://opensource.apple.com/source/objc4/objc4-437/runtime/runtime.h)

```objc
/// An opaque type that represents a method in a class definition.
typedef struct objc_method *Method; // 本质是一个结构体

struct objc_method {
    SEL method_name;        // 方法名称
    char *method_types;    // 参数和返回类型的描述字串
    IMP method_imp;         // 方法的具体的实现的指针
}
```

## Method Swizzling 实现方式
比如我们有一个控制器`ParentViewController`继承于`UIViewController`，子控制器`SubViewController`继承于`ParentViewController`。我们想替换`SubViewController`的`viewDidAppear`为`swizzle_viewDidAppear`, 运行后先显示ParentViewController页面，然后点击一个Button按钮，push到SubViewController页面，代码如下：（页面都是通过StoryBoard来构建的，请自行构建，我比较懒，这里就只贴上代码）

```objc

@interface ParentViewController : UIViewController
@end

@implementation ParentViewController
- (void)viewDidAppear:(BOOL)animated{
    NSLog(@"%@ %s (IMP = ParentViewController viewDidAppear)",self, _cmd);
    [super viewDidAppear:animated];
}
@end


@interface SubViewController : ParentViewController
@end

@implementation SubViewController
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        // 原方法名和替换方法名
        SEL originalSelector = @selector(viewDidAppear:);
        SEL swizzledSelector = @selector(swizzle_viewDidAppear:);
        
        // 原方法结构体和替换方法结构体
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        // 如果当前类没有原方法的实现IMP，先调用class_addMethod来给原方法添加默认的方法实现IMP
        BOOL didAddMethod = class_addMethod(class,
                                            originalSelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));
        
        if (didAddMethod) {// 添加方法实现IMP成功后，修改替换方法结构体内的方法实现IMP和方法类型编码TypeEncoding
            class_replaceMethod(class,
                                swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else { // 添加失败，调用交互两个方法的实现
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)swizzle_viewDidAppear:(BOOL)animated {
    NSLog(@"%@ %s (IMP = SubViewController swizzle_viewDidAppear)",self, _cmd);
    [self swizzle_viewDidAppear:animated];
}
@end

```

**代码说明：**

`dispatch_once` 保证方法替换只被执行一次

为什么要先调用类添加方法`class_addMethod`，然后判断添加失败后，再调用方法交换实现方法`method_exchangeImplementations`？

上面代码中`SubViewController`是没有Override父类的`viewDidAppear`。如果我们直接调用`method_exchangeImplementations`会怎么样？ 在这种情况下，我们试试


```objc
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        // 原方法名和替换方法名
        SEL originalSelector = @selector(viewDidAppear:);
        SEL swizzledSelector = @selector(swizzle_viewDidAppear:);
        
        // 原方法结构体和替换方法结构体
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        // 调用交互两个方法的实现
        method_exchangeImplementations(originalMethod, swizzledMethod);
    });
}
```
改成上面的代码，然后运行。哎哟，出错了！Let me see see 什么情况

```objc
2018-05-07 21:38:56.615884+0800 TestiOS[3469:385923] <ParentViewController: 0x7f8f38c09aa0> viewDidAppear: (IMP = SubViewController swizzle_viewDidAppear)
2018-05-07 21:38:56.616189+0800 TestiOS[3469:385923] -[ParentViewController swizzle_viewDidAppear:]: unrecognized selector sent to instance 0x7f8f38c09aa0
2018-05-07 21:38:56.621157+0800 TestiOS[3469:385923] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[ParentViewController swizzle_viewDidAppear:]: unrecognized selector sent to instance 0x7f8f38c09aa0'
*** First throw call stack:
(
	0   CoreFoundation                      0x000000010e0fd1e6 __exceptionPreprocess + 294
	1   libobjc.A.dylib                     0x000000010d792031 objc_exception_throw + 48
	2   CoreFoundation                      0x000000010e17e784 -[NSObject(NSObject) doesNotRecognizeSelector:] + 132
	3   UIKit                               0x000000010e7a873b -[UIResponder doesNotRecognizeSelector:] + 295
	4   CoreFoundation                      0x000000010e07f898 ___forwarding___ + 1432
	5   CoreFoundation                      0x000000010e07f278 _CF_forwarding_prep_0 + 120
	6   TestiOS                             0x000000010ce903fb -[SubViewController swizzle_viewDidAppear:] + 75
	7   UIKit                               0x000000010e723ebf -[UIViewController _setViewAppearState:isAnimating:] + 697
	8   UIKit                               0x000000010e75ac53 -[UINavigationController viewDidAppear:] + 187
	9   UIKit                               0x000000010e723ebf -[UIViewController _setViewAppearState:isAnimating:] + 697
	10  UIKit                               0x000000010e726cfb __64-[UIViewController viewDidMoveToWindow:shouldAppearOrDisappear:]_block_invoke + 42
	11  UIKit                               0x000000010e72503f -[UIViewController _executeAfterAppearanceBlock] + 78
	12  UIKit                               0x000000010e58564f _runAfterCACommitDeferredBlocks + 634
	13  UIKit                               0x000000010e57477e _cleanUpAfterCAFlushAndRunDeferredBlocks + 388
	14  UIKit                               0x000000010e5942d7 __34-[UIApplication _firstCommitBlock]_block_invoke_2 + 155
	15  CoreFoundation                      0x000000010e09fb0c __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__ + 12
	16  CoreFoundation                      0x000000010e0842db __CFRunLoopDoBlocks + 331
	17  CoreFoundation                      0x000000010e083a84 __CFRunLoopRun + 1284
	18  CoreFoundation                      0x000000010e08330b CFRunLoopRunSpecific + 635
	19  GraphicsServices                    0x0000000113267a73 GSEventRunModal + 62
	20  UIKit                               0x000000010e57a0b7 UIApplicationMain + 159
	21  TestiOS                             0x000000010ce9047f main + 111
	22  libdyld.dylib                       0x0000000111b56955 start + 1
)
```

控制台打印`ParentViewController`找不到`swizzle_viewDidAppear`方法。 这是为什么呢？

第一行日志显示，`ParentViewController`是调用自己的方法`viewDidAppear`，通过打印_cmd输出为`viewDidAppear`可知道，但是执行的是`SubViewController`的`swizzle_viewDidAppear`的方法实现IMP，在`swizzle_viewDidAppear`方法实现IMP里又调用了`[self swizzle_viewDidAppear:animated]`这行代码，但是此时self是`ParentViewController`实例对象，类方法列表里根本没有swizzle_viewDidAppear方法，所以就导致找不到方法错误。说的我都觉得挺绕口的，千言万语不如一张图来的直观，向下瞅:

![](子类方法替换1.png)

现在明白了吧，方法交换（Method Swizzling）在子类没有实现`viewDidAppear`方法的情况下会交换父类的`viewDidAppear`的实现IMP，所以在`swizzle_viewDidAppear`实现IMP中调用`swizzle_viewDidAppear`方法会触发`doesNotRecognizeSelector`找不到方法错误

---

如果我们的子类`SubViewController`重写Override了父类的`viewDidAppear`方法会怎么样？
我们在`SubViewController`中重写`viewDidAppear`方法

```objc
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        // 原方法名和替换方法名
        SEL originalSelector = @selector(viewDidAppear:);
        SEL swizzledSelector = @selector(swizzle_viewDidAppear:);
        
        // 原方法结构体和替换方法结构体
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        // 调用交互两个方法的实现
        method_exchangeImplementations(originalMethod, swizzledMethod);
    });
}

- (void)viewDidAppear:(BOOL)animated{
    NSLog(@"%@ %s (IMP = SubViewController viewDidAppear)",self, _cmd);
    [super viewDidAppear:animated];
}

- (void)swizzle_viewDidAppear:(BOOL)animated {
    NSLog(@"%@ %s (IMP = SubViewController swizzle_viewDidAppear)",self, _cmd);
    [self swizzle_viewDidAppear:animated];
}

```

改成上面的代码，然后运行。这次竟然成功了

```objc
2018-05-07 21:41:58.467190+0800 TestiOS[3556:400520] <ParentViewController: 0x7fd5a060b730> viewWillAppear
2018-05-07 21:41:58.475185+0800 TestiOS[3556:400520] <ParentViewController: 0x7fd5a060b730> viewDidAppear: (IMP = ParentViewController viewDidAppear)
2018-05-07 21:42:08.772307+0800 TestiOS[3556:400520] <SubViewController: 0x7fd5a0405170> viewWillAppear
2018-05-07 21:42:09.312426+0800 TestiOS[3556:400520] <SubViewController: 0x7fd5a0405170> viewDidAppear: (IMP = SubViewController swizzle_viewDidAppear)
2018-05-07 21:42:09.312643+0800 TestiOS[3556:400520] <SubViewController: 0x7fd5a0405170> swizzle_viewDidAppear: (IMP = SubViewController viewDidAppear)
2018-05-07 21:42:09.312792+0800 TestiOS[3556:400520] <SubViewController: 0x7fd5a0405170> viewDidAppear: (IMP = ParentViewController viewDidAppear)
```

你逗我玩儿呢，怎么子类实现`viewDidAppear`就好了。那是因为子类在检查到自己有`viewDidAppear`方法就直接交换自己的`viewDidAppear`方法实现IMP，直接上图，我不想废话了
![](子类方法替换2.png)

这下我们明白了，如果直接调用方法`method_exchangeImplementations`来交换方法，需要考虑到子类有没有相应的方法，如果没有就要特殊处理，那岂不是太麻烦了。哈哈😁不用你瞎操心，苹果有这个方法`class_addMethod`来帮助我们解决

---

我们把直接调用`method_exchangeImplementations`稍微做点修改

```objc
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        // 原方法名和替换方法名
        SEL originalSelector = @selector(viewDidAppear:);
        SEL swizzledSelector = @selector(swizzle_viewDidAppear:);
        
        // 原方法结构体和替换方法结构体
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        // 如果当前类没有原方法的实现IMP，先调用class_addMethod来给原方法添加默认的方法实现IMP
        BOOL didAddMethod = class_addMethod(class,
                                            originalSelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));
        
        if (didAddMethod) {// 添加方法实现IMP成功后，修改替换方法结构体内的方法实现IMP和方法类型编码TypeEncoding
            class_replaceMethod(class,
                                swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else { // 添加失败，调用交互两个方法的实现
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}
```

这样我们就不用关心子类有没有实现`viewDidAppear`方法，方法`class_addMethod`在子类没有实现`viewDidAppear`方法的时候，为其添加`swizzle_viewDidAppear`方法实现IMP，原方法`swizzle_viewDidAppear`指向父类的`viewDidAppear`方法实现IMP

![](子类方法替换3.png)

这就是为什么先要调用`class_addMethod`方法的原因了，如果子类`SubViewController`实现了方法`viewDidAppear`，那么`class_addMethod`方法会返回NO，意思子类存在`viewDidAppear`方法实现，就直接走`method_exchangeImplementations`方法交换

好了，就说到这里吧，有问题请留言

## 参考：
* [Method Swizzling](http://nshipster.cn/method-swizzling/)

