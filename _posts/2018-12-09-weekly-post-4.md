---
layout: post
title: 一周小结 5：实现一个以固定间距左对齐的 UICollectionViewFlowLayout
date: 2018-12-09 11:24 +0800
---

* TOC
{:toc}
这周是上一个 sprint 完结新一个 sprint 开始的一周，所以感觉节奏还比较轻松。主要完成的工作是继续解之前攒的 bug，以及设计妹纸对动效提出的优化要求。

# 以固定间距左对齐的 UICollectionViewFlowLayout

上个周期里写了一个选取多个标签的页面，每一个标签的内容是根据后端返回的字符串来给的，所以会有不同的宽度。设计希望这些标签能按照固定的间距左对齐排列。

首先，对于这种 Grid 类型的样式，使用 UICollectionViewFlowLayout 即可。但是它仅提供了两个最小值：`minimumLineSpacing`  和 `minimumInteritemSpacing`。当 collectionView 足够宽的时候，每一行（除了最后一行，似乎是个例外，会默认左对齐）元素之间的间距会大于这个最小距离。

有两种方案让 cell 以固定间距左对齐。

## 通过 sectionInset 挤压

按照上面的分析，元素之间的间距会大于最小值是因为 collectionView 可以放 cell 的地儿太宽了，如果每一行 collectionView 的宽度恰好是：

>  每一行能放下的最多 item 个数的宽度和 + 固定间距 x （最多元素个数 - 1）

正好就避免了“太宽”的情况。

不过使用这个方案的前提是，需要将每一行都作为一个 section（因为修改的是 `sectionInset`），而每行的 item 个数不同、每个 item 的宽度可能也不同。如果数据源原本就是如此还好，否则就需要修改数据源的结构，还是挺麻烦的。

一个例子：

```objective-c
- (UIEdgeInsets)collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout *)collectionViewLayout insetForSectionAtIndex:(NSInteger)section {
	CGFloat itemWidth, rowWidth = 0, maxRowWidth = 0;
	for (int i = 0; i < self.feedbackTags.count; ++i) {
		itemWidth = ...;
		if (rowWidth + itemWidth < TagsViewWidth) {
			rowWidth += itemWidth + kInteritemSpacing;
		} else {
			maxRowWidth = MAX(rowWidth, maxRowWidth);
			rowWidth = itemWidth;
		}
	}	
	return UIEdgeInsetsMake(0, 0, 0, TagsViewWidth - maxRowWidth);
}
```

## 继承 UICollectionViewFlowLayout

除了修改每个  `sectionInset` 之外，还可以通过继承 UICollectionViewFlowLayout，重写 `layoutAttributesForElementsInRect:` 方法来修改每个 item 的 layout。

```objective-c
- (NSArray *)layoutAttributesForElementsInRect:(CGRect)rect {
    NSArray* attributesToReturn = [super layoutAttributesForElementsInRect:rect];
    for (UICollectionViewLayoutAttributes* attributes in attributesToReturn) {
        if (attributes.representedElementKind == nil) {
            NSIndexPath* indexPath = attributes.indexPath;
            attributes.frame = [self layoutAttributesForItemAtIndexPath:indexPath].frame;
        }
    }
    return attributesToReturn;
}
```

- `representedElementKind` 如果为 UICollectionElementCategory 则其值就是 `nil`。而 UICollectionElementCategory 其实就是 collectionViewCell，对应的还有 UICollectionElementCategorySupplementaryView 和 UICollectionElementCategoryDecorationView。
- 所以 `layoutAttributesForElementsInRect:` 可以给 cell、supplementary 和 decoration 都指定 layout，UICollectionViewLayoutAttributes 包含了 `indexPath` 属性。

那么如何去设置每个 item 的 layout 呢？

很简单，如果从前一个 item 的最右边开始，加上当前 item 的宽度，不超过 collectionView 的内容宽度（`width - sectionInset.right ` ），那么该 item 可以放在当前一行，否则需要换行；一个边界情况是，对于第一个 item，肯定是需要放在最左边的。

