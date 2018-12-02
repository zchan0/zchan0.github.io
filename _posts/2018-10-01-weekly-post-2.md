---
layout: post
title: 周记 3：一个叫 AlamofireOAuth1 的库
date: 2018-10-01 17:19 +0800
---

国庆快乐 🎉

# AlamofireOAuth1

今天终于把 [AlamofireOAuth1](https://cocoapods.org/pods/AlamofireOAuth1) 憋出来了，截图纪念 🍻

![Imgur](https://i.imgur.com/rAMskUB.png)

说起来，这个库写了有两年时间了，第一个 [commit](https://github.com/zchan0/AlamofireOAuth/commit/dd2678cfc514742a56f86e59e87507f0ea7d0937) 还是 16 年的，也不好说是拖延症还是初心不改了（笑）。写这个的初衷是为了写饭否的客户端，饭否只支持 OAuth1 的授权方式；而 Swift 上并没有太多获取 OAuth1 授权的库可以选择，基本上都用的 [OAuthSwift](https://github.com/OAuthSwift/OAuthSwift)。这种情况下，当然就只能撸起袖子造轮子了。

目前 AlamfireOAuth1 主要做了三件事情：

1. 初始化 OAuth1 实例后，一步获取 access token
2. 提供一个基于 Keychain 的封装可以安全地存储 access token 
3. 提供一个方法 adapt URL request with access token 

