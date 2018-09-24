---
layout: post
title: 周记-1
date: 2018-09-15 13:11 +0800
---

* TOC
{:toc}
# 写在最前面的

> 这周一刚好是回北京实习的第三个月。说起来也没特别去记这个时间，只是刚好在 Day One 中查看这周的笔记发现的，还真是蛮有仪式感的一个开始。

虽然我一直有日常记笔记的习惯，但是却很少把东西整理出来，除了在需要查阅的时候，也很少去回顾。最近反思自己的开发效率时，发现在开发中、实现了业务逻辑之后，会花费不少时间去解与设计稿/产品稿不相符合的地方。对我来说这样的问题主要出于对 iOS 和 Objective-C 语言的**准确**理解上。

我想到的解决方案是尽量把这些记录下来，积少成多，反复思考，每周整理一次。希望这个过程中能去探究一些开发中不曾注意或者没有时间仔细研究的问题，并且通过写出来的方式锻炼一下表达能力（这点上我真是太糟糕了）。

目前想到的整理的内容包括这周 Code Review 提到的点，踩过的坑，库和语言的使用方式整理，工具的使用 Tips，Weekly read posts 还有一些想法等。

（没有给这段想到一个好的 ending 那就直接开始吧 😅

# Code Review 提到的

## 单例，strong 与内存泄漏

这个需求的背景是，开发某个功能时，由于 Web 实现的页面和 Native 的页面间会不断来回跳转，需要考虑清理当前 navigation controller 的栈（避免形成环、并且能 push 和 pop 到产品指定的页面等），转场的动画，以及传递参数等。

对于单个页面来说如何跳转并不应该是它需要关心的事情，这时候需要一个集中式来管理页面交通的角色，自然地想到了 **Coordinator**。在 App 的生命周期中当然只应该有一个 Coordinator 的实例，因为每个页面也只会有一个实例，因此把它实现为**单例**。

为了 push 和 pop 页面时动画的一致性，我给这个单例加了一个 base controller 属性，并用了 `strong` 来修饰，于是问题来了。由于单例会被全局持有，而 base controller 又被单例持有，所以页面关闭之后，base controller 仍然存在，造成了**内存泄漏**。

解决方式比较简单，将其声明为 `weak` 就好了。更多关于 Objective-C 实现单例，另写了一篇[笔记]({% post_url 2018-09-16-implement-singleton-with-objective-c %})。

# 踩到的坑

## Objective-C 中的 BOOL 类型

>  严格来说也不能算作是坑，只是自己的孤陋寡闻罢了 = =

