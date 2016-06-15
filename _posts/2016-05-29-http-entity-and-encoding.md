---
title: "Http Entity & Encoding"
layout: post
date: 2016-05-29
permalink : /blogs/http-entity-and-encoding
tag:
- HTTP
blog: true
---


每天各种媒体对象经由HTTP传送，如图像，文本，影片以及软件程序等。HTTP需确保它的报文被正确传送。

HTTP要确保它承载的对象满足一下条件：

- 可以被正确识别(Content-Type首部说明媒体格式，Content-Language首部说明语言)，以便浏览器和其他客户端能正确处理内容
- 可以被正确解包(Content-Length首部和Content-Encoding首部)
- 是最新的(通过Etags和Cache-Control)
- 符合用户的需要(基于Accept系列的内容协商首部)
- 在网络上可以快速有效传输(通过范围请求、差异编码以及其他数据压缩方法)
- 完整到达，未被篡改(通过传输编码和Content-MD5校验和首部)


## Part 1: 实体(Entity)是什么

我们知道报文包含消息类型(状态行和请求行)，消息头(常用头域,响应头域,请求头域,实体头域)和消息主体(承载请求和响应的实体主体)

**实体包括实体头域和实体主体**；实体头域就是消息头中的实体头域；实体主体即是消息主体；有些响应只包含实体头域(有点绕)

### 实体头域

RFC2616中，实体头域包括以下部分：

```java
entity-header = Allow ; 
	| Content-Encoding ;
	| Content-Language ; 
	| Content-Length ; 
	| Content-Location ; 
	| Content-MD5 ; 
	| Content-Range ; 
	| Content-Type ; 
	| Expires ; 
	| Last-Modified ; 
	| extension-header
```

说明：

- Content-Type:实体承载的对象的类型；
- Content-Length:所传送实体主体的长度或大小；
- Content-Language:与所传送对象最相配的人类语言；
- Content-Encoding:对象数据所做的任意变换(如压缩)；
- Content-Location:备用位置，请求时可通过它获取对象;
- Content-Range:如果这是部分实体，这个首部说明它是整体的哪个部分；
- Content-MD5：实体主体内容的校验和；
- Last-Modified:所传输内容在服务器上创建或最后修改的日期时间；
- Expires:实体数据将要失效的日期时间；
- Allow:资源所允许的各种请求方式；
- Etag:这份文档特定实例的唯一验证码;
- Cache-Control:指出应该如何缓存该文档


(实际上Etag在RFC2616中是响应头域的部分，而Cache-Control是常用头域，但是对许多涉及实体的操作来说，是一个重要头部；也不用纠结于在哪个首部中，只要知道各自的功能就行了)。

PS：常用头域适用于请求消息和响应消息，但是不适合传输实体，只能用于传输消息，其实就是说常用头域是与实体无关的头域，或者说消息实体可以没有，但是常用头域还是能够存在头部中


### 实体主体

实体主体就是消息主体，如果有Content-Encoding首部的话，实体主体的内容就已经是被指定内容编码算法进行编码过的主体，第一个字节就是编码(如压缩)后的第一个字节


## Part 2: Content-Length:实体大小

Content-Length首部指示出报文中实体主体的字节大小。
注意，HTTP允许对实体主体的内容进行编码，比如可以使之更安全或进行压缩以节省空间。如果主体进行了内容编码, Content-Length就是**编码后的主体的字节长度**，而不是未编码的原始主体的长度。比如，对文本文件进行了gzip压缩后，Content-Length首部就是压缩后的大小，而不是原始大小

**除非使用了分块编码**(chunked)(因为分块编码的每一块都指明这一块的大小)，否则Content-Length首部就是实体主体的报文必须使用的；使用Content-Length首部是为了检测出服务器崩溃而导致的报文截尾，并对共享持久连接的多个报文进行正确分段。

Content-Length能用来检测报文结束和与持久连接搭配。

### 检测截尾

HTTP1.0中采用关闭连接的方法确定报文结束。但是没有Content-Length，客户端无法区分到底是报文结束时正常的连接关闭，还是报文传输中由于服务器崩溃而导致的连接关闭。此时客户端需要通过Content-Length检查报文截尾

报文截尾的问题对缓存代理服务器来说很重要。如果缓存服务器收到被截尾的报文却没有识别出截尾的话，可能会**存储不完整的内容**并**多次**使用它提供服务器。缓存代理服务器通常不会为没有显示Content-Length首部的HTTP主体做缓存，一次减小缓存已截尾报文的风险

错误的 Content-Length 比缺少 Content-Length 更严重