```objective-c
- (UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath {
    UIEdgeInsets sectionInset = [(UICollectionViewFlowLayout *)self.collectionView.collectionViewLayout sectionInset];
    UICollectionViewLayoutAttributes *currentItemAttributes = [super layoutAttributesForItemAtIndexPath:indexPath];
    CGRect currentFrame = currentItemAttributes.frame;
    
    // 第一个默认左对齐
    if (indexPath.item == 0) {
        currentFrame.origin.x = sectionInset.left;
        currentItemAttributes.frame = currentFrame;
        return currentItemAttributes;
    }
    
    NSIndexPath *previousIndexPath = [NSIndexPath indexPathForItem:indexPath.item - 1 inSection:indexPath.section];
    CGRect previousFrame = [self layoutAttributesForItemAtIndexPath:previousIndexPath].frame;
    CGFloat previousFrameRightPoint = CGRectGetMaxX(previousFrame) + self.interitemSpacing;
    
    // 另起一行
    if (previousFrameRightPoint + currentFrame.size.width > self.collectionView.bounds.size.width - sectionInset.right) {
        currentFrame.origin.x = sectionInset.left;
        currentItemAttributes.frame = currentFrame;
        return currentItemAttributes;
    }
    
    // 同一行
    currentFrame.origin.x = previousFrameRightPoint;
    currentItemAttributes.frame = currentFrame;
    return currentItemAttributes;
}
```

实现好这个 layout 之后，可以放在项目里反复使用了。

# 一个点击按钮放大的效果实现

这期还有个需求是点击某个按钮的时候，这个按钮先放大 1.2 倍，然后再做接下来一些动画。

一开始直接用了 `bk_whenTapped:`，在 block 中完成按钮放大然后还原的动画，然而这并不能让设计同学满意。跟设计同学反复确认之后的需求是：

- 点击手势的一开始，按钮就放大
- 手势没有结束，按钮依然保持 1.2 倍，并且不会进行接下来的动画
- 手势结束，如果停在当前按钮的区域，则继续接下去的动画；否则，按钮变回原来的大小

通过 `bk_whenTapped` 添加的是 UITapGestureRecognizer，block 在点击完成之后才会执行。显然这连第一个需求都无法满足。

StackOverflow 上有人推荐 UILongPressGestureRecognizer，只需要将它的 `minimumPressDuration` 设置为 0，在手势的 `state` 为 UIGestureRecognizerStateBegan 时，手势所绑定的事件就会被执行。但是这却仍达不到第二个需求的要求。

UIView 是 UIResponder 的子类，UIResponder 有以下四个方法处理触摸事件，UIView 可以重写这些方法去自定义事件处理：

```objective-c
// Generally, all responders which do custom touch handling should override all four of these methods.
// Your responder will receive either touchesEnded:withEvent: or touchesCancelled:withEvent: for each
// touch it is handling (those touches it received in touchesBegan:withEvent:).
// *** You must handle cancelled touches to ensure correct behavior in your application.  Failure to
// do so is very likely to lead to incorrect behavior or crashes.
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event;
```

需求里提及的手势开始和结束都有了明确的处理时机，对于第三点，则需要添加一个对于触摸位置的判断：

```objective-c
- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesMoved:touches withEvent:event];
    
    CGPoint point = [touches.anyObject locationInView:self];
    _isTouchInside = CGRectContainsPoint(self.bounds, point);
}
```

这样，在手势结束的时候就可以通过 `_isTouchInside` 来判断手势最后是否停留在按钮的区域：

```objective-c
- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [super touchesEnded:touches withEvent:event];
    
    [UIView animateWithDuration:kAnimationDuration animations:^{
        self.transform = CGAffineTransformIdentity;
    } completion:^(BOOL finished) {
        if (_isTouchInside) {
            [self.animationCompletedSignal sendNext:nil];
        }
    }];
}
```

# More

虽然努力地想描述发现问题、分析问题以及解决问题的过程，但是文笔真的很苍白啊 = = 

总之，还是希望不管是解 bug 还是做需求都能够这样有条不紊地去思考。在提升业务能力的这个阶段，一方面是积累/扩宽解决问题的思路和能力，另一方面也是不断去巩固理解基础知识，再有就是能比较不同方案的优劣。

> Experts minimize the need for conscious reasoning.