---
layout: post
title: 周记 4：使用 BlocksKit 的一个注意事项
date: 2018-12-02 18:15 +0800
---

* TOC
{:toc}
距离上次写周记竟然就过去了两个月，都有点想不起当时开这个系列的坑的雄心壮志了。这两个月里感觉不停地被需求追着跑，停下来反思的时候都不太说得出到底学到些什么。如果硬要说的话，大概是 debug 能力比之前强了一些，业务能力也许也进步了一些？总的来说都是一些主观（也许强行安慰自己）的感受，并没有具体的数字可以说明，连自己都说服不了自己呢（微笑）。

唯二值得庆幸的是，现在又慢慢开始有时间了，以及这段时间虽然没有整理周记，但是也尽力把遇到的问题都记下了，哪怕真的只记下了问题 = =

现在的学习不可能像念书的时候有那么多时间，所以一切的选择都会基于很多考量。目前我能抓到的主要方式就是不断积累和反思，另外也希望能尽快适应工作的节奏，从而将自己的作息也调整到一个相对健康的节奏。

# Debug 的故事

这次先说说最近比较有收获的两次 debug 事件吧。

上个 sprint 里有个需求是，一个带搜索框的页面，搜索框可以根据用户上下滑动的操作出现和隐藏。实现并不难，在 scrollView 的代理方法里做一个简单的数学问题。由于之前已经封装好了类似效果的计算方法，我就直接调用了。然后测试同学发现了一个 case，就是 webView 的页面，在第一次加载的时候，这个搜索框会自动隐藏。

debug 的时候发现 webView 会在加载完 url 之后，就调用 `scrollViewDidScroll:`。所以一开始的思路个人觉得非常自然，就是想知道为什么 webView 会去调用以及有没有什么方法让它别调用了。当然一通搜索之后是没有找到答案的 = = 

后来换了个思路，看看调用之后当时的 offset 等数据长什么样，能不能调整算法 cover 住这种特殊情况。于是发现，是之前的算法在判断移动方向时，漏掉了一种情况。简单了改了下算法之后，果然就能用了，修改的代码也就五六行。

尽管写下来之后发现好像没什么意思 😂，但当时整个定位问题的过程还是有收获的。一方面提醒我，遇到 bug 的时候不能只去搜索表现形式来找解决问题的方案，而是要真正找到原因所在；另一方面，因为当时也算是一个紧急 bug，时间紧任务重、压力还挺大的，还是扛过来了。

另外一个 debug 故事是关于 BlocksKit 的。有一个模块是由上中下三个部分拼起来的，由于历史原因，虽然三个部分点击后跳转用到的 schema 是一样的，但是点击事件的处理仍然是各自处理的。测试同学发现了一个偶现的 bug，在某些时候，上面部分和中下跳转到了不同的地方。

第一次发现这个问题的时候感觉一头雾水，查了代码没有 typo 的问题，抓包发现后端给的 schema 也没有问题。后来对比了上中下三部分的实现，发现只有上面是通过 BlocksKit 的方法来加的点击事件，中下都是 UIButton，所以都是用的 ReactiveObjc。于是看了下 `bk_whenTapped:` 这个方法的注释：

```objective-c
/** Adds a recognizer for one finger tapping once.
 
 @warning This method has an _additive_ effect. Do not call it multiple
 times to set-up or tear-down. The view will discard the gesture recognizer
 on release.

 @param block The handler for the tap recognizer
 @see whenDoubleTapped:
 @see whenTouches:tapped:handler:
 */
- (void)bk_whenTapped:(void (^)(void))block;
```

注意到 warning 的部分，于是又继续去找了 BlocksKit 的源码：

```objective-c
- (void)bk_whenTouches:(NSUInteger)numberOfTouches tapped:(NSUInteger)numberOfTaps handler:(void (^)(void))block
{
	if (!block) return;
    // 1
	UITapGestureRecognizer *gesture = [UITapGestureRecognizer bk_recognizerWithHandler:^(UIGestureRecognizer *sender, UIGestureRecognizerState state, CGPoint location) {
		if (state == UIGestureRecognizerStateRecognized) block();
	}];
	
	gesture.numberOfTouchesRequired = numberOfTouches;
	gesture.numberOfTapsRequired = numberOfTaps;

	[self.gestureRecognizers enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
		if (![obj isKindOfClass:[UITapGestureRecognizer class]]) return;

		UITapGestureRecognizer *tap = obj;
		BOOL rightTouches = (tap.numberOfTouchesRequired == numberOfTouches);
		BOOL rightTaps = (tap.numberOfTapsRequired == numberOfTaps);
		if (rightTouches && rightTaps) {
            // 2
			[gesture requireGestureRecognizerToFail:tap];
		}
	}];

	[self addGestureRecognizer:gesture];
}

- (void)bk_whenTapped:(void (^)(void))block
{
	[self bk_whenTouches:1 tapped:1 handler:block];
}
```

可以看到 1，每次调用 `bk_whenTapped:` 都会添加一个新的手势在 UIView 上，那么问题来了，如果 UIView 有了一组手势，它会怎么响应呢？2 用到了一个方法：`requireGestureRecognizerToFail`，这个方法设定了两个手势之间的关系，完整的方法签名是：

