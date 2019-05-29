---
layout: post
title: 制作 CocoaPods 库并使用 TravisCI 持续集成
date: 2018-08-24 19:23 +0800
---

* TOC
{:toc}
一般 *howto* 应该是先从官方文档开始。以下两篇是非常好的入门教程了：

- [Making a CocoaPod](https://guides.cocoapods.org/making/making-a-cocoapod.html)

- [Using Pod Lib Create](https://guides.cocoapods.org/making/using-pod-lib-create.html)

# pod lib create
首先使用 `pod lib create [pod name]` 就可以在本地建立一个空的 pod 了。执行这个命令时，需要回答一些问题，根据实际情况回答 Yes or No 即可。

然后我们就拥有了一个空空的 Pod 👇

```bash
$ tree MyLib -L 2

  MyLib
  ├── .travis.yml
  ├── _Pods.xcproject
  ├── Example
  │   ├── MyLib
  │   ├── MyLib.xcodeproj
  │   ├── MyLib.xcworkspace
  │   ├── Podfile
  │   ├── Podfile.lock
  │   ├── Pods
  │   └── Tests
  ├── LICENSE
  ├── MyLib.podspec
  ├── Pod
  │   ├── Assets
  │   └── Classes
  │     └── RemoveMe.[swift/m]
  └── README.md
```

在 `ReplaceMe.swift` 层级中加入自己的文件（[有文章](https://www.jianshu.com/p/f218fe3baff8)说这里只能有一个子层级，待验证）。

# Travis CI 配置
第一次配置 `.travis.yml` 可能会觉得有些麻烦。虽然貌似有很多现成的模版可以使用，但建议还是先看懂它的含义。这样，当 CI 挂了的时候，才能有效的 debug。

> 可以参考 TravisCI 提供的文档 [Building an Objective-C or Swift Project](https://docs.travis-ci.com/user/languages/objective-c/) 来学习。

以下是 [ChronosCore](https://github.com/zchan0/ChronosCore) 的 `travis.yml` 👇

```shell
osx_image: xcode9.4
language: swift
cache: cocoapods
podfile: Example/Podfile
before_install:
- gem install cocoapods # Since Travis is not always on latest version
- pod install --project-directory=Example
- brew update
- brew outdated xctool || brew upgrade xctool
script:
- xcodebuild test -workspace Example/ChronosSwift.xcworkspace -scheme ChronosSwift-Example -sdk iphonesimulator -destination 'platform=iOS Simulator,name=iPhone 8,OS=11.4' build-for-testing ONLY_ACTIVE_ARCH=NO
- xctool run-tests -workspace Example/ChronosSwift.xcworkspace -scheme ChronosSwift-Example -sdk iphonesimulator ONLY_ACTIVE_ARCH=NO
- pod lib lint --allow-warnings
- pod spec lint --allow-warnings
- pod trunk push --allow-warnings
```

简单的说明：
在 `script` 部分之前主要是编译环境的配置。需要指定 Xcode 的版本、安装必要的软件，比如 CocoaPods。

虽然在本地可以通过 Xcode 直接运行 Test，但是在 CI 中则需要使用 command line。阿婆有一篇 [Automating the Test Process]( https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/testing_with_xcode/chapters/08-automation.html#//apple_ref/doc/uid/TP40014132-CH7-SW1)中提供了一些命令。比如编译：

```shell
xcodebuild test -project MyAppProject.xcodeproj -scheme MyApp -destination 'platform=Simulator,name=iPhone,OS=8.1'
```

在 `script` 中加入这些命令，就可以在 CI 中进行编译、测试等。`scripts` 最后三行是为了在测试通过后，能自动地验证 spec，从而自动发布库的更新到 CocoaPods 上，后面会详细说明。

但其实这里上面提到的代码并没有用 🙃 没有仔细去研究原因。参考了 objc.io 的文章 [Travis CI for iOS](https://www.objc.io/issues/6-build-tools/travis-ci/) 之后，转向了使用 Xctool（尽管 Facebook 貌似已经挺久没有更新这个库了）。使用的时候还有一些坑，比如可能会出现：

```shell
...
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Failed to inialize simulated service context with error: (null)'
...
```

Google 之后找到了[解决方式](https://github.com/facebook/xctool/issues/740)，将 Xctool 升级到 0.3.4 就可以了。由于 TravisCI 的服务器里已经安装了 Xctool，但是版本并不是 0.3.4，所以使用了 `brew outdated xctool || brew upgrade xctool` 来更新版本。

# 发车准备
在 TravisCI 第一次运行之前，可以先在本地先运行下 spec 验证的命令。

```shell
pod lib lint
pod spec lint
```

这个步骤可能出现的坑也还蛮多的，不过大多数是可以通过 log 修改。比如，

```shell
    - WARN  | [iOS] swift: The validator used Swift 3.2 by default because no Swift version was specified. To specify a Swift version during validation, add the `swift_version` attribute in your podspec. Note that usage of the `--swift-version` parameter or a `.swift-version` file is now deprecated.
```

之前看到一种方式是，新建一个 `.swift-version` 的文件，其中写上使用的 swift version。但是看这个 message 的意思是这种方式是 deprecated 了，所以推荐采用在 `XXX.podspec` 中加入：

```shell
s.swift_version = ‘4.0’
```

另外还可能遇到没有明确的提示的 message，比如：

```shell
The 'source_files' pattern did not match any file` error
```

搜到的比较靠前的 stack overflow [问题](https://stackoverflow.com/questions/43073261/error-ios-file-patterns-the-source-files-pattern-did-not-match-any-file)中，被采纳的答案是 tag 不一致，要注意 `.podspec` 中的 version 和 git tag 一定要是对应的。

但其实我的问题是第二个没人投票的回答解决的 😂。因为我是将文件直接复制到 ChronosCore 项目里的，被复制的文件其实并不在 `'ChronosCore/Classes/**/*'` 这个路径下。移动一下物理位置就可以了。

# 利用 Travis CI 自动发布
前面提到 `script` 最后三行是为了将更新后的 Pod 直接发布到 CocoaPods 上。这里还需要给 TravisCI 提前设置一下 CocoaPods 的 access token，这样才能成功发布。
方式是，需要现在本地生成一个 token，代码如下：

```shell
$ pod trunk register YOUR-EMAIL --description='CI Automation'
[!] Please verify the session by clicking the link in the verification email that has been sent to YOUR-EMAIL
```

在邮箱中确定后，在 ` ~/.netrc` 中查看生成的 token

```shell
$ grep -A2 'trunk.cocoapods.org' ~/.netrc
machine trunk.cocoapods.org
  login YOUR-EMAIL
  password **********
```

然后在 TravisCI 对应的 repo 的设置中，加上这个环境变量：



![env](https://i.imgur.com/gGNYqLq.png)



# Done 🎉



![Imgur](https://i.imgur.com/rNxEXmW.png)

# Reference

- [Automated CocoaPod releases with CI](https://fuller.li/posts/automated-cocoapods-releases-with-ci/)
- [Publishing to CocoaPods from Travis](https://stackoverflow.com/a/31511532/9246748)
- [流程参考](https://www.jianshu.com/p/c5620ddca9ad)
- [Travis CI](https://www.objc.io/issues/6-build-tools/travis-ci/#troubleshooting-travis)
- [配置 Travis CI 使其不要 build 某些分支或者 tag](https://github.com/travis-ci/travis-ci/issues/8051)