---
title: 浅谈HTTP协议中的Content-Type
date: 2016/3/12 21:45:54 
tags: 
     - HTTP协议
     - 学习笔记
---
{% cq %}
这周工作中遇到一个奇葩问题，公司IIS服务器下Log日志记录的JSON被转成了乱码，觉得很奇葩是因为我同一套WebService部署了很多遍，为什么其他服务下日志记录的好好的，你就不行，后来仔细观察下，妈个鸡，这不是乱码，他只不过把JSON中一些中文和特殊字符给我转成了Unicode格式，那么问题又回来了，谁让你瞎他瞄转换的(╯°口°)╯(┴—┴！我细细的吸了一口水，发现事情并不简单（背后肯定有着肮脏的PY交易）。最后的原因你肯定也猜到了，就是HTTP协议搞的鬼，既然如此，就再整理一下HTTP协议吧  
 {% endcq %}

## 协议概述
HTTP协议是互联网应用最为广泛的网络协议，最初由W3C(万维网协会)发布制定，目前的标准版本是1.1。协议内容非常简单（扶额思考），最初是为了定义客户端与服务端进行交互的标准格式，但由于其移植性太强，现在广泛应用于各种Web应用前后台交互
HTTP协议位于TCP/IP协议簇的应用层，直接依赖于处于传输层的TCP协议，所以和TCP类似，处理端对端（进程对进程）通信

在HTTP 0.9和1.0版本使用的非持续连接，而在1.1后改为持续连接，意味着不必为每个web对象创建一个新的连接，一个连接可以传送多个对象
<!-- more -->
以下是wiki对HTTP协议的概述
>HTTP是一个客户端终端（用户）和服务器端（网站）请求和应答的标准（TCP）。通过使用Web浏览器、网络爬虫或者其它的工具，客户端发起一个HTTP请求到服务器上指定端口（默认端口为80）。我们称这个客户端为用户代理程序（user agent）。应答的服务器上存储着一些资源，比如HTML文件和图像。我们称这个应答服务器为源服务器（origin server）。在用户代理和源服务器中间可能存在多个“中间层”，比如代理服务器、网关或者隧道（tunnel）。

>尽管TCP/IP协议是互联网上最流行的应用，HTTP协议中，并没有规定必须使用它或它支持的层。事实上，HTTP可以在任何互联网协议上，或其他网络上实现。HTTP假定其下层协议提供可靠的传输。因此，任何能够提供这种保证的协议都可以被其使用。因此也就是其在TCP/IP协议族使用TCP作为其传输层。

>通常，由HTTP客户端发起一个请求，创建一个到服务器指定端口（默认是80端口）的TCP连接。HTTP服务器则在那个端口监听客户端的请求。一旦收到请求，服务器会向客户端返回一个状态，比如"HTTP/1.1 200 OK"，以及返回的内容，如请求的文件、错误消息、或者其它信息。

其协议内容分为**请求（Request）**及**响应（Response）**，但两者的格式都是类似的，所以以下按照格式逐级说明

## 协议格式
HTTP协议格式如下：

1. **行**: 包括请求行已经响应行
	1. 请求行：包括一个请求最基本的信息，请求方式、请求路径(可选)、以及遵循的协议版本
		
			格式：[请求方法]SPS[请求路径(可选)]SPS[协议版本]
			示例：GET /path1/path2/fileName.html HTTP/1.1
	
	2. 响应行：包含响应状态以及服务端HTTP版本协议
			
			格式：[协议版本]SPS[状态码]SPS[状态描述]
			示例：HTTP/1.1 304 Not Modified


2. **头**: 包括通用头、请求头、响应头和实体头，代表请求（响应）的各种属性（部分是浏览器、代理主动添加的），以键值队形式保存
		
		格式：[属性名称]:[属性值]
		示例: Connection: keep-alive
			 Accept-Encoding: gzip, deflate, sdch
			 Accept-Language: zh-CN,zh;q=0.8
  
3. **（空行）** 

4. **其它消息体（可选）**：通常是PostData
## 行

### 请求行
这里我要唠叨一下题外话，虽然请求（Request）这个词叫做请求。从请求发起者的角度来看其含义不仅仅是请求，其实她只是表示一个动作的发起，可以是请求，可以是强迫，可以是要求。而如果站在响应方的角度来看，你请求，我不响应你，你要求，我不答应你，这时候也不叫请求了，叫做乞求。所以这个命名是略有歧义的，如果W3C跟我一样是个深度命名强迫症患者，肯定不会起这个名字，但现在也没法改名，因为改名字是很难[向下兼容](https://zh.wikipedia.org/wiki/%E5%90%91%E4%B8%8B%E5%85%BC%E5%AE%B9 "wiki-向下兼容")的（自己开发中经常遇到名字不准确强行改名，结果踩了一堆雷）

*言归正传*，请求看起来就像一个**对某某说一个流行动词**，为什么咋么说呢？  

首先，请求头的请求方式就相当于这个动词，像**下面给你吃**，而请求路径就是某某了；  

其次，为什么是某某呢，因为这个某某你可能根本就认识啊，什么？认识？我的朋友啊，知道名字就算认识了吗，别忘了你知道人家的**名字**，连**姓**什么都不知道，你这样天真，怎么找的到女朋友！  

再次，为什么是**流行词**，因为版本号啊，版本高的听的懂版本低的话，但是反之就不行，你对外婆说下面给你吃和对女友说下面给你吃，结果能一样吗？就相当于你和同龄人是一个版本，你们有着共同词汇，但是外婆属于初始版本，你虽然可以理解外婆的话，但外婆不能全部理解你的话（以及行为）(苦笑)

#### 请求方法
HTTP规定的*请求方法*一共有8种，同时规定HTTP服务器至少应该事先GET和HEAD方法，其它方法都是可选的。最常用的是POST和GET。此外，请求方法是可扩展的

具体请见以下wiki引用
>HTTP/1.1协议中共定义了八种方法（也叫“动作”）来以不同方式操作指定的资源：
 
>> OPTIONS：这个方法可使服务器传回该资源所支持的所有HTTP请求方法。用'*'来代替资源名称，向Web服务器发送OPTIONS请求，可以测试服务器功能是否正常运作。  
>> HEAD：与GET方法一样，都是向服务器发出指定资源的请求。只不过服务器将不传回资源的本文部分。它的好处在于，使用这个方法可以在不必传输全部内容的情况下，就可以获取其中“关于该资源的信息”（元信息或称元数据）。  
>> GET：向指定的资源发出“显示”请求。使用GET方法应该只用在读取数据，而不应当被用于产生“副作用”的操作中，例如在Web Application中。其中一个原因是GET可能会被网络蜘蛛等随意访问。参见安全方法  
>> POST：向指定资源提交数据，请求服务器进行处理（例如提交表单或者上传文件）。数据被包含在请求本文中。这个请求可能会创建新的资源或修改现有资源，或二者皆有。  
>> PUT：向指定资源位置上传其最新内容。  
>> DELETE：请求服务器删除Request-URI所标识的资源。  
>> TRACE：回显服务器收到的请求，主要用于测试或诊断。  
>> CONNECT：HTTP/1.1协议中预留给能够将连接改为管道方式的代理服务器。通常用于SSL加密服务器的链接（经由非加密的HTTP代理服务器）。 
 
>方法名称是区分大小写的。当某个请求所针对的资源不支持对应的请求方法的时候，服务器应当返回状态码405（Method Not Allowed），当服务器不认识或者不支持对应的请求方法的时候，应当返回状态码501（Not Implemented）。

#### 请求路径
HTTP1.1之后请求路径分为相对和绝对两种形式，如果请求通过代理服务器，则可以使用相对路径
### 响应行
- 1XX 等待后续请求
- 2XX 请求成功
- 3XX 重定向
- 4XX 客户端错误
- 5XX 服务端错误


状态码和状态详细描述可以参照 [RFC官方文档](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) 
## 头
### 通用头(General Header)
针对通用头的概念目前很模糊，首先看下官方的解释

![通用头](http://7xr9ql.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20160309162858.png "RFC")

可以看到官方在HTTP1.1中声明了9个通用头，分别是Cache-Control、Connection、Date、Pragma、Traile、Transfer-Encoding、Upgrade、Via、Warning。通用头必须双方都支持才生效，否则会被接收方当做实体头。如果通信双方都承认的一般头字段，也可以当成(语义上的)通用头。


从官方文档可以看出，虽然他声明了几个标准通用头，但是通用头的拓展性是非常大的，只要双方承认并使用，就可以当做通用头。实际使用中，像谷歌和火狐的浏览器调试工具都无视了通用头，详见下图。

![](http://7xr9ql.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20160309182535.png "谷歌调试工具Network格式")

![](http://7xr9ql.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20160309182551.png "火狐FirebugNetwork格式")

谷歌自带的调试工具甚至自己定了一个General，里面放了请求地址，请求方法，响应状态和终端地址

具体通用头释义戳[这里](http://www.ecdoer.com/post/http-seo.html)

### 请求头（Request Header）
官方对于请求头的解释

![](http://7xr9ql.com1.z0.glb.clouddn.com/QQ%E6%88%AA%E5%9B%BE20160311170618.png "RFC HTTP1.1 5.3节")

网上流传的一段翻译
>请求头用于说明是谁或什么在发送请求、请求源于何处，或者客户端的喜好及能力。服务器可以根据请求头部给出的客户端信息，试着为客户端提供更好的响应。请求头域可能包含下列字段Accept、Accept-Charset、Accept- Encoding、Accept-Language、Authorization、From、Host、If-Modified-Since、If-Match、If-None-Match、If-Range、If-Range、If-Unmodified-Since、Max-Forwards、Proxy-Authorization、Range、Referer、User-Agent。对请求头域的扩展要求通讯双方都支持，如果存在不支持的请求头域，一般将会作为实体头域处理。


在HTTP/1.1协议中，所有的请求头，除Host外，都是可选的。请求头种类非常多，还在不断扩展，具体请求头释义戳[这里](http://www.ecdoer.com/post/http-seo.html)或直接浏览[RCF HTTP1.1](https://tools.ietf.org/html/rfc2616)

### 响应头（Request Header）
官方对于响应头的解释
![](http://7xr9ql.com1.z0.glb.clouddn.com/234.png "RFC HTTP1.1 6.2节")
网上流传的一段翻译
>响应头向客户端提供一些额外信息，比如谁在发送响应、响应者的功能，甚至与响应相关的一些特殊指令。这些头部有助于客户端处理响应，并在将来发起更好的请求。响应头域包含Age、Location、Proxy-Authenticate、Public、Retry- After、Server、Vary、Warning、WWW-Authenticate。对响应头域的扩展要求通讯双方都支持，如果存在不支持的响应头域，一般将会作为实体头域处理。

具体响应头释义戳[这里](http://www.ecdoer.com/post/http-seo.html)或直接浏览[RCF HTTP1.1](https://tools.ietf.org/html/rfc2616)

### 实体头（Entity Header）
官方对于实体头的解释
![](http://7xr9ql.com1.z0.glb.clouddn.com/123.png "RFC HTTP1.1 7.1节")

网上流传的一段翻译
>实体头部提供了有关实体及其内容的大量信息，从有关对象类型的信息，到能够对资源使用的各种有效的请求方法。总之，实体头部可以告知接收者它在对什么进行处理。请求消息和响应消息都可以包含实体信息，实体信息一般由实体头域和实体组成。实体头域包含关于实体的原信息，实体头包括信息性头部Allow、Location，内容头部Content-Base、Content-Encoding、Content-Language、Content-Length、Content-Location、Content-MD5、Content-Range、Content-Type，缓存头部Etag、Expires、Last-Modified、extension-header。

具体实体头释义戳[这里](http://www.ecdoer.com/post/http-seo.html)或直接浏览[RCF HTTP1.1](https://tools.ietf.org/html/rfc2616)，这里只重点介绍一下坑了我一次的Content-Type
#### Content-Type
Content-Type**本来**是用来定义浏览器如何正确对响应流的处理，他是通过响应流的媒体类型来确定的。

	格式： Content-Type: [媒体类型]/[具体格式]   
	如： Content-Type: image/jpeg

媒体类型目前是IANA(The Internet Assigned Numbers Authority，互联网数字分配机构)定义的8个大类，分别是：

* application— (比如: application/vnd.ms-excel.)
* audio (比如: audio/mpeg.)
* image (比如: image/png.)
* message (比如,:message/http.)
* model(比如:model/vrml.)
* multipart (比如:multipart/form-data.)
* text(比如:text/html.)
* video(比如:video/quicktime.)

不同浏览器对于Content-Type的处理方式不能会不同，我引用来一个2010年统计的图作为参考

![](http://7xr9ql.com1.z0.glb.clouddn.com/231.png )

但是现在他的作用不仅仅如此，在常用的HTTP请求模拟工具PostMan和HTTPRequester中都可以看到他的身影

![](http://7xr9ql.com1.z0.glb.clouddn.com/post.png "PostMan界面")

![](http://7xr9ql.com1.z0.glb.clouddn.com/987546.png "HTTPRequester界面")

那么Content-Type在这里究竟起什么作用，笔者搜寻变半天终于找到一个比较满意的答案，当别人无法给你答案时，你只能自己摸索出一个答案。

以下是博主在*Asp.Net WebService*环境下使用PostMan做的实验

1. Content-Type: Text
![](http://7xr9ql.com1.z0.glb.clouddn.com/1111.png)

![](http://7xr9ql.com1.z0.glb.clouddn.com/12.png)
2. Content-Type: text/plain
![](http://7xr9ql.com1.z0.glb.clouddn.com/21.png)

![](http://7xr9ql.com1.z0.glb.clouddn.com/22.png)
3. Content-Type: application/x-www-form-urlencoded 
![](http://7xr9ql.com1.z0.glb.clouddn.com/31.png)

![](http://7xr9ql.com1.z0.glb.clouddn.com/32.png)
4. Content-Type: application/json
![](http://7xr9ql.com1.z0.glb.clouddn.com/41.png)

![](http://7xr9ql.com1.z0.glb.clouddn.com/42.png)

![](http://7xr9ql.com1.z0.glb.clouddn.com/43.png "JSON返回值")   
5. Content-Type: application/javascript、application/xml、text/xml、text/html  
返回：请求格式无效(WebService不支持)

通过实验可以看出，当PostMan设置Content-Type为Text时，其实Request中是没有Content-Type这一属性的，但是服务端自动解析Content-Type为默认配置（text/plain,utf-8）；当Content-Type为application/x-www-form-urlencoded时，实体在请求发出时就会被转译为unicode格式；但我有一个**疑问**就是为什么Content-Type为application/json时，请求输入流字节长度是正确的，但是转成字符串之后为空串，而且请求实体也绝对没有丢（即图中参数a和参数b）

## 结论 
Content-Type为application/x-www-form-urlencoded时，客户端会在请求发出前把实体进行转译，而之所以有些服务下能够正常接收入参大概是因为他们的Content-Type设置为text/plain的缘故。Content-Type这个属性出现在请求头中只是声明实体的格式，让服务端准备好解析手段

其它属性如果后续有使用到还会继续补充内容
## 参照
HTTP 协议漫谈:<http://blog.jobbole.com/88199/>

HTTP响应头和请求头信息对照表：<http://tools.jb51.net/table/http_header>

RFC 2016 HTTP/1.1:<https://tools.ietf.org/html/rfc2616>

四种常见的 POST 提交数据方式:<https://imququ.com/post/four-ways-to-post-data-in-http.html>

SodaZhcn的简书-Http协议详解：<http://www.jianshu.com/p/e83d323c6bcc>

HTTP头信息解读【SEO必知】：<http://www.ecdoer.com/post/http-seo.html>

yangfch3的笔记 HTTP协议详解：<https://www.zybuluo.com/yangfch3/note/167490>