<h1 id="content-length-and-persistent-connection"></h1>

### Content-Length 与持久连接

Content-Length首部对于持久连接是必不可少的(除非进行了分块编码)。如果响应通过持久连接传输，说明可能有另一条HTTP响应跟随其后。因为**连接是持久的，客户端无法依赖连接关闭判断报文的结束**。客户端通过Content-Length就可以知道报文在何处结束，下一条报文从何处开始。如果没有Content-Length首部,HTTP应用程序就不知道某个实体主体在哪里结束，下一条报文从哪里开始。

但有一种情况下，使用持久连接时可以没有Content-Length首部，即采用**分块编码(chunked encoding)**时。在分块编码情况下，数据是分为一系列的块发送的，**每块都有各自的大小说明**。即使服务器在生成首部时不知道整个实体大小（因为使用分块编码时通常实体是动态生成的）,仍然可以使用分块编码传输若干已知大小的块



### 确定实体主体长度的规则

1. 如果特定HTTP报文类型**不允许带主体**，将忽略Content-Length首部。(如HEAD方法请求服务器发送等价的GET请求时出现的首部，但不要包括实体，因为GET响应中有Content-Length，所以HEAD响应中也有，但是需要忽略处理；那些规定不能带有实体主体的报文，不管带有什么首部字段，都必须在首部之后的第一个空行终止)
2. 如果报文中**包含传输编码**的Transfer-Encoding首部(即不采用默认的HTTP“恒等”编码，也就是不进行编码)，实体就应有一个称为零字节块的特殊模式结束，除非报文已经因连接关闭而结束
3. 如果报文含有Content-Length首部，并且报文类型允许有实体首部，而没有非恒等的Transfer-Encoding首部，那么Content-Length的值就是主体的值。如果收到的报文极有Content-Length首部字段又有非恒等的Transfer-Encoding首部,将忽略Content-Length，因为传输编码会改变实体主体的表示和传输方式
4. 如果报文**使用了multipart/byteranges**(多部分/字节范围)媒体类型，并且没有用Content-Length首部指出实体主体长度，那么多部分报文中每个部分都要说明它自己的大小。这种多部分类型是唯一的一种自定界的实体主体类型，因此除非发送方知道接收方可以解析它，否则不能发送这种媒体类型
5. 如果上面的规则**都不匹配**，实体就在连接关闭时结束。实际上，只有服务器可以使用连接关闭来只是报文的结束，客户端不能这样做，因为这样会导致服务器无法发回响应。(其实客户端可以使用半关闭，仅仅关闭输出，接受输入，但是因为有些服务器将客户的半关闭当做客户端要从服务器断开连接，HTTP协议中没有对连接管理进行良好的规范)

HTTP/1.1规范中建议对于带有主体但没有Content-Length首部的请求，服务器如果无法确定报文的长度，就应当发送400 Bad Request或411 Length Required响应。

