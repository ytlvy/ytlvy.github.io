---
layout: post
title: "Websocket Tunnel Over Cloud IDE"
date: 2015-05-31 09:29:57 +0800
comments: true
categories: FQ
---
### bbs
> https://groups.google.com/forum/#!forum/go-gost

### 原理
> 在平台中建立websocket服务器，将数据通过websocket传输给客户端实现一个tunnel

### cloud Ide
- Cloud 9：https://c9.io
- Koding：https://koding.com/
- Nitrous：https://lite.nitrous.io/

<!--more-->

### server
```
$ wget https://bintray.com/artifact/download/ginuerzh/gost/gost_1.2_linux_amd64.tar.gz
$ tar zxf gost_1.2_linux_amd64.tar.gz
$ cd gost_1.2_linux_amd64/
$ ./gost -ws //no output
```

### client
> wget https://bintray.com/ginuerzh/gost/gost/

#### c9
```
./gost -L :8899 -S demo-project-gostwebsocket.c9.io -ws

```
<!-- 
```
./gost -L :8899 -S ytlvy-ytlvy.c9.io -ws
./gost -L :8899 -S demo-project-ytlvy.c9.io -ws  //now
```
-->


#### Koding
```
52.74.10.210:8080
```

#### nitrousbox
```
./gost -L :8899 -S appName-appid.apne1.nitrousbox.com:8080 -p 'password' -m `encrypt type` -ws
```

<!-- 
```
./gost -L :8899 -S ytlvy-228959.apne1.nitrousbox.com:8080 -ws
./gost -L :8899 -S ytlvy-228959.apne1.nitrousbox.com:8080 -m=rc4-md5 -p=123456 -ws
```
-->
### behind proxy
如果处在http代理环境中(代理要支持websocket)，可增加上层代理(-P参数)：
```
./gost -L :8899 -S demo-project-gostwebsocket.c9.io -P your_proxy_ip:port -ws
```

### 安全处理 
> 更改gost文件名，防止自动重启

### Android设置
> gost支持作为shadowsocks服务器运行(-ss参数)，这样就可以让android手机通过shadowsocks(影梭)使用代理了。

#### 相关参数：
```
-ss    开启shadowsocks模式
-sm   设置shadowsocks加密方式(默认为rc4-md5)
-sp    设置shadowsocks加密密码(默认为ginuerzh@gmail.com)
```
当无-ss参数时，-sm, -sp参数无效。
以上三个参数对服务端无效。

#### 相关命令：
##### 服务端：
> 无需特殊设置
shadowsocks模式只与客户端有关，与服务端无关。

##### 客户端：
```
./gost -L :8899 -S demo-project-gostwebsocket.c9.io -ws -sm=rc4-md5 -sp=ginuerzh@gmail.com -ss
```
> 在手机的shadowsocks软件中设置好服务器(运行gost电脑的IP)，端口(8899)，加密方法和密码就可以使用了。
注：shadowsocks模式与正常模式是不兼容的，当作为shadowsocks模式使用时(有-ss参数)，浏览器不能使用


### encryt type
> websocket加密功能需要客户端和服务端gost版本都为1.2版及以上

#### 目前支持的加密方法
> tls, aes-128-cfb, aes-192-cfb, aes-256-cfb, des-cfb, bf-cfb, cast5-cfb, rc4-md5, rc4, table

#### Client端设置
1. Client端通过-m参数设置加密方式，默认为不加密(-m参数为空)。
2. 如果设置的加密方式不被支持，则默认为不加密。
3. 当设置的加密方式为tls时，-p参数无效。
4. 当设置的加密方式为非tls时，通过-p参数设置加密密码，且不能为空，默认密码为ginuerzh@gmail.com；-p参数必须与Server端的-p参数相同。

#### Server端设置
1. Server端通过-m参数设置加密方式，默认为不加密(-m参数为空)。
2. 如果设置的加密方式不被支持，默认为不处理 (此情形被视为错误，须用户自行改正)。
3. 如果没有设置加密方式(-m参数为空)，则由client端控制加密方式，即client端可通过-m参数指定Server端使用哪种加密方式。
4. 如果设置了加密方式(-m参数不为空)，client端必须使用与Server端相同的加密方式。
5. 当设置的加密方式为tls时，-p参数无效；-key参数可手动指定公钥文件，-cert参数可手动指定私钥文件，如果未指定，则使用默认的公钥与私钥。
6. 当设置的加密方式为非tls时，-key，-cert参数无效；通过-p参数设置加密密码，且不能为空，默认密码为ginuerzh@gmail.com















