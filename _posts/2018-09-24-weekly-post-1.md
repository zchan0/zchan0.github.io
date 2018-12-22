---
layout: post
title: 一周小结 2：弱单例模式实践
date: 2018-09-24 11:01 +0800
---

* TOC
{:toc}
# 弱单例模式实践

这周尝试了[这篇笔记]({% post_url 2018-09-16-implement-singleton-with-objective-c %})里提到的弱单例模式。使用场景在[上周]({% post_url 2018-09-16-weekly-post-0 %})提到过：

> 这个需求的背景是，开发某个功能时，由于 Web 实现的页面和 Native 的页面间会不断来回跳转，需要考虑清理当前 navigation controller 的栈（避免形成环、并且能 push 和 pop 到产品指定的页面等），转场的动画，以及传递参数等。
>
> 对于单个页面来说如何跳转并不应该是它需要关心的事情，这时候需要一个集中式来管理页面交通的角色，自然地想到了 **Coordinator**。在 App 的生命周期中当然只应该有一个 Coordinator 的实例，因为每个页面也只会有一个实例，因此把它实现为**单例**。

之前使用 `weak` 修饰 base controller，解决了内存泄漏的问题。但是，单例的生命周期是和整个 App 的生命周期一致的，所以 Coordinator 并没有被释放掉。

这一周新加的需求需要 Coordinator 保存一些参数，以此来确定跳转的页面的样式、请求接口等。于是想到干脆尝试一下弱单例模式，这样就能安心地给单例加参数，而不用担心内存泄漏的问题。

