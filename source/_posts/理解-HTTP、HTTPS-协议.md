---
title: 理解 HTTP、HTTPS 协议
categories: 
  - Web
  - Network
tags: [HTTP, HTTPS]
abbrlink: f1215e4d
date: 2018-09-11 15:48:54
references:
  - name: 关于HTTP协议，一篇就够了
    url: http://www.cnblogs.com/ranyonsue/p/5984001.html
  - name: 计算机网络
    url: https://github.com/frank-lam/2019_campus_apply/blob/master/notes/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C.md
---

## 一、主要特点

1. **简单快速**：客户向服务器请求服务时，只需传送请求方法和路径。请求方法常用的有GET、HEAD、POST。每种方法规定了客户与服务器联系的类型不同。由于HTTP协议简单，使得HTTP服务器的程序规模小，因而通信速度很快。

2. **灵活**：HTTP允许传输任意类型的数据对象。正在传输的类型由`Content-Type`加以标记。

3. **无连接**：无连接的含义是限制每次连接只处理一个请求。服务器处理完客户的请求，并收到客户的应答后，即断开连接。采用这种方式可以节省传输时间。

4. **无状态**：HTTP协议是无状态协议。无状态是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大。另一方面，在服务器不需要先前信息时它的应答就较快。

## 二、URL、URI、URN区别

（1）`URI`，是uniform resource identifier，**统一资源标识符**，用来唯一的标识一个资源。

Web上可用的每种资源如HTML文档、图像、视频片段、程序等都是一个来URI来定位的。URI一般由三部组成：

- 访问资源的命名机制

- 存放资源的主机名

- 资源自身的名称，由路径表示，着重强调于资源。

（2）`URL`，是uniform resource locator，**统一资源定位器**，它**是一种具体的URI**，即URL可以用来标识一个资源，而且还指明了如何locate这个资源。

采用URL可以用一种统一的格式来描述各种信息资源，包括文件、服务器的地址和目录等。URL一般由三部组成：

- 协议(或称为服务方式)

- 存有该资源的主机IP地址(有时也包括端口号)

- 主机资源的具体地址。如目录和文件名等

（3）`URN`，uniform resource name，**统一资源命名**，是通过名字来标识资源，比如mailto:jitwxs@foxmail.com。

URI是以一种抽象的、高层次概念定义统一资源标识，而URL和URN则是具体的资源标识的方式。URI 包含 URL 和 URN，目前 WEB 只有 URL 比较流行，所以见到的基本都是 URL。

![URI-URL-URN](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911143626724.jpg)

## 三、请求报文

![请求报文](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911144241814.png)

![Get请求](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911144251564.png)

上图是一个GET请求的例子：

（1）第一部分：`请求行`，用来说明**请求类型**、**要访问的资源**以及**所使用的HTTP版本**。

（2）第二部分：`请求头`，紧接着请求行（即第一行）之后的部分，用来说明服务器要使用的**附加信息**。

从第二行起为请求头部，`HOST`将指出**请求的目的地**，`User-Agent`说明用户的浏览器信息。

（3）第三部分：`空行`，**请求头部后面的空行**是必须的。

即使第四部分的请求数据为空，也必须有空行。

（4）第四部分：请求数据也叫`主体`，可以**添加任意的其他数据**。

这个例子的请求数据为空。

![Post请求](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/2018091114430292.png)

上图是一个POST请求的例子，请求头中的`Content-Type`指明了主体部分的**MIME类型**，关于`Content-type`，主要有以下四种常见的取值：

（1）`application/x-www-form-urlencoded`

提交的数据按照 key1=val1&key2=val2 的方式进行编码，key 和 val 都进行了 URL 转码。

```http
POST http://www.example.com HTTP/1.1
Content-Type: application/x-www-form-urlencoded;charset=utf-8
 
title=test&sub%5B%5D=1&sub%5B%5D=2&sub%5B%5D=3
```

（2）`multipart/form-data`

我们使用表单上传文件时，必须让 form 的 enctyped 等于这个值。

```http
POST http://www.example.com HTTP/1.1
Content-Type:multipart/form-data; boundary=----WebKitFormBoundaryrGKCBY7qhFd3TrwA
 
------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="text"
 
title
------WebKitFormBoundaryrGKCBY7qhFd3TrwA
Content-Disposition: form-data; name="file"; filename="chrome.png"
Content-Type: image/png
 
PNG ... content of chrome.png ...
------WebKitFormBoundaryrGKCBY7qhFd3TrwA--
```