在解一个 iPhone 4s 上的 bug 的时候，发现同样是返回值为 BOOL 的方法，通过`method_getReturnType()` 拿到的 type 在 iphone 4s 上是 `c`，而在其它机型上则是 `B`，这就比较迷了。在 [iOS-深挖BOOL](https://www.jianshu.com/p/d23334e7cb35) 这篇里，才发现 BOOL 并不是一个真正的类型，而是由 `typedef` 定义的其它类型的**别名**而已。

根据那篇文章，我也在 `objc.h` 这个文件中，找到了定义 BOOL 的代码。

```c++
// iOS 11.4

/// Type to represent a boolean value.

#if defined(__OBJC_BOOL_IS_BOOL)
    // Honor __OBJC_BOOL_IS_BOOL when available.
#   if __OBJC_BOOL_IS_BOOL
#       define OBJC_BOOL_IS_BOOL 1
#   else
#       define OBJC_BOOL_IS_BOOL 0
#   endif
#else
    // __OBJC_BOOL_IS_BOOL not set.
#   if TARGET_OS_OSX || (TARGET_OS_IOS && !__LP64__ && !__ARM_ARCH_7K)
#      define OBJC_BOOL_IS_BOOL 0
#   else
#      define OBJC_BOOL_IS_BOOL 1
#   endif
#endif

#if OBJC_BOOL_IS_BOOL
    typedef bool BOOL;
#else
#   define OBJC_BOOL_IS_CHAR 1
    typedef signed char BOOL; 
    // BOOL is explicitly signed so @encode(BOOL) == "c" rather than "C" 
    // even if -funsigned-char is used.
#endif
```

可以看到，在 L14，在 macOS 系统，不是 64 位的 iOS 系统，以及 watchOS 上，`OBJC_BOOL_IS_BOOL` 被置为了 0；而 L21 ~ 25 中，如果 `OBJC_BOOL_IS_BOOL` 为 0，那么 BOOL 其实是 `signed char`，而 `@encode(char)` 的值正是 `c`。

破案了。

之前的代码没有考虑到 32 位机型上通过 `method_getReturnType` 拿到的 BOOL 的值是 `c`，导致 if 条件不满足，造成了 iPhone 4s 上的异常。

至于为什么会有这样的设计，应该算是一个历史遗留问题了。NSHipster 的 [Type Encodings](https://nshipster.com/type-encodings/) 里提到：

> `BOOL` is `c`, rather than `i`, as one might expect. Reason being, `char` is smaller than an `int`, and when Objective-C was originally designed in the 80’s, bits (much like the dollar) were more valuable than they are today. `BOOL` is specifically a `signed char` (even if `-funsigned-char` is set), to ensure a consistent type between compilers, since `char` could be either `signed` or `unsigned`.

# How-tos

## Associated Object

使用 Associated Object 可以在 category 中定义 property，使用上只是看上去复杂，其实很简单。比如 setter 的方法：

```objective-c
objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPolicy policy) 
```

看上去有点类似于 key-value 存储，不同的是，key 不是一个字符串，而是 `void *`，可以自己定义，比如：

```objective-c
static void * AssociatedObjectKey = &AssociatedObjectKey;
```

也可以简单用 `@selector` 的值：

```objc
objc_setAssociatedObject(self, @selector(methodName), VALUE, OBJC_ASSOCIATION_COPY_NONATOMIC);
```

另一点是，可以指定存储的 policy，类似于 property 的内存关键字（不确定是不是叫这个），即 weak/strong/assgin/copy，例如：

```objc
// 参考地址：https://nshipster.com/associated-objects/
// assgin, unsafe_unretained
OBJC_ASSOCIATION_ASSIGN

// strong, nonatomic
OBJC_ASSOCIATION_RETAIN_NONATOMIC

// strong, atomic
OBJC_ASSOCIATION_RETAIN

// copy, nonatomic
OBJC_ASSOCIATION_COPY_NONATOMIC

// copy, atomc
OBJC_ASSOCIATION_COPY
```

由上面的对应关系可知，使用 `assign` 来修饰的关联对象其实并不等价于 `weak` 来修饰的 property，并不会置为 `nil`。所以使用这种 policy 需要谨慎。

> Weak associations to objects made with `OBJC_ASSOCIATION_ASSIGN` are not zero weak references, but rather follow a behavior similar to `unsafe_unretained`, which means that one should be cautious when accessing weakly associated objects within an implementation.

另外需要关注的是，关联对象的生命周期。

>  According to the Deallocation Timeline described in [WWDC 2011, Session 322](https://developer.apple.com/videos/wwdc/2011/#322-video) (~36:00), associated objects are erased surprisingly late in the object lifecycle, in `object_dispose()` , which is invoked by NSObject `-dealloc`.

也就是说，关联对象是在对象（HOST）被释放的跟着释放的，更准确来说应该是在其之后。**`dealloc` 方法的调用顺序是从子类到父类直至 NSObject 的，NSObject 的 `dealloc` 会调用 `object_dispose()` 函数，进而移除 Associated Object**。（来自 [Associated Object 与 Dealloc](http://yulingtianxia.com/blog/2017/12/15/Associated-Object-and-Dealloc/)）

```c++
id object_dispose(id obj)
{
    if (!obj) return nil;
	// 销毁对象
    objc_destructInstance(obj);    
  	// 释放内存
    free(obj);

    return nil;
}

void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        // C++ 析构
        if (cxx) object_cxxDestruct(obj);
        // 移除 Associated Object
        if (assoc) _object_remove_assocations(obj);
        // ARC 下调用实例变量的 release 方法，移除 weak 引用
        obj->clearDeallocating();
    }

    return obj;
}
```

考虑这一点是因为如果 HOST 对象是一个全局的，这个对象一旦添加了，其它的地方也会存在（特别是 block，在其它地方也可能会执行）。所以需要在合适的时机把它释放掉。

目前的解决方案是，在调用之后，将其置为 `nil`。

## NSRange 

NSRange 的一些使用方法，参考 [NSHipster](https://nshipster.com/nsrange/)。

需要注意的地方是，NSRange 并不是一个类，而是结构体。对于手动创建的 range，使用之前需要先判断是否越界，比如：

```objc
if (NSLocationInRange(range.location, NSMakeRange(0, array.count)) &&
    range.length >= 1 &&
    NSMaxRange(range) <= array.count) {...}
```

# Tips

## 合并 commit message

> 其实也蛮常用的，因为在不同的分支上开发后直接 merge 到 master 的话会产生无意义的 commit，而且对于 release 上的 commit 往往还需要加更多的说明。

- 先将需要的 commit pick 到新的 branch
- 使用 `git rebase -i HEAD~3` // 3 表示需要合并的数字
- 第一个 commit 选择 pick，后面的都是 squash
- 保存后可以添加新的 commit message

如果需要重命名 commit，也可以通过 `git rebase -i` 召唤出 editor，然后在需要修改的 commit 前填 e (dit)，然后保存退出，再进入就可以编写新的 commit message 了。

[参考](https://github.com/Jisuanke/tech-exp/issues/13)

## iOS 中的行间距设置

为了更好地实现设计稿，设置多行文字的行间距几乎是无可避免的。通常我们有两种方式来设置行间距：一种是 `lineSpacing`，一种是 `maximumLineHeight` 和 `minimumLineHeight`，都是通过设置 NSMutableParagraphStyle。

需要注意的是，这里的 `lineSpacing` 和 Pages 里设置行间距用 1.5 行距之类的不同，而是行与行中间的间距。我拿到的设计稿上直接是没有这个值的，需要手动计算 = = 抛开这个不说，直接设置 `lineSpacing` 还是不精确的，在[在iOS中如何正确的实现行间距与行高](https://juejin.im/post/5abc54edf265da23826e0dc9)这篇文可以看到这么一张图：

![示意图](https://user-gold-cdn.xitu.io/2018/3/29/1626fabdbf367b64?imageView2/0/w/1280/h/960/ignore-error/1)

图中红色部分的高度是 `label.font.lineHeight`，所以为了完全精准的 10pt 行间距，应该这么设置：

```objc
paragraphStyle.lineSpacing = 10 - (label.font.lineHeight - label.font.pointSize); 
```

其中 `pointSize` 是字体的大小，参考[UIFont的lineHeight与pointSize](https://blog.csdn.net/a2331046/article/details/52904529)。

第二种看上去比较准确，从我拿到的设计稿中也能直接拿到，但其实有一些问题，因为它是在文本顶部多出间距，而不是上下均匀间距。尽管可以通过手动调整 `baselineOffset` 来修改，但是就比第一种方式更加麻烦了（代码较多，并且准确度也不如第一种高）。

# Weekly Read Posts

- [怎样花两年时间去面试一个人](http://mindhacks.cn/2011/11/04/how-to-interview-a-person-for-two-years/)

  可以从招聘者的角度看待面试这件事情，脱颖而出的核心在于阅读积淀 + GitHub 项目。

- [今日头条iOS端安装包大小优化—思路与实践](https://techblog.toutiao.com/2018/06/04/gan-huo-jin-ri-tou-tiao-iosduan-an-zhuang-bao-da-xiao-you-hua-si-lu-yu-shi-jian/)

  整理项目的工程资源时看到的，不仅做了代码和资源两方面的优化，还加了监控和量化。

- [更可靠和高精度的 iOS 定时器](http://blog.lessfun.com/blog/2016/08/05/reliable-timer-in-ios/)

  这篇列出了 iOS 中常用的定时器方法，并且比较分析了各自的精度、可靠性等。

- [Push 和 Pop 的那些坑](http://blog.lessfun.com/blog/2015/09/09/uiviewcontroller-push-pop-trap/)

  主要提到了 连续 push/pop 及手势返回等可能造成 crash 的操作及对应的解决方案。

再贴两个准备周末仔细看的链接吧。

- [Debuging with Xcode](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/debugging_with_xcode/chapters/special_debugging_workflows.html#//apple_ref/doc/uid/TP40015022-CH9-DontLinkElementID_1)，觉得自己的 debug 能力（效率 + 准确率）有待提高，可以看一下这个有好的辅助工具可以学习一下。
- [细说ReactiveCocoa的冷信号与热信号](https://tech.meituan.com/talk_about_reactivecocoas_cold_signal_and_hot_signal_part_1.html) 与 [ReactiveCocoa中潜在的内存泄漏及解决方案](https://tech.meituan.com/potential_memory_leak_in_reactivecocoa.html)，上一期遇到了这种特征不明显的循环引用，leader 当时推荐了美团关于 RAC 的系列文章，感觉非常需要学习一下了。