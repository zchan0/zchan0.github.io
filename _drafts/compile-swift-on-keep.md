本地环境：Xcode 10.2.1 (10E1001), New Build System

![Screen Shot 2019-05-05 at 4.05.36 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-05 at 4.05.36 PM.png)

## KEPBadgeDetailView

查了下 Build log，有一些不相关的 Swift 文件，最多的是 

- LocationNode.swift
- KEPFollowedHsahtagViewController.swift
- WatermarkCenterVC.swift
- KEPTrainingLiveBriefView.swift
- KEPPleasureDetailViewController.swift
- KEPBadgeGroupShareView.swift
- KEPCurrentUserFollowingController.swift
- SCNVector3+Extensions.swift
- KEPVideoEditorCutBar.swift
- ...

其中有一些也是编译耗时比较长的，不太确定有什么关联。（可以查一下 WWDC 有一个 session：Behind Xcode Build Process 啥的

![Screen Shot 2019-05-06 at 3.42.51 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 3.42.51 PM.png)

这个文件修改如上，主要改的点：

1. 三元运算符（包括 KEPDeviceManager 里的实现）
2. 提前计算：初始化方法（这一点其实可以再研究

结果：

28294.4 ms -> 1789.8 ms

## KEPVideoEditorCutBar

直接看 git diff 

![Screen Shot 2019-05-06 at 3.53.14 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 3.53.14 PM.png)

修改的点：

1. if 条件中不能包含大量运算，可以提前计算好
2. 显式声明这种通过运算初始化的变量的类型

结果：

上述修改无效 == 需要再看看

## LocationNode

![IMG_EF64D17A8CCE-1](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/IMG_EF64D17A8CCE-1.jpeg)

修改的点：

1. 显式声明变量的类型
2. 提前计算，不在初始化方法中再去计算

结果：

6754.9 ms -> 1651.6 ms

## TrainingLevelProgressCardView

这个文件里有两个方法：`updateInfo:` 和 `contentAnimation` 。

修改的点都是将 `filter` 改为使用 `first` 。[Performance, functional programming and collections in Swift](https://www.avanderlee.com/swift/performance-collections/) 里提到的一点是：

> Prefer contains over first(where:) != nil

`updateInfo:` 

![Screen Shot 2019-05-06 at 6.24.57 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 6.24.57 PM.png)

`contentAnimation`

![Screen Shot 2019-05-06 at 6.30.22 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 6.30.22 PM.png)

结果：

`updateInfo:` : 6032.6 ms -> 4158.2 ms

`contentAnimation`: 6268.6 ms -> 3752.6 ms

有明显改善，但是还有优化空间。还有字符串拼接和三元运算符的问题，可以再修改一下。

## CommenTool

// 拼写 😓️

主要是 `fillHeader` 方法。

![Screen Shot 2019-05-06 at 7.00.42 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 7.00.42 PM.png)

修改的点：提前计算，if 条件判断中不能进行大量计算。

结果：5201.3 ms -> 2170.7ms

有明显改善，也还可以看看其它优化的地方。比如：

```swift
var fields = (KUtils.loadUserCookie(url.absoluteString) as? [String : String]) ?? [:]
```

使用了三元运算符，以及没有显式声明 `fields` 的类型。

## MiniProfileUserView

![Screen Shot 2019-05-06 at 7.04.33 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 7.04.33 PM.png)

修改的点：显式声明变量类型

结果：5036.9 ms -> <10 ms

## KEPVideoDetailListViewController

![Screen Shot 2019-05-06 at 7.06.14 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 7.06.14 PM.png)

修改的点：三元运算符

结果：无效= =

## KEPBadgeDetailViewController

![Screen Shot 2019-05-06 at 7.08.05 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 7.08.05 PM.png)

修改的点：三元运算符 + 显式声明变量类型

结果：4702.0 ms -> 3466.3 ms

还有优化空间

## PersonalQRCodeViewController

![Screen Shot 2019-05-06 at 7.22.34 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 7.22.34 PM.png)

修改的点：增加临时变量 + 显式声明变量类型

结果：4309.2 ms -> 1980.7 ms

效果还比较明显，降了一半，但是还可以看看。感觉是调用 `fillHeader()` 导致的，可以注释调用试试。

---

## 第一次修改小结

![Screen Shot 2019-05-06 at 6.08.04 PM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-06 at 6.08.04 PM.png)

第一波修改效果还可以，主要修改的点在于：

- 减少使用三元运算符和 nil coalescing
- 显式声明变量类型，尤其是使用复杂的表达式初始化的变量，以及 closure 的参数类型和返回值类型
- 提前计算，尤其是 if 条件判断和初始化方法里，避免大量计算
- 函数式的性能问题

总的改善时间：24 min -> 16 min 52 s

---

## KEPVideoEditorCutBar

![Screen Shot 2019-05-07 at 10.59.53 AM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-07 at 10.59.53 AM.png)

第二次修改主要的点在于：先提前计算好了 `width` 再进行比较（感觉不仅在 if 条件判断中不应该进行大量计算，普通的判断也应该先提前计算再比较）

结果：29083.6 ms -> 2024.5 ms

## KEPTrainingLivefView

>  这个文件第一次没有修改，是在第一次修改后排到前面来了

![Screen Shot 2019-05-07 at 11.06.15 AM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-07 at 11.06.15 AM.png)

修改的点也主要在「提前计算」这一条上。

结果：6033.8 ms -> 3090.4 ms

## KEPVideoDetailListViewController

![Screen Shot 2019-05-07 at 11.15.44 AM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-07 at 11.15.44 AM.png)

跟第一次相比，4873.1 ms -> 4076.7 ms，没有明显改善。

---

## 第二次修改小结

![Screen Shot 2019-05-07 at 10.49.45 AM](/Users/zhengcc/Documents/Dropbox/Projects/Blog/assets/img/Screen Shot 2019-05-07 at 10.49.45 AM.png)

主要修改的点还是那些，看上去有明显减少的是「提前计算」这一条。但是另外有几个文件的效果不明显，可以再进行第三次修改。

第三次修改着眼于两个问题：

- 第一次修改中有几个文件还需要优化的点
- 去掉一些可能包含复杂运算的函数调用，会不会提升时间？

总的来说，解决了耗时最长的那个问题，还是有帮助的。两次修改可以将编译时间减少至少五分钟左右。

