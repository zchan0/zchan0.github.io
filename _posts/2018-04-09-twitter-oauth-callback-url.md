---
layout: post
title: Twitter OAuth1 Callback URL 的一个坑
date: 2018-04-09 00:54 +0800
---

* TOC
{:toc}
在完成 Twitter OAuth1 的 Demo 的时候遇到一个比较奇葩的问题，在此记录一下。 

问题
===

一开始的问题是，在 [Application Settings](http://dev.twitter.com/apps/) - Callback URL 中留的空白，因为 caption 说执行 OAuth1 过程中会根据 `oauth_callback` 的值动态设置。然而结果是，一直采用 PIN code 的方式，即通过授权后，Twitter 返回了一串 PIN code。

我的第一反应是哪一步流程出错了，导致 Twitter 以为我采用了`oob` 的验证方式。然而排查一通后，虽然确实有几个流程上的小问题，但是并不该影响回调。

一通搜索之后，发现你需要在 Application Settings - Callback URL 中设定一个值，哪怕是 `dummy.com`，才能确保 Twitter 会开始跳转，而不是显示 PIN code。（然而文档上并没有提及这一点。。

> BTW, 在 Application Settings 里修改了一次 Callback URL 之后，就不能再将其置为空了。如果新设置的 URL 不符合格式要求，会保留前一个设定。

然后可能出现的问题是，对于 mobile application 来说，需要跳转到自己的 app 中，一般应该要定义一个 custom URI scheme，比如 `myapp://callback`。但这并不符合 Twitter 对于 Callback URL 的格式要求，Twitter 只接受 HTTP(S) 的 URL。

解决
===

参考这个[回答](https://stackoverflow.com/a/2401135/9246748)，一个解决办法是，让 `https://example.com/callback` 跳转到 custom URI。

当时想到了 OAuthSwift 这个库，也是近几年完成的，应该也会遇到这个问题，于是就去研究了下它的跳转方式，发现它的 Twitter example 用到的 Callback URL 是 `http://oauthswift.herokuapp.com/callback/twitter`，在 Safari 中（Chrome 不会显示）打开这个网址，会得到下面的页面：

![oauthswift-redirect](https://i.imgur.com/SvO1D70.png)

果然 OAuthSwift 是通过这个方式来跳转的！通过网址还能得知，这个跳转是部署在 Heroku 上的。

知道了思路接下来的步骤就简单了。不管是部署在什么地方，要完成**两件事**：

- 跳转到 custom URI，比如 `alamofire-oauth1://alamofire-oauth1/allback/twitter`
- 带上 `query` 里的值（不然没有 token 啦

下面是 PHP 实现并部署在 Heroku 上的一个例子。

```php
<?php
function Redirect($url, $permanent = false) {
    if (headers_sent() === false) {
        header('Location: ' . $url, true, ($permanent === true) ? 301 : 302);
    }

    exit();
}

$query = $_SERVER['QUERY_STRING'];

Redirect('alamofire-oauth1://alamofire-oauth1/callback/twitter' . '?' . $query, true);
?>
```

> 写的时候因为想顺便试试 Heroku 就直接写了，现在想想应该扔 Github 上应该也是可行的，可以利用 Github Pages，具体就不试啦。

剩下的，只要将 Callback URL 设置为上面部署好的 URL 即可。