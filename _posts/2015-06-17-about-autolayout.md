---
layout: post
title: AutoLayout 的一些使用心得
date: 2015-06-17 16:45 +0800
---

* TOC
{:toc}
上周终于把前段的密集课程及考试什么的完成了，有时间来 review 下 [YoNote](https://github.com/zchan0/YoNote)里的代码。虽说代码好像写了不少，但能说的好像不太多＝＝这篇准备说下使用AutoLayout 的一些感受。

# AutoLayout是什么

我的理解是，AutoLayout 是通过对界面上各个元素设定约束，来达到在各个不同设备上统一布局的目的。

# 使用 AutoLayout 的方式

使用 AutoLayout 总体上说有两种方式，一种是在 Xib 里通过 GUI 直接设置，另一种是使用代码。其中使用代码的方式常用的有以下几种:

- 苹果自己的 AutoLayout 方法
- 第三方库 PureLayout
- 第三方库 Masonry
- 别的...

对我来说，苹果那堆代码太繁琐抽象，Masonry 还没怎么用过，所以要么用Xib，要么用 PureLayout。

到底选用哪一种，一般取决于需要布局的元素的复杂程度。如果需要控制的元素很多或者是一个 View 上有很多子 View，我会选择用 Xib，毕竟代码写起来太费时间了；相反，如果界面不复杂、要控制的元素比较少，或者元素可能会发生变化，我会考虑用代码，使用啊或者控制方便一些。

# 如何开始布局

虽然我还不能像某大神那么自信地说 “什么界面都能在两(半)小时里做出来”，偶尔有的元素还是会不乖乱跑，但是大体上有个思路。

>  抓住主要矛盾，在布局中到底是哪个元素是“自变量”，因为它别的元素的位置才会跟着变化，先确定自变量，别的元素跟着就确定下来了。

## 一个例子

比如 YoNote 的 `ItemCell`，长这样:

![itemcell.png](http://home.ustc.edu.cn/~sa514014/img/itemcell.png)

分析可以知道它的问题在于 `memoLabel` 内容显示这个 view 必须固定在左上方，而且与 `collectionLabel`的距离是固定的；同时 `imageView` 也必须固定在右边。

具体来说，在这个 table cell 里，`memoLabel` 和 `imageView` 是必须首先固定的，因为三个 label 都与 `imageView` 有一定的距离，而这个距离只有当 `imageView` 固定之后才能确定；同样，`collectionLabel` 与 `memoLabel` 的距离也只有当 `memoLabel` 固定之后才能确定。

于是，按这个思路，先固定 `imageView`：与父视图 `contentView` 的上下右距离确定

```objective-c
[self.iv autoPinEdgesToSuperviewEdgesWithInsets:ALEdgeInsetsMake(kLabelVerticalInsets, kLabelHorizontalInsets, kLabelVerticalInsets, kLabelHorizontalInsets) excludingEdge:ALEdgeLeading];
```

然后是 `memoLabel` ，与父视图的左边及上边确定，然后确定高度以及和`imageView` 的距离。（宽度不用管了，左边与父视图的距离以及右边与`imageView` 的距离确定就能确定宽度，高度一定要固定，方能固定`collectionLabel`）

```objective-c
[self.memoLabel autoPinEdgeToSuperviewEdge:ALEdgeTop withInset:kLabelVerticalInsets];  
[self.memoLabel autoPinEdgeToSuperviewEdge:ALEdgeLeading withInset:kLabelHorizontalInsets];
[self.memoLabel autoSetDimension:ALDimensionHeight toSize:kMemoLabelHeightToSize];
[self.memoLabel autoPinEdge:ALEdgeTrailing toEdge:ALEdgeLeading ofView:self.iv withOffset:-kLabelHorizontalInsets relation:NSLayoutRelationLessThanOrEqual];
```

最后是 `collectionLabel` 和 `tagLabel`，与父视图保持左边的距离，同时`collectionLabel` 与 `memoLabel`以及 `tagLabel` 在上下保持距离。

```objective-c
[self.collectionNameLabel autoPinEdgeToSuperviewEdge:ALEdgeLeading withInset:kLabelHorizontalInsets];
[self.tagLabel autoPinEdgeToSuperviewEdge:ALEdgeLeading withInset:kLabelHorizontalInsets];
[self.collectionNameLabel autoPinEdge:ALEdgeTop toEdge:ALEdgeBottom ofView:self.memoLabel withOffset:kLabelVerticalInsets];
[self.collectionNameLabel autoPinEdge:ALEdgeBottom toEdge:ALEdgeTop ofView:self.tagLabel withOffset:-kLabelVerticalInsets];
```

Done！