---
layout: post
title: 使用 HyperApp 自建机场
date: 2018-02-15 11:10 +0800
---

* TOC
{:toc}
前几天折腾了好久酸酸乳的问题，因为某些情况梯子特别不稳。幸好前两天买了一个新的服务备用。不过为了 Plan B/C/D，还是自己弄了俩 VPS 装了 SS(R) 和 V2Ray，配合 HyperApp 还是很简单的。 

HyperApp 的作者有写一篇非常详细的新手图文教程（[地址](https://www.hyperapp.fun/zh/proxy/get-started.html)），这里主要记录一下配置 ShadowsocksR 和 V2Ray 过程中可能需要注意的。使用的是 Google Cloud Platform 和 AWS 提供的 VPS（都是可以免费使用一年左右的）。

准备
===

HyperApp，iOS 上一个基于 SSH 和 Docker 的自动化部署工具，虽然应用没有 Bitmap 多，但是更符合国情 😂 。感觉最方便的一点是，如果添加了新的服务器，可以在 HyperApp 上使用已经配置好的 app 直接安装。

GCP 账号，新用户可申请 300 刀且使用时长为1年的抵扣金，真是相当良心了。（用过几天后算了下大概每天 1 刀，自用的话还是有点贵的）不过虽然我选了 asia-east 的机房，但是创建实例后发现还是在美国的，延迟大概在 200~300ms。

AWS 账号，新用户可以直接免费用一年，只要选择 free tier 的配置即可。学生的话还可以申请教育优惠，最多有110刀 credits。我选的东京的机房延迟在 100~200ms，甚至有时候还能冲进100，自用的话也是足够了。

其它的 VPS，可以考虑的还有 Vultr，搬瓦工，Linode 等等，虽然感觉墙一发威都是重灾区，但是小众一点的也需要考虑运营商跑路的问题，另外还需要注意 HyperApp 不支持 OpenVZ 架构的 VPS。

过程
===

使用 HyperApp 自己搭梯子一般有三步：HyperApp 连接到服务器，安装配置各个爱国应用，配置各平台的客户端。

第一步没什么好说的了，毕竟作者都简化到扫二维码的地步了 = = 这一步需要注意的是最好安装一下 BBR，HyperApp 有现成的脚本；另外要补充的就只有 GCE 和 AWS 的防火墙规则。

## 防火墙规则

GCE 如果使用 80 和 443 之外的端口，需要在[这里](https://console.cloud.google.com/networking/firewalls/list)添加规则，规则填写如下：

```
名称：随便输入一个名称
目标：选择 网络中的所有示例
来源过滤：0.0.0.0/0 // anywhere
协议和端口：指定的协议和端口 下面输入 tcp;udp:端口号 //对应科学上网应用中的端口号
```

AWS 默认只开启了 SSH 的 22 端口，直接 `ping` public IP 都 `ping` 不通 😂 在 AWS 的管理面板中，创建新的 security groups。然后选中使用的 instance，在`'action-networking-change security groups'` 选择新建的 security groups。

```
// ping
Type: Custom ICMP rule
Protocol: Echo Request
Port: N/A
Source: anywhere, i.e: 0.0.0.0/0

// 自定义端口号
Type: Custom TCP/UDP rule
Protocol: TCP/UDP
Port: 自定义的端口号
Source: anywhere, i.e: 0.0.0.0/0
```

![security groups](https://i.imgur.com/j1CEXYH.jpg)

## 部署科学上网应用

目前主要用的科学上网应用有 SS，SSR 和 V2Ray，SS(R) 的安装配置都很简单，只有 V2Ray 稍微有点麻烦。

### V2Ray

原版教程[在此](https://www.hyperapp.fun/zh/proxy/V2Ray.html)，不能更完整了。所以只是整理和补充下过程。

V2Ray 有三种传输方式：TCP，WebSocket 和 mKCP，这也是相比 SS(R) 麻烦一些的地方之一；另外还有就是用 V2Ray 需要 SSL 证书。考虑到手机上的客户端用的是 Shadowrocket，它不支持 mKCP 的方式，所以我只试了前面两个方案。

### 生成 SSL 证书

>  参考：[《如何自动生成 SSL 证书》](https://www.hyperapp.fun/zh/SSL.html)

首先需要一个域名（可以参考[这篇](https://www.hyperapp.fun/zh/Get-Domain.html)申请一年免费的域名），并将域名解析到服务器上。接下来要自动生成域名的 SSL 证书，HyperApp 提供了两个方案。一是 Nginx SSL Support；二是使用 cerbot。

如果使用 Nginx SSL Support，在安装完 Nginx 和 Nginx SSL Support 后，再安装 V2Ray。注意配置的时候先不要勾选 `Enable TLS`，然后在最底部的 Nginx and SSL options 中，根据前面教程里的说明填入域名有关信息。填写完后选择安装 V2Ray，过几分钟在 `/srv/docker/certs/` 下应该就会出现证书了（*Nginx SSL Support 会自动监听其它应用里面 SSL Support 相关设置，并自动生成证书。*）

使用 cerbot 就更简单了，按[此教程](https://www.hyperapp.fun/zh/developer/certbot.html)填写即可。

>  如果没有正确生成证书，可能要检查一下域名是否已经正确解析到服务器上，**正确解析**需要保证服务器的 80/443 端口是开启的，这又回到上面的防火墙规则的部分。

# 客户端使用

iOS 上推荐使用 Shadowrocket，它支持 SS/SSR，V2Ray（除了 KCP）。对于 SS(R)，可以用 HyperApp 生成 QR 让 Shadowrocket 扫描导入配置信息。

![qr](https://i.imgur.com/CbiMxfE.jpg)

macOS 可以用 ShadowsocksX-NG/Surge (SS) 或者 ShadowsocksX-NG-R8 (SSR)，同样可以扫描二维码导入。

V2Ray 需要单独的客户端。macOS 可以直接 `brew cask install v2rayx` 安装。不过那个客户端非常简陋，而且还总是连接不上 = = 暂时还没找到原因。

另外如果有 Surge 的话，可以考虑采取在终端后台启动 V2Ray 然后用 Surge 连过去的方式，具体步骤可以参考[这里](http://telegra.ph/bookshop-v2-cmd-now-02-12)。