---
title: Nginx 缓存使用官方指南
categories:
  - 服务器
  - Nginx
typora-root-url: ..
abbrlink: cd6f6a52
date: 2018-05-11 14:11:17
copyright_author: 张帆
---

我们都知道，应用程序和网站一样，其性能关乎生存。但如何使你的应用程序或者网站性能更好，并没有一个明确的答案。代码质量和架构是其中的一个原因，但是在很多例子中我们看到，你可以通过关注一些十分基础的应用内容分发技术，来提高终端用户的体验。其中一个例子就是实现和调整应用栈（application stack）的缓存。这篇文章，通过几个例子来讲述如何使用Nginx缓存。此外，结尾处还列举了一些常见问题及解答。

## 基础

一个 `web 缓存`坐落于客户端和“原始服务器（origin server）”中间，它保留了所有可见内容的拷贝。如果一个客户端请求的内容在缓存中存储，则可以直接在缓存中获得该内容而不需要与服务器通信。这样一来，由于 web 缓存距离客户端“更近”，就可以提高响应性能，并更有效率的使用应用服务器，因为服务器不用每次请求都进行页面生成工作。

在浏览器和应用服务器之间，存在多种“潜在”缓存，如：客户端浏览器缓存、中间缓存、内容分发网络（CDN）和服务器上的负载平衡和反向代理。缓存，仅在`反向代理`和`负载均衡`的层面，就对性能提高有很大的帮助。

举个例子说明，去年，我接手了一项任务，这项任务的内容是对一个加载缓慢的网站进行性能优化。首先引起我注意的事情是，这个网站差不多花费了超过1秒钟才生成了主页。经过一系列调试，我发现加载缓慢的原因在于页面被标记为不可缓存，即为了响应每一个请求，页面都是动态生成的。由于页面本身并不需要经常性的变更，并且不涉及个性化，那么这样做其实并没有必要。为了验证一下我的结论，我将页面标记为每5秒缓存一次，仅仅做了这一个调整，就能明显的感受到性能的提升。第一个字节到达的时间降低到几毫秒，同时页面的加载明显要更快。

并不是只有大规模的内容分发网络（CDN）可以在使用缓存中受益——**缓存还可以提高负载平衡器、反向代理和应用服务器前端web服务的性能**。通过上面的例子，我们看到，缓存内容结果，可以更高效的使用应用服务器，因为不需要每次都去做重复的页面生成工作。此外，**Web 缓存还可以用来提高网站可靠性**。当服务器宕机或者繁忙时，比起返回错误信息给用户，不如通过配置Nginx将已经缓存下来的内容发送给用户。这意味着，网站在应用服务器或者数据库故障的情况下，可以保持部分甚至全部的功能运转。

下一部分讨论如何安装和配置Nginx的基础缓存（Basic Caching）。

## 如何安装和配置基础缓存

我们只需要两个命令就可以启用基础缓存： [proxy_cache_path](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_path) 和 [proxy_cache](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache)。`proxy_cache_path`用来**设置缓存的路径和配置**，`proxy_cache`用来**启用缓存**。

```nginx
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m 
                 use_temp_path=off;
server {
    ...
    location / {
        proxy_cache my_cache;
        proxy_pass http://my_upstream;
    }
}
```

`proxy_cache_path` 命令中的参数及对应配置说明如下：

 1. 用于缓存的本地磁盘目录是 `/path/to/cache/`。

 2. `levels` 在 `/path/to/cache/` 设置了一个**两级层次结构的目录**。**将大量的文件放置在单个目录中会导致文件访问缓慢**，所以针对大多数部署，我们**推荐使用两级目录层次结构**。如果 `levels` 参数没有配置，则 Nginx 会将所有的文件放到同一个目录中。

 3. `keys_zone` 设置一个`共享内存区`，该内存区**用于存储缓存键和元数据**，有些类似计时器的用途。将键的拷贝放入内存可以使 Nginx 在不检索磁盘的情况下快速决定一个请求是 `HIT` 还是 `MISS`，这样大大提高了检索速度。一个1MB的内存空间可以存储大约8000个 key，那么上面配置的10MB内存空间可以存储差不多80000个 key。

 4. `max_size` 设置了**缓存的上限**（在上面的例子中是10G）。这是一个可选项；如果不指定具体值，那就是允许缓存不断增长，占用所有可用的磁盘空间。当缓存达到这个上线，处理器便调用 `cache manager` 来移除最近最少被使用的文件，这样把缓存的空间降低至这个限制之下。

 5. `inactive` 指定了**项目在不被访问的情况下能够在内存中保持的时间**。在上面的例子中，如果一个文件在60分钟之内没有被请求，则缓存管理将会自动将其在内存中删除，不管该文件是否过期。该参数默认值为10分钟（10m）。注意，非活动内容有别于过期内容。Nginx **不会自动删除由缓存控制头部指定的过期内容**（本例中 Cache-Control:max-age=120）,**过期内容只有在 `inactive`指定时间内没有被访问的情况下才会被删除**。如果过期内容被访问了，那么 Nginx 就会将其从原服务器上刷新，并更新对应的 inactive 计时器。

 6. Nginx 最初会将注定写入缓存的文件先放入一个临时存储区域， `use_temp_path=off` 命令指示 Nginx **将在缓存这些文件时将它们写入同一个目录下**。我们强烈建议你将参数设置为 `off` 来避免在文件系统中不必要的数据拷贝。

 7. 最终， `proxy_cache` 命令启动缓存那些URL与location部分匹配的内容（本例中，为`/`）。你同样可以将 `proxy_cache` 命令添加到 `server` 部分，这将会将缓存应用到所有的那些 `location` 中未指定自己的 `proxy_cache` 命令的服务中。