```objective-c
- (void)requireGestureRecognizerToFail:(UIGestureRecognizer *)otherGestureRecognizer;
```

文档里是这么说的：

> This method creates a relationship with another gesture recognizer that delays the receiver’s transition out of [`UIGestureRecognizerStatePossible`](https://developer.apple.com/documentation/uikit/uigesturerecognizerstate/uigesturerecognizerstatepossible?language=objc). The state that the receiver transitions to depends on what happens with `otherGestureRecognizer`: 
>
> - If `otherGestureRecognizer` transitions to [`UIGestureRecognizerStateFailed`](https://developer.apple.com/documentation/uikit/uigesturerecognizerstate/uigesturerecognizerstatefailed?language=objc), the receiver transitions to its normal next state.
> - if `otherGestureRecognizer` transitions to [`UIGestureRecognizerStateRecognized`](https://developer.apple.com/documentation/uikit/uigesturerecognizerstate/uigesturerecognizerstaterecognized?language=objc) or [`UIGestureRecognizerStateBegan`](https://developer.apple.com/documentation/uikit/uigesturerecognizerstate/uigesturerecognizerstatebegan?language=objc), the receiver transitions to [`UIGestureRecognizerStateFailed`](https://developer.apple.com/documentation/uikit/uigesturerecognizerstate/uigesturerecognizerstatefailed?language=objc).
>
> An example where this method might be called is when you want a single-tap gesture require that a double-tap gesture fail.

所以，只有当 `otherGestureRecognizer` 响应失败的时候，才会去响应当前手势。回到 BlocksKit 的源码 2 可以看到，BlocksKit 将原来已有的手势设定为 `otherGestureRecognizer` 。所以如果在 UITableViewCell 这种可能会被复用的 UIView 里通过 `bk_whenTapped:` 来添加手势，如果手势应该响应的内容会发生变化，那么就有可能出现响应事件不一致的情况。

到此，终于破案了。。

# 一些 Tips

## extern vs. static 在 Objective-C 中这两个的使用有什么区别？

• 对于 static，在引用了相应的 .h 文件的 .m 文件中，都会创建一个 aConstant，在那个 .m 文件中，aConstant 是全局静态变量

• 对于 extern，aConstant 的定义是在其自己的 .m 文件中，所以所有引用 .h 文件，其实是共享的一个 aConstant（在内存空间来看）

>  In the extern  example, you have 1 MyGlobalConstant. 
>
> In the static  example, you will have tens if not hundreds of separate MyGlobalConstants all with the same value.

参考：

• https://stackoverflow.com/a/25351208

## 关于 UILabel 设置 AttributedText 以后末尾...不出现的问题

两种方案：

• 在设置了 attributedText 之后手动设置 lineBreakMode（也就是跟着再调用一次= =

• 直接作为富文本属性

虽然第一种方案很奇怪，但是第二种方案有一个副作用，就是可能导致富文本的高度计算失败。。

另一种可能导致富文本高度计算出错的是设置 paragraphStyle 的 maxLineHeight 和 minLineHeight，怀疑是设置 paragraphStyle 回导致高度计算出错。

参考：

- https://www.cnblogs.com/xyzaijing/p/3887103.html

## UITextView 最大字数限制问题

这个其实挺常用的，一般（包括我）第一时间都会想到使用 `shouldChangeTextInRange` 这个方法，超过某字数后就返回 NO。但是这么写有个人觉得挺严重的交互问题，那就是因为这个方法是针对每个输入的字符来判断的，而不管是英文还是中文，尤其是中文，虽然一个汉字是一个字符，但是输入的时候拼音可能是五六七八或者更多个字母，那么可能输入到最后只剩两三个字符的时候一个汉字都无法输入了，那最后的几个可输入的字符数就浪费了。

可以换一个思路，输入的时候可以随便输入，只需要对输入的字符串进行判断是否超过最大字数，如果超过了就进行字符串截断即可。

于是有两个问题需要解决。首先是时机问题，`textDidChange:` 是一个比较好的时机，能拿到用户这一次停止输入的时候的字符串；但是另外一个问题又来了，英文有单词联想的功能，中文停止时可能还没有选中拼音对应的汉字，所以那个时候拿到的 text 可能并不准确。可以通过 `markedTextRange` 来解决：

```objective-c
[[[NSNotificationCenter defaultCenter] rac_addObserverForName:UITextViewTextDidChangeNotification object:nil] subscribeNext:^(NSNotification * _Nullable x) {
        @strongify(self);
        UITextView *textView = x.object;
        // 通过 markedTextRange 判断是否存在高亮字符，没有才进行字数统计和截断操作
        if ((!textView.markedTextRange || textView.markedTextRange.isEmpty)
            && textView.text.length > kWordCountLimit) {
            self.textView.text = [textView.text substringToIndex:kWordCountLimit];
        }
    }];
```

## 忽略大小写比较

没什么好说的只是一个积累，直接上代码吧 = =

```objective-c
if( [@"Some String" caseInsensitiveCompare:@"some string"] == NSOrderedSame ) {
// strings are equal except for possibly case
}
```

