# HTTP协议
## 5层网络模型
![5层网络模型](./images/2019-03-08-13-24-10.png)

网络分层 | 作用 | 简单理解(有利于记忆)
---------|----------|---------
 物理层 | 主要作用定义物理设备如何传输数据 | 网络物理硬件之间通过电缆如何安全传输数据（网卡）
 数据链路层 | 在通信的实体间建立数据链路连接 | 使用软件在网络硬件上建立数据链路连接，让软件和网络硬件进行通信（Hub）
 网络层 | 为数据在节点之间传输创建逻辑链路 | 网络节点A如何找到网络节点B的地址（路由器）
 传输层 | 向用户提供可靠的端到端(End To End)的服务，TCP、UDP | 网络节点A上软件和网络节点B之间的软件建立传输数据的连接，包括拆包、分包、组装包等功能
 应用层 | 为应用软件提供很多服务 | 比如：HTTP将TCP进行封装，提供接口让应用程序之间方便调用，隐藏了传输层的拆包、分包等功能

 ## HTTP协议的发展历史

阶段 | 特点 
---------|----------
 HTTP/0.9 | 只有一个GET命令；没有header；服务器发送完毕，就关闭TCP连接
 HTTP/1.0 | 增加了很多命令（POST\PUT\DELETE等），增加了status code和header，支持多字符集，权限、缓存等
 HTTP/1.1 | 增加了长链接和pipeline，连接需要3次握手，pipeline是串行传输的，增加了host
 HTTP/2.0 | 所有数据以二进制传输（HTTP/1.1和之前版本都是是字符串明文传输的），pipeline是并行传输的，头信息压缩以及推送
 HTTPS | HTTP/1.1基础上传输的字符串明文进行非对称加密解密

 ## HTTP连接三次握手

 ![](./images/2019-03-08-14-02-58.png)
 
 连接三次握手，避免由于网络通信延迟而导致服务器创建无效的链接
 可以使用winshake抓包软件

 ## URI URL和URN

 URI：统一资源标识符，Uniform Resource Identifier
> 用来表示唯一互联网上的信息资源
> 包括了URL和URN

 URL：统一资源定位器，Uniform Resource Locator
> http://user:pass@host.com:80/path?query=string#hash
> 不同的协议也会有不同URL格式，比如HTTP、FTP、邮件

URN：永久统一资源定位符

## HTTP报文格式

![](./images/2019-03-08-14-32-26.png)

## CORS跨域请求


## 缓存头Cache-Control

Cache-Control特性：
1. public
    > HTTP返回的响应内容经过的中间节点（包括所有代理服务器、发出该HTTP请求的浏览器）都会被缓存起来
2. private
    > 只有发出该HTTP请求的浏览器才可以缓存HTTP返回的响应内容
3. no-cache
    > HTTP返回的响应都不会被缓存
4. max-age=`<seconds>`
    > 缓存的时间长度，如果过期，则重新从服务器中请求
5. s-maxage=`<seconds>`
    > 缓存的时间长度，但是这个只能由代理服务器使用，浏览器不能使用的。
6. max-stale
    > 浏览器可以接收超出max-age时间的响应消息，max-stale在请求设置中有效，在响应设置中无效
    max-age和max-stale在请求中同时使用的情况下，缓存的时间可以为max-age和max-stale的和


## 缓存验证Last-Modified和ETag

