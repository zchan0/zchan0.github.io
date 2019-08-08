---
layout: post
title: Implement Singleton with Objective-C
date: 2018-09-16 13:41 +0800
---

* TOC
{:toc}
# 单例模式的 Objective-C 实现

使用单例的原因是，想要在**全局**增加一个类的实例所提供的资源的访问点。这里的全局的含义，也说明了从内存上来看，单例是存在于整个程序/app 的生命周期，程序结束时才会被释放。

所以需要做到的事情是：**保证所访问的实例即是类唯一的实例**。

Objective-C 的实现方式和 GoF 用 C++ 实现的方式类似，也是使用 `static` 变量：

```c++
+ (instance)sharedInstance {
    static ClassA * instance = nil;
    if (instance == nil) {
        instance = [[self alloc] init];
        // setup
    }
    return instance;
}	
```

如果要保证线程安全（单例也是共享资源），则可以在 intance 的 `nil` 检查周围加上 `@synchronized` block 或者 NSLock 实例。

```objective-c
+ (instance)sharedInstance {
    static ClassA * instance = nil;
    @synchronized(self) {
        if (instance == nil) {
            instance = [[self alloc] init];
            // setup
        }
    }
    return instance;
}
```

## @synchronized 与锁

简单讲的话，可以把 `@synchronized` 看作语法糖，对于 synchronized 的变量，编译器会在 block 将加锁/解锁等事情做好。

maybe transformed the following:

```objective-c
- (NSString *)myString {
    @synchronized(self) {
      return [[myString retain] autorelease];
    }
}
```

to:

```objective-c
- (NSString *)myString {
    NSString *retval = nil;
    pthread_mutex_t *self_mutex = LOOK_UP_MUTEX(self);
    pthread_mutex_lock(self_mutex);
    retval = [[myString retain] autorelease];
    pthread_mutex_unlock(self_mutex);
    return retval;
}
```

从而保证在 `@synchronized` 的 block 里，能有序使用资源。

# 内存泄漏问题

单例虽然看上去带来了很多便利，但是网上也有很多比较反对使用。我觉得倒并不是模式本身的错，而是使用方式的问题。因为单例生成后会存在于 app 的整个生命周期中，单例所持有的属性、如果是 `strong` 的话，那么可能会有内存泄漏的风险。
因为 `strong`，单例会持有其属性变量，在访问完成局部的（即其生命周期不应该是整个 app 的）时，其持有的变量本来应该释放掉，却仍然存在于内存中，于是造成了泄漏。

解决这个问题，可以使用 `weak` 来修饰属性。使用 `weak` 修饰变量，单例并不会持有这个变量，在变量的生命周期结束时，其会被置为 `nil`。

# 弱单例（Weak Singleton）

[这篇](https://www.ios-blog.co.uk/tutorials/objective-c-ios-weak-singletons/)文章介绍了一种 weak 单例模式。这种方式，可以改变传统的单例存在于整个 app 的生命周期的情况，当没有对象持有单例时，它就会被释放掉；下一次访问的时候，它又会新建一个新的实例。

使用这种方式要注意两点：

- 其实这种 weak 单例模式是无状态的（因为它会被释放
- 使用的时候，需要一个对象来持有这个单例

# 弱单例模式实践

使用场景：

> 开发某个功能时，由于 Web 实现的页面和 Native 的页面间会不断来回跳转，需要考虑清理当前 navigation controller 的栈（避免形成环、并且能 push 和 pop 到产品指定的页面等），转场的动画，以及传递参数等。
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
