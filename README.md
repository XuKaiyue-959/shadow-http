# shadow-http
![GitHub](https://img.shields.io/github/license/mahaoqu/shadow-http.svg?style=flat-square)
A Shadowsocks-like http proxy server.

Shakdow-http是一个与Shadowsocks兼容的HTTP隧道程序，提供加密的端对端TCP隧道通信。

程序由两部分组成，分别是在隧道客户端shadow-client和隧道服务器shadow-server。隧道客户端负责监听本地的https代理端口，处理HTTP隧道请求。收到请求后，它会使用密码加密信息并连接隧道服务器，由隧道服务器来连接代理目标。

一般隧道客户端运行在本机上，而隧道服务器运行在远方服务器上。这样，本机所有在公开网络上的通信内容都是使用加密传输的，可以保证较强的安全性。

这样通过两重转发，我们就在本地客户端和远程服务器之间建立起了一个虚拟的TCP连接通道。

```
+-------------+  HTTP    +-------------+            +--------------+           +---------------+
|  本地客户端， | CONNECT  |  隧道客户端   |  加密传输   |  隧道服务器    |  TCP连接  |  远程服务器，   |
|  如浏览器等。 | <----->  |   client    |  <----->   |    server    |  <----->  |  如google.com |
+-------------+          +-------------+            +--------------+           +---------------+
```


### 使用方式：
客户端使用命令行参数配置本机监听端口，远程连接地址和端口，加密方式和密码。目前仅支持AES-256-CFB加密。
```
python client.py [-h] -i HOST -p PORT [-l LOCAL] -c PASSWORD [-m METHOD] [-v]
```

要求运行环境Python 3.4以上版本。

依赖于PyCryptodome库，可以使用pip进行安装。
```
pip install pycryptodome
```
Windows下可以尝试：
```
pip install pycryptodomex
```


### 本地代理配置：

在本地运行服务后，可以使用

* Windows：

  控制面板 - Internet选项 - “连接”属性卡 - 局域网设置 - “代理服务器”设置 - 高级

  在“安全(Secure)”一栏中填写环回地址（127.0.0.1）和监听的本地端口。

  ![windows-client](pics/windows-client.png)

* Mac版本：

  系统偏好设置… - 网络 - 高级… -“代理”选项卡 - 安全网页代理(HTTPS)

  在安全网页代理服务器一栏中填写环回地址（127.0.0.1）和监听的本地端口。

  ![mac-client](pics/mac-client.png)

## 协议

### HTTP隧道协议

可以使用HTTP的CONNECT方法来启动一个Web隧道。通过这种方法，可以建立任意的TCP的连接隧道。

本地客户端（如浏览器）会首先发送一条类似这样的连接请求：
```
CONNECT www.google.com:443 HTTP/1.1
Host: www.google.com:443
Proxy-Connection: keep-alive
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/72.0.3626.121 Safari/537.36
```
请求的起始行中表明了要连接的目标地址和端口号。

如果连接成功，隧道客户端应做出回应：
```
HTTP/1.1 200 Connection Established
```

这时在本地请求端口和远程的443端口之间就建立起了一个隧道，双方向隧道发送的任何信息都会被隧道转发到另一方。

### Shadowsocks协议
Shadowsocks协议是隧道客户端和隧道服务器之间采取的通信协议。在隧道客户端和隧道服务器的一个连接开始时，会先传输一个头部，来通知客户端开启远程的连接。

```
+------+----------+----------+
| ATYP | DST.ADDR | DST.PORT |
+------+----------+----------+
|  1   | Variable |    2     |
+------+----------+----------+
```
* ATYP 字段：address type 的缩写，取值为：
    * 0x01：IPv4
    * 0x03：域名
    * 0x04：IPv6
* DST.ADDR 字段：destination address 的缩写，取值随 ATYP 变化：
    * ATYP == 0x01：4 个字节的 IPv4 地址
    * ATYP == 0x03：1 个字节表示域名长度，紧随其后的是对应的域名
    * ATYP == 0x04：16 个字节的 IPv6 地址
* DST.PORT 字段：目的服务器的端口，占2字节。

隧道服务器需要解析这个头部，并建立起对应的连接。
之后，对于来自隧道客户端的数据，隧道服务器会解密后发向远程服务器，
对于来自远程服务器的数据，隧道服务器会加密后发向隧道客户端。

## 设计与实现

### 模型

在隧道客户端中，我们需要维护一个客户和隧道服务器间的两重连接。使用一个连接类（connection）来抽象每一个连接。

使用事件驱动模型设计程序。这里使用python标准库中的`selectors`模块。
我们可以通过selector的register方法，为每一个套接字注册其函数，之后可以使用modify方法修改或者使用unregister方法取消注册。

每一个连接是一个状态机。在不同的状态下，注册的事件处理函数会处理连接过程的不同部分。一个连接的过程如下：

客户端：

1. 初始状态。此时等待本地套接字可读。
    1. 本地可读，如果读到完整的HTTP请求，就解析它，获得远程地址和端口。尝试连接远程套接字，并转换到等待远程连接状态。如果解析失败就销毁这个连接。
    2. 同上，如果没有读到HTTP头结束，就把已读到的内容加入缓冲区。
    3. 如果本地套接字提前终止，就销毁这个连接。

2. 等待远程连接状态。此时等待远程套接字可写，本地套接字可读。
    1. 本地可读。说明本地出现了错误。此时销毁这个连接。
    2. 远程套接字变为可写。说明连接已经建立。向本地套接字发送HTTP回复。并向远程套接字写入加密后的Shadow协议头，然后进入连接已经建立状态。
    3. 远程套接字既可读又可写。此时说明远程套接字发生了错误。此时销毁这个连接。

3. 连接已经建立状态。此时等待远程套接字可读，本地套接字可读。
    1. 远程可读。读入之后解密，写入本地套接字。
    2. 本地可读。读入之后加密，写入远程套接字。


服务器：
1. 初始状态。此时等待本地套接字可读。
    1. 解析读到的头部，如果是地址，或者是域名且在缓冲中有对应的地址，就试图连接并跳到第三个状态。
    2. 如果是域名，就在缓存中没有找到，就发送DNS请求并跳到第二个状态。

2. 等待DNS连接状态。此时等待本地套接字可读，DNS套接字可读。
    1. 本地可读。把读到的内容加入缓冲区的尾部。
    2. DNS套接字可读，验证完整性后尝试连接并进入第三个状态。

3. 等待远程连接状态。此时等待远程套接字可写，本地套接字可读。
    1. 本地可读。把读到的内容加入缓冲区的尾部。
    2. 远程可写。说明连接已经建立，进入连接已经建立状态。

4. 连接已经建立状态。此时等待远程套接字可读，本地套接字可读。
    1. 远程可读。读入之后加密，写入本地套接字。
    2. 本地可读。读入之后解密，写入远程套接字。


### 错误处理

* 超时处理：暂无
* 其他错误处理：使用套接字读写时，可能出现ConnectionError。
可能是对方重置了连接或者提前关闭。在Python中，`ConnectionError`是`BrokenPipeError`,
`ConnectionAbortedError`,`ConnectionRefusedError`和`ConnectionResetError`的父类。
只需要捕获这个异常并销毁对应的连接即可。

## 参考资料：
1. [PyCryptodome 3.8.0 documentation](https://pycryptodome.readthedocs.io/en/latest/)
2. 《HTTP权威指南》8.5节
3. [Shadowsocks 源码分析——协议与结构](https://loggerhead.me/posts/shadowsocks-yuan-ma-fen-xi-xie-yi-yu-jie-gou.html)
4. [selectors — 高级 I/O 复用库 — Python 3.7.3 文档](https://docs.python.org/zh-cn/3/library/selectors.html)

## 注意事项：
1. 为每一个函数写注释。