## 陈旧总比没有强

Nginx [内容缓存](https://www.nginx.com/products/content-caching-nginx-plus/)的一个非常强大的特性是：**当无法从原始服务器获取最新的内容时，Nginx可 以分发缓存中的陈旧**（`stale`，编者注：即过期内容）内容。

这种情况一般发生在关联缓存内容的原始服务器宕机或者繁忙时。比起对客户端传达错误信息，Nginx 可发送在其内存中的陈旧的文件。Nginx 的这种代理方式，为服务器提供额外级别的容错能力，并确保了在服务器故障或流量峰值的情况下的正常运行。为了开启该功能，只需要添加[proxy_cache_use_stale](http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=1.14624247.1568941527.1438257987#proxy_cache_use_stale)命令即可：

```nginx
location / {
    ...
    proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
}
```

按照上面例子中的配置，当 Nginx 收到服务器返回的 error，timeout 或者其他指定的 5xx 错误，并且在其缓存中有请求文件的陈旧版本，则会将这些陈旧版本的文件而不是错误信息发送给客户端。

## 缓存微调

Nginx 提供了丰富的可选项配置用于缓存性能的微调。下面是使用了几个配置的例子：

```nginx
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m
                 use_temp_path=off;
server {
    ...
    location / {
        proxy_cache my_cache;
        proxy_cache_revalidate on;
        proxy_cache_min_uses 3;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;
        proxy_pass http://my_upstream;
    }
}
```

这些命令配置了下列的行为：

 1. [proxy_cache_revalidate](http://nginx.org/r/proxy_cache_revalidate?_ga=1.80437143.1235345339.1438303904) 指示Nginx在刷新来自服务器的内容时使用GET请求。如果客户端的请求项已经被缓存过了，但是在缓存控制头部中定义为过期，那么 Nginx 就会在GET请求中包含 `If-Modified-Since` 字段，发送至服务器端。这项配置可以节约带宽，因为对于 Nginx 已经缓存过的文件，服务器只会在该文件请求头中 `Last-Modified` 记录的时间内被修改时才将全部文件一起发送。
 2. [proxy_cache_min_uses](http://nginx.org/r/proxy_cache_min_uses?_ga=1.82886422.1235345339.1438303904) 设置了**在 Nginx 缓存前，客户端请求一个条目的最短时间**。当缓存不断被填满时，这项设置便十分有用，因为这确保了只有那些被经常访问的内容才会被添加到缓存中。该项默认值为1。
 3. [proxy_cache_use_stale](http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=1.13131319.1235345339.1438303904#proxy_cache_use_stale) 中的 `updating` 参数告知 Nginx **在客户端请求的项目的更新正在原服务器中下载时发送旧内容，而不是向服务器转发重复的请求**。第一个请求陈旧文件的用户不得不等待文件在原服务器中更新完毕。陈旧的文件会返回给随后的请求直到更新后的文件被全部下载。

 4. 当 [proxy_cache_lock](http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=1.86844376.1568941527.1438257987#proxy_cache_lock) 被启用时，**当多个客户端请求一个缓存中不存在的文件（或称之为一个MI SS），只有这些请求中的第一个被允许发送至服务器**。其他请求在第一个请求得到满意结果之后在缓存中得到文件。如果不启用 `proxy_cache_lock` ，则所有在缓存中找不到文件的请求都会直接与服务器通信。

## 跨多硬盘分割缓存

使用 Nginx，不需要建立一个 `RAID`（磁盘阵列）。如果有多个硬盘，Nginx 可以用来在多个硬盘之间分割缓存。下面是一个基于请求URI跨越两个硬盘之间均分缓存的例子：

```nginx
proxy_cache_path /path/to/hdd1 levels=1:2 keys_zone=my_cache_hdd1:10m max_size=10g 
                 inactive=60m use_temp_path=off;
proxy_cache_path /path/to/hdd2 levels=1:2 keys_zone=my_cache_hdd2:10m max_size=10g 
                 inactive=60m use_temp_path=off;
split_clients $request_uri $my_cache {
    50%    "my_cache_hdd1";
    50%    "my_cache_hdd2";
}
server {
    ...
    location / {
        proxy_cache $my_cache;
        proxy_pass http://my_upstream;
    }
}
```

上例中的两个 `proxy_cache_path` 定义了两个缓存（`my_cache_hdd1` 和 `my_cache_hd22`）分属两个不同的硬盘。[split_clients](http://nginx.org/en/docs/http/ngx_http_split_clients_module.html?_ga=1.10291573.1568941527.1438257987)配置部分指定了请求结果的一半在 my_cache_hdd1 中缓存，另一半在 my_cache_hdd2 中缓存。

基于 `$request_uri`（请求URI）变量的哈希值决定了每一个请求使用哪一个缓存，对于指定 URI 的请求结果通常会被缓存在同一个缓存中。

## 常见问题解答

### 可以检测 Nginx 缓存状态吗？

可以，使用 [add_header](http://nginx.org/en/docs/http/ngx_http_headers_module.html?&_ga=1.253128009.1568941527.1438257987#add_header) 指令：

```nginx
add_header X-Cache-Status $upstream_cache_status;
```

上面的例子中，在对客户端的响应中添加了一个 `X-Cache-Status` HTTP 响应头，下面是[$upstream_cache_status](http://nginx.org/en/docs/http/ngx_http_upstream_module.html?&_ga=1.18350585.1568941527.1438257987#var_upstream_cache_status)的可能值：

 1. `MISS`——响应在缓存中找不到，所以需要在服务器中取得。这个响应之后可能会被缓存起来。

 2. `BYPASS`——响应来自原始服务器而不是缓存，因为请求匹配了一个 `proxy_cache_bypass`（见下面[我可以在缓存中打个洞吗？](https://www.nginx.com/blog/nginx-caching-guide/#caching-guide-faq-hole-punch)）。这个响应之后可能会被缓存起来。

 3. `EXPIRED`——缓存中的某一项过期了，来自原始服务器的响应包含最新的内容。

 4.  `STALE`——内容陈旧是因为原始服务器不能正确响应。需要配置 `proxy_cache_use_stale`。

 5.  `UPDATING`——内容过期了，因为相对于之前的请求，响应的入口（`entry`）已经更新，并且 `proxy_cache_use_stale` 的 `updating` 已被设置。

 6. `REVALIDATED`——[proxy_cache_revalidate](http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=1.86328024.1568941527.1438257987#proxy_cache_revalidate)命令被启用，Nginx检测得知当前的缓存内容依然有效（`If-Modified-Since`或者`If-None-Match`）。

 7. `HIT`——响应包含来自缓存的最新有效的内容。 

### Nginx 如何决定是否缓存？

默认情况下，Nginx 需要考虑从原始服务器得到的 `Cache-Control` 标头。当在响应头部中 `Cache-Control` 被配置为 `Private`，`No-Cache`，`No-Store` 或者 `Set-Cookie`，Nginx 不进行缓存。Nginx 仅仅缓存 `GET` 和 `HEAD` 客户端请求。你也可以参照下面的解答覆盖这些默认值。

### Cache-Control 头部可否被忽略？

可以，使用 `proxy_ignore_headers` 命令。如下列配置：

```nginx
location /images/ {
    proxy_cache my_cache;
    proxy_ignore_headers Cache-Control;
    proxy_cache_valid any 30m;
    ...
}
```
Nginx会忽略所有 `/images/` 下的 `Cache-Control` 头。[proxy_cache_valid](http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=1.250506950.1568941527.1438257987#proxy_cache_valid)命令强制规定缓存数据的过期时间，如果忽略 `Cache-Control` 头，则该命令是十分必要的。Nginx 不会缓存没有过期时间的文件。

### 当在头部设置了S et-Cookie 之后 Nginx 还能缓存内容吗？

可以，使用 `proxy_ignore_headers` 命令，参见之前的解答。

### Nginx 能否缓存 POST 请求？

可以，使用 [proxy_cache_methods](http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=1.244788933.1568941527.1438257987#proxy_cache_methods) 命令：

```nginx
proxy_cache_methods GET HEAD POST;
```

这个例子中可以缓存 POST 请求，其他附加的方法可以依次列出来的。

### Nginx 可以缓存动态内容吗？

可以，提供的 `Cache-Control` 头部可以做到。缓存动态内容，甚至短时间内的内容可以减少在原始数据库和服务器中加载，可以提高第一个字节的到达时间，因为页面不需要对每个请求都生成一次。

### 我可以再缓存中打个洞（Punch a Hole）吗？

可以，使用 `proxy_cache_bypass` 命令：

```nginx
location / {
    proxy_cache_bypass $cookie_nocache $arg_nocache;
    ...
}
```

这个命令定义了哪种类型的请求**需要向服务器请求而不是尝试首先在缓存中查找**。有些时候又被称作`在内存中“打个洞”`。

在上面的例子中，Nginx 会针对 `nocache cookie` 或者参数进行直接请求服务器，如：

`http://www.example.com/?nocache=true`。Nginx 依然可以为将那些没有避开缓存的请求缓存响应结果。

### Nginx 使用哪些缓存键？

Nginx 生成的键的默认格式是类似于下面的 [Nginx 变量](http://nginx.org/en/docs/varindex.html?_ga=1.49172198.1568941527.1438257987)的 MD5 哈希值: `$scheme$proxy_host$request_uri`，实际的算法有些复杂。

```nginx
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m
                 use_temp_path=off;
server {
    ...
    location / {
        proxy_cache $my_cache;
        proxy_pass http://my_upstream;
    }
}
```

按照上面的配置，`http://www.example.org/my_image.jpg` 的缓存键被计算为 `md5(“http://my_upstream:80/my_image.jpg”)`。

注意，[$proxy_host](http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=1.245124677.1568941527.1438257987#var_proxy_host)变量用于哈希之后的值而不是实际的主机名（www.example.com）。`$proxy_host`被定义为[proxy_pass](http://nginx.org/en/docs/http/ngx_http_proxy_module.html?&_ga=1.85290457.1568941527.1438257987#proxy_pass)中指定的代理服务器的主机名和端口号。

为了改变变量（或其他项）作为基础键，可以使用 [proxy_cache_key](http://nginx.org/r/proxy_cache_key?_ga=1.47539815.1568941527.1438257987)命令（下面的问题会讲到）。

### 可以使用 Cookie 作为缓存键的一部分吗？

可以，缓存键可以配置为任意值，如：

```nginx
proxy_cache_key $proxy_host$request_uri$cookie_jessionid;
```

### Nginx 使用 Etag 头部吗？

在 Nginx 1.7.3 和 [Nginx Plus R5](https://www.nginx.com/blog/nginx-plus-r5-released/) 及之后的版本，配合使用 `If-None-Match`， Etag 是完全支持的。

### Nginx 如何处理字节范围请求？

如果缓存中的文件是最新的，Nginx 会对客户端提出的字节范围请求传递指定的字节。如果文件并没有被提前缓存，或者是陈旧的，那么 Nginx 会从服务器上下载完整文件。如果请求了单字节范围，Nginx 会尽快的将该字节发送给客户端，如果在下载的数据流中刚好有这个字节。如果请求指定了同一个文件中的多个字节范围，Nginx 则会在文件下载完毕时将整个文件发送给客户端。

一旦文件下载完毕，Nginx 将整个数据移动到缓存中，这样一来，无论将来的字节范围请求是单字节还是多字节范围，Nginx 都可以在缓存中找到指定的内容立即响应。

### Nginx 支持缓存清洗吗？

[Nginx Plus](https://www.nginx.com/products/content-caching-nginx-plus/)支持有选择性的清洗缓存。当原始服务器上文件已经被更新，但是Nginx Plus缓存中文件依然有效（`Cache-Control:max-age`依然有效，`proxy_cache_path`命令中`inactive`参数设置的超时时间没有过期），这个功能便十分有用。使用Nginx Plus的缓存清洗特性，这个文件可以被轻易的删除。详细信息，参见[Purging Content from the Cache](https://www.nginx.com/products/content-caching-nginx-plus/#purging)。

### Nginx 如何处理 Pragma 头部？

当客户端添加了 `Pragma:no-cache` 头部，则**请求会绕过缓存直接访问服务器请求内容**。Nginx 默认不考虑 Pragma 头部，不过你可以使用下面的`proxy_cache_bypass` 的命令来配置该特性：

```nginx
location /images/ {
    proxy_cache my_cache;
    proxy_cache_bypass $http_pragma;
    ...
}
```

### Nginx 支持 Vary 头部吗？

是的，在[Nginx Plus R5](https://www.nginx.com/blog/nginx-plus-r5-released/)、Nginx1.7.7 和之后的版本中是支持的。看看这篇不错的文章： [good overview of the Vary header](https://www.fastly.com/blog/best-practices-for-using-the-vary-header/)。
