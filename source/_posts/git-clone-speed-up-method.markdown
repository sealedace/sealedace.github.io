---
layout: post
title: "git clone在GitHub的加速方法"
date: 2019-03-11 11:34:28 +0800
comments: true
categories: DevTips
---


本文的方法是在git配置为SSH方式的情况下，使用nc命令(netcat)实现，走Shadowsocks代理，从而加速从GitHub上clone代码。

> 引用的别人的回答：
> https://www.zhihu.com/people/kamikat/activities
>
> 知乎原文
> https://www.zhihu.com/question/27159393


如果坚持使用`http/https`方式，可以看上面链接里的方法，直接在git里全局配置

```shell
$ git config --global http.proxy http://mydomain\\myusername:mypassword@myproxyserver:8080/
```

或者这样，如果一次性使用代理

```shell
$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git --config "http.proxy=proxyHost:proxyPort"
```

如果想用SSH，建议使用下面的方法。

#### 前提条件

* netcat程序(MacOS/Linux系统自带的)
* 配置好的Shadowsocks
* git配置成SSH方式

```
  Host github.com
  User git
  ProxyCommand /usr/bin/nc -X 5 -x 127.0.0.1:1086 %h %p
  IdentityFile ~/.ssh/id_rsa
```

将上面这段内容**进行相应修改**后写进 ~/.ssh/config 文件（因为用到私钥认证所以带了`IdentityFile`选项）。
我这里 `127.0.0.1:1086`对应是`Shadowsocks`里配置的`SOCKS5`代理，`-X 5`代表使用`SOCKS5`。具体参见manpage。

这样SSH就会通过`nc`命令打开的管道连接到GitHub。

效果：

![](/images/77EC39D7-FCD8-44D8-ACD3-AC8FDACEB928.jpg)

