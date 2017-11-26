---
layout: post
title: Cross GFW in Terminal
date: 2015-12-07 9:42:20 -0500
categories: tools
---

## Proxychains
升级到El Capitan之前的神器，使用方式可以参考这篇文章 [^1]。不过他主要讲的使用SS的配置方式，如果是VPN的话， [ProxyList]下面应该填写

```bash
http    192.168.89.3    8080    justu    hidden
//协议名称    服务器地址    端口号    用户名    密码
```

具体格式可以参考`proxychains.conf`文件里的`ProxyList Format Example`

## Polipo
由于终端里不能走SS的Socks5协议，只有Http[s]，所以需要将Http[s]的请求都转到Socks5上，这一步需要一个polipo的小软件，Mac的话通过Homebrew就可以安装，可以参考@clowwindy大神的介绍 [^2]。

### Steps

- 开Shadowsocks，在网络中查看本地的端口号
- 终端里输下面的命令启动polipo

```bash
polipo socksParentProxy=localhost:1080
// 这个端口号是SS的本地端口号
```

- 然后在要执行的命令前加

```bash
http_proxy=http://localhost:8123 https_proxy=http://localhost:8123
// 8123是polipo的默认端口号
```

## Tips

- 在.zshrc或者.bashrc中添加

```bash
alias gfw=http_proxy=http://localhost:8123 https_proxy=http://localhost:8123
```

- 设置完之后，来一发`gfw curl ip.cn`爽一爽 🚀

## UPDATE: 05/16/2017

刚换了[fish shell](https://fishshell.com)，它的语法和Bash不太一样。如果想带着环境变量执行命令(Run commands with environment variables)，需要加上`env `。比如上面的命令在fish中应该是：

```bash
env http_proxy=http://localhost:8123 https_proxy=http://localhost:8123 curl ip.cn
```

为了方便使用，可以写个function。在`fish_prompt.fish`里，添加下面的function：

```bash
function gfw
    env http_proxy=http://localhost:8123 https_proxy=http://localhost:8123 $argv
end
```

这样就可以在fish中接着用`gfw curl ip.cn`了。

---

[^1]: http://www.dreamxu.com/proxychains-ng/
[^2]: https://github.com/shadowsocks/shadowsocks/wiki/Convert-Shadowsocks-into-an-HTTP-proxy
