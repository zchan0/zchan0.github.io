---
layout: post
title: iOS 实现分享到微信
date: 2017-04-20 10:10 -0500
categories: notes iOS
---

* TOC
{:toc}

主要内容包括集成微信 SDK 的步骤，如何通过 Swift 调用，以及如何实现一次分享。Demo 封装了一个 ShareManager，可以快速实现分享文字、图片和链接到微信。

# 安装和配置
## Step 1
在[微信开放平台](https://open.weixin.qq.com/)注册应用程序 ID，获得 AppID。

> 微信对新建的应用需要审核，审核需要7天。Demo 里使用的是 [WeDemo](https://github.com/Tencent/WeDemo) 的 AppID

## Step 2
* 添加 URL schemes 为 AppID，这样微信可以回调起你的 app。

![](https://ww1.sinaimg.cn/large/006tNbRwgy1fea42f8zsjj318505gdgd.jpg)

* 在 `info.plist` 中添加白名单，否则在没有安装微信的环境（比如模拟器）中会报错 `-canOpenURL: failed for URL`

![](https://ww4.sinaimg.cn/large/006tNbRwgy1fea42e7pg9j30mk01qdfx.jpg)

## Step 3
* 通过 CocoaPods 集成微信 SDK

```bash
pod 'WechatOpenSDK'
```

* 添加 Objective-C Bridging Header
    确认在 `Building setting - Swift Compiler - Code Generation` 中添加了 bridging header 的路径。比如 `“YourApp/YourApp-Bridging-Header.h”`

* 在 `YourApp-Bridging-Header.h` 中添加下面的代码，然后不用`import`，就可以在 Swift 中使用相关方法了。

```objective-c
#import “WXApi.h”
```

# 如何实现
* 向微信终端程序注册第三方应用

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {        		WXApi.registerApp("APP-ID")
    return true
}
```

* 发送请求到微信
    过程：创建多媒体消息(`WXMediaMessage`)或者富文本(`String`)消息，然后创建`SendMessageToWXReq`请求，最后通过`WXApi.send()`方法向微信发起请求。

* 处理微信的回应
    实现`onResp`协议方法

# Demo
```swift
// 分享文字
ShareManager.shared.sendText(text, inScene: WXSceneSession)
// 分享图片
ShareManager.shared.sendImage(imageData, inScene: WXSceneTimeline)
// 分享网页链接
let url = "https://httpbin.org/get"
ShareManager.shared.sendLink(url, text, inScene: WXSceneSession)
```

[Repo](https://github.com/zchan0/WechatShareDemo)

