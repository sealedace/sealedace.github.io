---
layout: post
title: "git在GitHub的加速方法"
date: 2019-03-11 11:34:28 +0800
comments: true
categories: DevTips
---



> 在国内git加速太常用了，这里记录一下。<br>http 和 SSH 两种方法不一样，分别处理。


笔者的系统环境：

```
操作系统: macOS Mojave 10.14.3
Shadowsocks http代理: 0.0.0.0:1087
Shadowsocks SOCKS5代理: 127.0.0.1:1086
```

---

## git使用http协议

* 如果想把git全局都走某个代理：


```shell
$ git config --global http.proxy http://mydomain\\myusername:mypassword@myproxyserver:8080/
```

拿笔者的环境为例：

```shell
$ git config --global http.proxy 127.0.0.1:1087
```

或者直接编辑`~/.gitconfig`文件，添加下面的内容

```
[http]
      proxy = 127.0.0.1:1087
```

* 如果只对某个git域名走代理，比如GitHub：

```shell
$ git config --global http.https://github.com.proxy http://127.0.0.1:1087
```

或者直接编辑`~/.gitconfig`文件，添加下面的内容

```
[http "https://github.com"]
       proxy = 127.0.0.1:1087
```

* 如果只是一次性使用代理：

```shell
$ git clone https://xxxx.git --config "http.proxy=proxyHost:proxyPort"
```

拿笔者的环境为例：

```shell
$ git clone https://xxxx.git --config "http.proxy=127.0.0.1:1087"
```


---

## git使用SSH协议

使用nc命令(netcat)实现，走Shadowsocks代理，从而加速从GitHub上clone代码。

> nc命令macOS/Linux系统自带，如果没有自行安装。


```
  Host github.com
  User git
  ProxyCommand /usr/bin/nc -X 5 -x 127.0.0.1:1086 %h %p
  IdentityFile ~/.ssh/id_rsa
```

将上面这段内容**进行相应修改**后写进 ~/.ssh/config 文件（因为用到私钥认证所以带了`IdentityFile`选项）。
我这里`127.0.0.1:1086`对应是`Shadowsocks`里配置的`SOCKS5`代理，`-X 5`代表使用`SOCKS5`。具体参见manpage。

这样SSH就会通过`nc`命令打开的管道连接到GitHub。


![](/images/77EC39D7-FCD8-44D8-ACD3-AC8FDACEB928.jpg)

---

> 参考：<br>https://gist.github.com/evantoli/f8c23a37eb3558ab8765<br>http://cms-sw.github.io/tutorial-proxy.html