Content-Length在前面已经讲了，后面会涉及其它相关的内容，如 
[内容编码和传输编码](#content-encoding)


## Part 3: Content-MD5:实体摘要

HTTP虽然建立在TCP/IP可靠传输协议之上实现，但仍有很多因素会导致报文的一部分在传输过程中被修改，如中间代理错误，不兼容的转码代理等。为检测实体主体的数据是否被不经意被修改，发送方可以在生成初始的主体时，生成一个数据的校验和，这样接收方就可以通过检验这个校验和来捕获所有意外的实体修改了。

只有产生响应的原始服务器可以计算并发送Content-MD5首部。中间代理和缓存不应该修改或添加这个首部，否则会与验证端到端完整性的目的冲突。

**Content-MD5首部的计算应当在对内容进行内容编码后(如果需要内容编码的话)，未进行任何传输编码之前进行**。为了验证报文的完整性，客户端必须先进行传输编码的解码，然后计算所得到的为进行传输编码的实体主体的MD5。

比如，如果一个文档使用gzip算法进行压缩，然后用分块编码发送，那么就对整个经gzip压缩的主体进行MD5计算，而不是某块进行验证。

除了检查报文完整性，MD5还可以用来作为散列表的关键字，用来快速定位文档，并消除重复内容的存储(也就是hash功能，没什么神奇的)

## Part 4: Content-Type: 媒体类型

Content-Type首部字段说明了实体主体的MIME类型。客户端应用程序使用MIME类型解释和处理其内容。MIME类型由一个主媒体类型后面跟一斜杠加上一个子类型组成。

如果不指明，默认是text/html

常用的媒体类型：
text/html、text/plain、text/css、text/javascript、image/gif、image/jpeg、multipart/form-data(支持上传文件)、application/json 、application/x-www-form-urlencoded(不支持上传文件，表单默认)


服务器针对不同的媒体类型，发送格式是不一样的，客户端需要根据不同的Content-Type来决定如何处理响应实体。或者客户端可以指明发送了不同类型媒体类型的数据。反正都是表明后面的实体是什么类型的。

比如multipart/form-data


这里需要注意的是，实体头域不仅仅只是用于响应中(否则就是响应头域了)，因此如Content-Type类型可以用在请求的首部中。比如经常使用的异步中指定 contentType:'application/x-www-form-urlencoded'，再如form表单中的enctype属性。其实想想也就知道了，在GET和POST请求中，客户端实际将发送的部分作为请求的部分，Content-Type首部告诉服务器，这些数据实际上是怎么组织起来的，以让服务器能够处理。


而实际上，RFC2616 7.2.1中也明确指出：

> 任一包含了实体主体的 HTTP/1.1 消息都应包括 Content-Type 头域以定义实体主体的媒体类型。如果只有媒体类型没有被 Content-Type 头域指定时，接收者可能会尝试猜测媒体类型，这通
过观察实体主体的内容并且/或者通过观察 URI 指定资源的扩展名。如果媒体类型仍然不知道，
接收者应该把类型看作”application/octec-stream”。

Content-Type首部还可以进一步说明文本的字符编码；如Content-Type: text/html;charset=utf-8

字符集、字符编码和MIME可以单独开一篇。



<h5 id="content-encoding"></h5>

## Part 5:内容编码

HTTP应用程序有时在发送之前会对内容进行编码，如把很大的HTML文档发送给慢速的客户端前，服务器可能会对文档进行压缩，以减少传输时间

**内容编码的过程**：

1. 服务器生成响应报文，其中有原始的Content-Type和Content-Length首部
2. 内容编码器创建编码后的报文，编码后同样有Content-Type但Content-Length可能不同，同时会加上Content-Encoding首部，这样应用程序就能进行解码
3. 接收程序得到编码后的报文，进行解码，获得原始报文


**内容编码的类型**：

- gzip：采用GUN zip压缩(RFC 1952文档)
- compress:采用 UNIX 的文件压缩程序(RFC 1951文档)
- deflate：实体用zlib格式压缩(RFC 1950文档)
- identity:不对实体进行编码，默认


**Accept-Encoding首部**：
为了防止服务器返回客户程序无法解码的内容，客户端可以将自己支持的内容编码列表放到Accept-Encoding首部发给服务器，如果不包含改首部，服务器会认为客户端能接受任何的编码方式(相当于发送Accept-Encoding:*)

编码列表中用逗号隔开每一个编码，同时可以给每个编码附带Q(质量)说明编码的优先级，Q的范围从0.0到1.0，0.0是客户端不想接受的编码，1.0是最希望接受的编码，*表示任何其它编码方式

* Accept-Encoding: compress, gzip
* Accept-Encoding: *
* Accept-Encoding: compress;q=0.5, gzip;q=1.0
* Accept-Encoding: gzip;q=1.0, identity;q=0.5, *;q=0



## Part 6:传输编码

内容编码是对报文主体进行的可逆转换。内容编码和内容的具体格式细节紧密相关。比如会用gzip压缩文本文件，但不是JPEG文件，因为JPEG文件用gzip压缩不是很好

传输编码也是作用于主体上的可逆转换，但是与内容格式无关。使用传输编码是为了改变报文中数据在网络中传输的方式。

传输编码是HTTP1.1引入的特性，服务器需要注意不要将经传输编码后的报文发送给非HTTP/1.1的应用程序。同样的，如果服务器收到无法理解经过传输编码的报文，应用501 Unimplemented状态回复。但HTTP/1.1应用程序都必须支持分块编码。。

**Transfer-Encoding首部**:
用于告知接收方为了可靠的传输报文，已经对其进行了何种编码，接收方能够根据该首部进行解码

对应的有 **TE首部**：用于请求首部中，告知服务器能够接受哪种传输编码扩展。

### 分块编码

HTTP规范中目前只定义了一种传输编码，就是分块编码(chunked)

分块编码将报文分割成若干个大小已知的块，块之间紧挨着发送，因此不需要发送前就知道整个报文的大小。

分块编码是传输编码，是报文的属性，而不是主体的属性

### 分块 与持久连接

对照[Content-Length 与持久连接](#content-length-and-persistent-connection)

当使用持久连接时，服务器在写主体前，必须知道主体大小并在Content-Length首部发送，但如果主体时动态创建的，就可能无法提前知道主体的长度，也就是Content-Length无法使用，此时就需要使用分块编码。

分块编码允许服务器将主体逐块发送，说明每块主体的大小就可以了。因为主体时动态创建的，服务器可以缓冲它的一部分，发送其大小和相应的块，服务器可以用大小为0的块表示主体结束信号。这样持久连接就能继续保持，为下一次响应准备。

具体的分块编码是怎样的，查看下面的图：

![](img/2016-05-29-chunked-encoding.png)

简单的说，就是每个分块包含一个长度值和这个分块的数据。长度用十六进制形式表示，并用CRLF与数据分隔开。分块中数据的大小以字节计算，不包括长度值与数据间的CRLF和分块结束的CRLF序列

客户端其实也可以发送分块给服务器，但是由于不知道服务器是否支持分块编码(因为服务器没有响应TE首部)，所以客户端必须做好服务器用 411 Length Required（需要Content-Length首部）响应来拒绝分块请求的准备



### 内容编码和传输编码的结合

![](img/2015-05-29-transfer-encoding-and-content-encoding.png)

### 传输编码的规则

当对报文使用传输编码时：

1. 传输编码集合中必须包括“chunked”。唯一的例外是使用关闭连接结束报文
2. 分块传输编码必须是最后一个作用到报文主体上的(也就是要在内容编码和MD5等计算之后)
3. 分块传输编码不能多次作用到一个报文主体上




## Part 7: Content-Range:范围请求

在[Http Cache](https://zebinlin.github.io/blogs/http-cache)中已经了解了客户端如何要求服务器只在客户端的资源副本失效时才发送副本，但是还可以进一步的细化：只请求文档的某一部分或者说是某一个范围。

这也是断点续传的原理所在：比如下载一个很大的电影，但是下载到一半，网络崩溃了，连接中断，如果要重新再来就不好了，所以通过使用Range和Content-Range实现下载某个范围内的数据。Range首部也在P2P（Peer-to-Peer）中得到广泛的应用，从不同的对等实体中同时下载多媒体文件的不同部分。



有了范围请求，客户端就可以请求获取曾经获取失败的实体的一个范围，来恢复下载该实体。当时，要保证这个范围的数据没有被修改过，比如被服务器删除这个文件，实体不在了，自然就恢复不了。

```java
GET /bigfile.html HTTP/1.1
Host: www.test.com
Range: bytes=4000-
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64)
```

客户端请求文档开头4000字节之后的部分，没有给出结束部分，因为客户端可能不知道实际的文档大小。
还可以用Range首部请求多个范围的。如为了加快下载速度，从不同服务器下载同一文档的不同部分。此时响应的也是一个实体，只是它是一个多部分主体以及加上 Content-Type: multipart/byteranges 首部


并不是所有服务器都支持范围请求，服务器可以通过在响应中包含Accept-Range首部的形式向客户端说明可以接受范围请求。它的值是计算范围的单位，通常是bytes

```java
HTTP/1.1 200 OK
Date: Sun, 29 May 2016 05:50:31 GMT
Server: WEBrick/1.3.1 (Ruby/2.2.4/2015-12-16)
Accept-Range: bytes
```

范围请求的响应代码是206，而不是200

![](img/2016-05-29-content-range.png)


## Part 8: 保证实体最新 & 差异编码

有时希望不要总是从服务器中获取，因为已经保证本地的缓存副本是有效的（还是新鲜的），使用了Cache-Control(或Expires)

差异编码就是希望在实体有一点点的改变就立即返回新的实体，也就是Etags和If-None-Match涉及新鲜度的检测(或If-Modified-Since和Last-Modified)


这个在[Http Cache](https://zebinlin.github.io/blogs/http-cache)已经描述了。




## Part X:这篇文章看过之后要学会什么

1. 实体包含什么
2. 区分传输编码和内容编码
3. 实体长度(Content-Length)的计算规则和局限性(不能处理动态创建的实体)
4. 内容编码(Content-Encoding)的类型，过程
5. 实体摘要(Content-MD5)和常用的媒体类型(Content-Type)
6. 传输编码(Transfer-Encoding)的用途
7. 对实体主体操作的先后顺序(先内容，再MD5，最后再分块)
8. 持久连接与Content-Length首部和分块的联系
9. 如何保证实体是最新的(Cache-Control,Etags,Exipres,If-None-Match...)
10. 范围请求(Content-Range)的应用 

总之就是加深对这几个首部的理解。