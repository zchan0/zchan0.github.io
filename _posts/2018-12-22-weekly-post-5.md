---
layout: post
title: 一周小结 6：算一算时间差
date: 2018-12-22 13:13 +0800
---

* TOC
{:toc}
# 算一算时间差

## 方案一

说到计算时间差的方式，一般都会想到使用 [`components:fromDate:toDate:options:`](https://developer.apple.com/documentation/foundation/nscalendar/1408595-ordinalityofunit) ，比如：

```objective-c
NSDate *startDate = ...;
NSDate *endDate = ...;
 
NSCalendar *gregorian = [[NSCalendar alloc] initWithCalendarIdentifier:NSGregorianCalendar];
 
NSUInteger unitFlags = NSMonthCalendarUnit | NSDayCalendarUnit;
 
NSDateComponents *components = [gregorian components:unitFlags fromDate:startDate  toDate:endDate options:0];
NSInteger months = [components month];
NSInteger days = [components day];
```

这种方式计算结果的时候只会到提供的最小时间单位，比如上面的代码设定的最小单位是 `NSCalendarUnitDay`，则会通过判断两个时间点是否是超过 24 小时来计算天数。

这样比较有一个问题，对于 12/18 23:30 和 12/19 12:13 这两个时间点，由于不超过 24 小时，会被认为是同一天。但是在一些场景下，人们会认为超过了凌晨十二点则是“第二天”了。

**解决方案**：

可以将 `startDate` 和 `endDate` 借由 `startOfDayForDate:` 方法先转换到当天的 00:00 之后再传入。

## 方案二

另一种方案是使用 [`ordinalityOfUnit:inUnit:forDate:`](https://developer.apple.com/documentation/foundation/nscalendar/1408595-ordinalityofunit)。这个方法在计算相差多少天时，不是通过是否超过 24 小时来判断的，而是会考虑上面提到的凌晨十二点。

```objective-c
@implementation NSCalendar (MySpecialCalculations)
- (NSInteger)daysWithinEraFromDate:(NSDate *)startDate toDate:(NSDate *)endDate {
     NSInteger startDay=[self ordinalityOfUnit:NSDayCalendarUnit inUnit: NSEraCalendarUnit forDate:startDate];
     NSInteger endDay=[self ordinalityOfUnit:NSDayCalendarUnit  inUnit: NSEraCalendarUnit forDate:endDate];
     return endDay - startDay;
}
@end
```

但是这个方法仍然有问题。尽管它是 NSCalendar 的方法，尽管 NSCalendar 的 timezone 设置了当前时区，但是其计算的时候仍然是按照 UTC 来计算的，导致对于某些时间点，转成 UTC 之后就恰好是同一天了。比如 12/22 07:30 与 12/11 23:00 UTC+8，转成 UTC 之后则是 12/21 23:30 与 12/21 15:00 UTC，对于东八区的用户来说这两个时间点相差了一天，而对于 UTC 来说仍然是同一天。

**解决方案**：

可以对于传入的参数加上当前时区相对于 UTC 的偏移量 `[[NSTimeZone localTimeZone] secondsFromGMT]`。

```objective-c
NSDate *nowDate = [[NSDate date] dateByAddingTimeInterval:[[NSTimeZone localTimeZone] secondsFromGMT]];
```

参考：[Performing Calendar Calculations | Apple](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/DatesAndTimes/Articles/dtCalendricalCalculations.html#//apple_ref/doc/uid/TP40007836-SW8)

# 从 WebView 预加载说起

之前看到同事的代码里通过下面的代码来实现 webView 的预加载。

```objective-c
[self.webViewController.view layoutSubviews]
```

于是搜索了下，在[这篇文章](https://coderwall.com/p/trjkcg/preloading-uiwebview-or-wkwebview-in-swift)里提到可以使用 `webViewController.view.setNeedsLayout()` ，并且异步地调用。不要使用 `layoutIfNeeded` ，因为会阻塞主线程（它必须在主线程去执行）。比如：

```swift
dispatch_async(dispatch_get_main_queue(), { () -> Void in
	// this will asynchronously call the viewDidLoad, but will not layout everything yet and will not block the main thread
	// do not call webViewController.view.layoutIfNeeded() here, because that will block the main thread in a RELEASE mode, which sucks big time
	webViewController.view.setNeedsLayout()
)}
```

前两天遇到一个有点奇怪的 bug ：webViewController 的网络请求失败的空页面出现了布局错误。

![布局错误](https://i.imgur.com/xIRMsH0.png)

Debug 后发现 webViewController 在创建 errorView 时候的 `frame` 不正确，为整个屏幕的大小。发现在给`webViewController.view` 的 `frame` 赋正确的值之前，先触发了 webViewController 的 `viewDidLoad` 方法，在 `viewDidLoad` 方法里自然会去请求加载 URL，由于请求失败，则会显示请求错误的页面。而由于 errorView 是使用懒加载的方式创建的， 并且它的布局是通过 `initWithFrame:` 来设置的，此时的 view 大是整个屏幕的大小了。

破案了。改成用约束布局就可以了。

经过这个 bug 可以小结两件事：

1. 应该不用调用 `setNeedsLayout` 也可以触发 `viewDidLoad`，关键点在于访问了 `view`。
2. 使用懒加载创建的属性变量，需要考虑被设置的属性值是否可能变化，变化后有没有其它地方会 update 这个值。比如上述的使用 `initWithFrame:` 来布局，以及在懒加载里给 collectionView 或者 tableView 设置 delegate 和 datasource。