定义一个弱单例很简单（来自[这篇](https://www.ios-blog.co.uk/tutorials/objective-c-ios-weak-singletons/)）：

```objective-c
+ (id)weakSharedInstance {
    static __weak ASingletonClass *instance;
    ASingletonClass *strongInstance = instance;
    @synchronized(self) {
        if (strongInstance == nil) {
            strongInstance = [[[self class] alloc] init];
            instance = strongInstance;
        }
    }
    return strongInstance;
}
```

使用的时候，需要一个 controller 来持有这个弱单例。如何选择这个 controller 呢？我觉得可以理解为和这个单例生命周期同步的那一个，比如之前提到的作为“局部小世界”入口的 base controller。

```objective-c
@interface ClassOfControllerA: UIViewController

@property (nonatomic, strong) ASingletonClass *coordinator;

@end

@implementation ClassOfControllerA

- (instancetype)init {
    self = [super init];
    if (self) {
        self.coordinator = [ASingletonClass weakSharedInstance];
        self.coordinator.baseController = self;
    }
    return self;
}

@end
```

Done!

# NSTimer 的使用

## Creating and Scheduing a Timer

有三种方式可以创建 timer（但貌似一般都用其中两种）：

- 使用 `scheduledTimerWithTimeInterval:invocation:repeats:` ，或者其它以 scheduledTimerXXX 开头的，这种默认是在当前 run loop 中执行，并且 mode 是 default 
- 使用 `timerWithTimeInterval`，或者其它以 timerWithXXX 开头的，这种并没有指定 timer schedule 的 run loop，必须使用 `[NSRunLoop addTimer:]` 将其添加到某个 run loop 中

## Stopping a Timer

使用 `invalidate` 就可以了。需要注意的是，在哪个 run loop 创建的 timer，就需要在哪个 run loop 中 invalide；一般还会将其置为 `nil` 。

## Memory Management

> Run loops maintain strong references to their timers, so you don’t have to maintain your own strong reference to a timer after you have added it to a run loop.

根据这段，看上去如果是使用 scheduledTimerXXX 开头创建的 timer，因为已经加入了当前的 run loop，所以已经被持有了。

> Because the run loop maintains the timer, from the perspective of memory management there’s typically no need to keep a reference to a timer after you’ve scheduled it. 
>
> Since the timer is passed as an argument when you specify its method as a selector, you can invalidate a repeating timer when appropriate within that method.

这种情况下，如果需要 stop a timer，可以在它们的 selector 或者 block 里，使 timer 停止。

> In many situations, however, you also want the option of invalidating the timer—perhaps even before it starts. In this case, you do need to keep a reference to the timer, so that you can send it an invalidate message whenever appropriate. 
>
> If you create an **unscheduled timer** (see “Unscheduled Timers”), then you must maintain a strong reference to the timer (in a reference-counted environment, you retain it) so that it is not deallocated before you use it.

但是在需要在某些特定时刻 invalidate timer 的场景，就必须要自己持有 timer 了。并且，如果是 unscheduled timer，必须用 strong 的方式持有 timer，这样才能在使用 timer 前保证它不被释放掉。

## 使用心得

- Timer 和 run loop 是紧密联系在一起的，所以需要注意创建和停止的方式
- 如果 timer 不起作用，看一下是不是 run loop 没放对，或者是 mode 设置不对
- 一般容易出错的地方就是滑动列表时添加的 timer，以及轮播图片中的 timer。特别是前者，因为默认放在当前 run loop，又是 default mode，所以在滑动的时候占用了主线程，timer 会失效。解决方式就是将 timer 放在别的 run loop 中，并且设置为 commonXXX 的 mode

## 参考

- [Using NSTimer](https://www.ios-blog.com/tutorials/objective-c/objective-c-using-nstimer/)
- [Run Loops](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)

# 踩到的坑

>  这次也不是 iOS 本身的坑，还是使用上容易出错的地方。

## 使用宏定义

在使用宏定义的时候，如果是用来做计算的，**一定要加上括号**。因为宏本质是代码直接替换，所以如果不加括号，很可能会出现运算顺序错误的问题。

# Tips

## iOS 中图片展示问题

- Xcode 中图片处理相关的技巧，[这篇](https://krakendev.io/blog/4-xcode-asset-catalog-secrets-you-need-to-know )里面提到了通过设置为 template 来改变 tintColor 以改变图片颜色，以及如何设置 slicing，来保证圆角。
- 图片一般需要裁剪，为了能更好地显示图片，可以根据设计给的比例（比如 16:9）来设定图片的约束，一般是定宽，然后根据 aspect ratio 来算高度，再通过设置 aspectfill mode 就能比较好的展示图片了。 

## 正确使用NS_DESIGNATED_INITIALIZER

这个使用场景是如果给类定义了新的初始化方法，希望使用者能使用这个而非原来的，一般会加 `NS_DESIGNATED_INITIALIZER` 这个宏。
但是，加上了这个宏之后，会出现一些 warning，详情见[这篇](https://blog.csdn.net/zcube/article/details/51657417)。所以更好的方式是直接使用 `NS_UNAVAILABLE` 来给 `init` 加 warning 就好。

更多的 Objective-C 的关键字或者更多宏，可以参考 [Adopting Modern Objective-C](https://developer.apple.com/library/archive/releasenotes/ObjectiveC/ModernizationObjC/AdoptingModernObjective-C/AdoptingModernObjective-C.html#//apple_ref/doc/uid/TP40014150)。

## Git 恢复删除的分支和 commit

```bash
git checkout -b <deleted branch name> <HEAD commit sha>
```

这里删除的 commit sha 可以通过 `git reflog` 来找。

参考：

- [Atlassian Notes](https://confluence.atlassian.com/bbkb/how-to-restore-a-deleted-branch-765757540.html)
- [StackOverflow](https://stackoverflow.com/questions/3640764/can-i-recover-a-branch-after-its-deletion-in-git)

# Weekly Read Posts

- [ReactiveCocoa Tutorial – The Definitive Introduction: Part 1/2](https://www.raywenderlich.com/2493-reactivecocoa-tutorial-the-definitive-introduction-part-1-2)，这周继续学习 RAC，鉴于之前的使用方式出现了问题，所以打算从头开始看。而这是一篇不错的入门文章（raywenderlich 家的文章其实都挺不错的，特别是入门来说）
- [Reusable data sources in Swift](https://www.swiftbysundell.com/posts/reusable-data-sources-in-swift)，这里的 reusable 主要是代码层面的复用。使用场景是，如果不同的 model 有类似的 UI 或者交互，比如 List 类型的，可以复用它们的 data source；而且利用这种类型的 data source，还可以简单的组合拼装 List（这一种使用方式很方便有趣）。