（3）`application/json`

告诉服务端消息主体是序列化后的 JSON 字符串，JSON 格式支持比键值对复杂得多的结构化数据。

```http
POST http://www.example.com HTTP/1.1
Content-Type: application/json;charset=utf-8
 
{"title":"test","sub":[1,2,3]}
```

（4）`text/xml`

以XML形式发送消息，随着json的流行，这种方式使用的越来越少。

```http
POST http://www.example.com HTTP/1.1
Content-Type: text/xml
 
<?xml version="1.0"?>
<methodCall>
    <methodName>examples.getStateName</methodName>
    <params>
        <param>
            <value><i4>41</i4></value>
        </param>
    </params>
</methodCall>
```

## 四、响应报文

![响应报文](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911151312444.png)

（1）第一部分：`状态行`，由**HTTP协议版本号**， **状态码**， **状态消息**三部分组成。

（2）第二部分：`响应头`，用来说明客户端要使用的一些附加信息

`Content-Type`指定了响应体MIME类型的HTML(text/html)

（3）第三部分：`空行`，响应头后面的空行是必须的

（4）第四部分：`响应体`，服务器返回给客户端的文本信息。

![200响应](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911151325189.png)

![404响应](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911151336276.png)

## 五、HTTP状态

服务器返回的 **响应报文** 中第一行为状态行，包含了状态码以及原因短语，用来告知客户端请求的结果。

| 状态码 | 类别                             | 原因短语                   |
| ------ | -------------------------------- | -------------------------- |
| 1XX    | Informational（信息性状态码）    | 接收的请求正在处理         |
| 2XX    | Success（成功状态码）            | 请求正常处理完毕           |
| 3XX    | Redirection（重定向状态码）      | 需要进行附加操作以完成请求 |
| 4XX    | Client Error（客户端错误状态码） | 服务器无法处理请求         |
| 5XX    | Server Error（服务器错误状态码） | 服务器处理请求出错         |

下面列举出一些常见的HTTP状态码：

- **100 Continue** ：表明到目前为止都很正常，客户端可以继续发送请求或者忽略这个响应。

- **200 OK**

- **204 No Content** ：请求已经成功处理，但是返回的响应报文不包含实体的主体部分。一般在只需要从客户端往服务器发送信息，而不需要返回数据时使用。

- **301 Moved Permanently** ：永久性重定向

- **302 Found** ：临时性重定向

- **400 Bad Request** ：请求报文中存在语法错误。

- **401 Unauthorized** ：该状态码表示发送的请求需要有认证信息（BASIC 认证、DIGEST 认证）。如果之前已进行过一次请求，则表示用户认证失败。

- **403 Forbidden** ：请求被拒绝，服务器端没有必要给出拒绝的详细理由。

- **404 Not Found**

- **500 Internal Server Error** ：服务器正在执行请求时发生错误。

- **503 Service Unavailable** ：服务器暂时处于超负载或正在进行停机维护，现在无法处理请求。

## 六、GET请求与POST请求区别

- GET传送的数据量**较小**，不能大于2KB。POST传送的数据量**较大**，一般被默认为不受限制。

- GET提交请求的数据会**附在URL之后**，POST提交把提交的数据**放置在HTTP包的包体**中。因此，GET提交的数据会在地址栏中显示出来，而POST提交，地址栏不会改变。

- GET用于**信息获取**，POST是用于修**改服务器上的资源**的请求。 GET和 POST只是一种传递数据的方式，GET也可以把数据传到服务器，他们的本质都是发送请求和接收结果，只是组织格式和数据量上面有差别。

- POST请求相对GET请求是**安全**的，这里安全的含义仅仅是指是非修改信息。

## 七、HTTP的无连接

HTTP协议是无状态的（stateless），指的是协议对于事务处理**没有记忆能力**，服务器不知道客户端是什么状态。也就是说，打开一个服务器上的网页和上一次打开这个服务器上的网页之间没有任何联系。HTTP是一个**无状态的面向连接**的协议，**无状态不代表HTTP不能保持TCP连接，更不能代表HTTP使用的是UDP协议（无连接）**。 

缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大。另一方面，在服务器不需要先前信息时它的应答就较快。 

## 八、HTTPS

**HTTP 有以下安全性问题：**

- 使用明文进行通信，内容可能会被窃听；
- 不验证通信方的身份，通信方的身份有可能遭遇伪装；
- 无法证明报文的完整性，报文有可能遭篡改。

`HTTPS`（Hyper Text Transfer Protocol over Secure Socket Layer），是以安全为目标的HTTP通道，简单讲是HTTP的安全版。

HTTPS 并不是新协议，而是**让 HTTP 先和 SSL（Secure Sockets Layer）通信，再由 SSL 和 TCP 通信**。也就是说 HTTPS 使用了隧道进行通信。通过使用 SSL，HTTPS 具有了**加密（防窃听）**、**认证（防伪装）**和**完整性保护（防篡改）**。

![https](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911154039309.jpg)

**（1）对称密钥加密**

对称密钥加密（Symmetric-Key Encryption），**加密和解密使用同一密钥**。

- 优点：运算速度快；
- 缺点：无法安全地将密钥传输给通信方。

![对称密钥加密](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911154106813.png)

Alice 给 Bob 发送数据时，把数据用对称加密后发送给 Bob，发送过程中由于对数据进行了加密，因此即使有人窃取了数据也没法破解，因为它不知道密钥是什么。但是同样的问题是 Bob 收到数据后也一筹莫展，因为它也不知道密钥是什么，那么 Alice 是不是可以把数据和密钥一同发给 Bob 呢。当然不行，一旦把密钥和密钥一起发送的话，那就跟发送明文没什么区别了，因为一旦有人把密钥和数据同时获取了，密文就破解了。所以对称加密的密钥配是个问题。如何解决呢，公钥加密是一个办法。

**（2）非对称密钥加密（公钥加密）**

非对称密钥加密，又称公开密钥加密（Public-Key Encryption），**加密和解密使用不同的密钥**。

公开密钥所有人都可以获得，通信发送方获得接收方的公开密钥之后，就可以使用公开密钥进行加密，接收方收到通信内容后使用私有密钥解密。

非对称密钥除了用来加密，还可以用来进行签名。因为私有密钥无法被其他人获取，因此通信发送方使用其私有密钥进行签名，通信接收方使用发送方的公开密钥对签名进行解密，就能判断这个签名是否正确。

- 优点：可以更安全地将公开密钥传输给通信发送方；
- 缺点：运算速度慢。

![非对称密钥加密](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911154134355.png)

还是以Alice 给 Bob 发送数据为例，公钥加密算法由接收者 Bob 发起：

1. Bob 生成公钥和私钥对，私钥自己保存，不能透露给任何人。

2. Bob 把公钥发送给 Alice，发送过程中即使被人窃取也没关系

3. Alice 用公钥把数据进行加密，并发送给 Bob，发送过程中被人窃取了同样没关系，因为没有配对的私钥进行解密是没法破解的

4. Bob 用配对的私钥解密。

虽然公钥加密解决了密钥配送的问题，但是你没法确认公钥是不是合法的，Bob 发送的公钥你不能肯定真的是 Bob 发的，因为也有可能在 Bob 把公钥发送给 Alice 的过程中出现中间人攻击，把真实的公钥掉包替换。而对于 Alice 来说完全不知。还有一个缺点是它的运行速度比对称加密慢很多。

**（3）HTTPS采用的加密方式**

HTTPS采用混合的加密机制，**使用非对称密钥加密用于传输对称密钥来保证安全性，之后使用对称密钥加密进行通信来保证效率**。

![https加密方式](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201809/20180911154203951.png)

## 九、HTTP和HTTPS的区别

- http是HTTP协议运行在`TCP`之上。所有传输的内容都是**明文**，客户端和服务器端都无法验证对方的身份。 

- https是HTTP运行在`SSL/TLS`之上，SSL/TLS运行在`TCP`之上。所有传输的内容都经过**加密**，加密采用`对称加`密，但对称加密的密钥用服务器方的证书进行了`非对称加密`。

- http和https使用的是完全不同的连接方式用的端口也不一样,前者是**80**，后者是**443**。 
