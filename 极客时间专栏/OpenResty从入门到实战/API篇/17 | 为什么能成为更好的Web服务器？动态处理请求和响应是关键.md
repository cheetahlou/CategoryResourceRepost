<audio id="audio" title="17 | 为什么能成为更好的Web服务器？动态处理请求和响应是关键" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/48/6e/4896687fb4708633494c82751b8f1b6e.mp3"></audio>

你好，我是温铭。经过前面内容的铺垫后， 相信你已经对 OpenResty 的概念和如何学习它有了基本的认识。今天这节课，我们来看一下 OpenResty 如何处理终端请求和响应。

虽然 OpenResty 是基于 NGINX 的 Web 服务器，但它与 NGINX 却有本质的不同：NGINX 由静态的配置文件驱动，而 OpenResty 是由 Lua API 驱动的，所以能提供更多的灵活性和可编程性。

下面，就让我来带你领略 Lua API 带来的好处吧。

## API 分类

首先我们要知道，OpenResty 的 API 主要分为下面几个大类：

- 处理请求和响应；
- SSL 相关；
- shared dict；
- cosocket；
- 处理四层流量；
- process 和 worker；
- 获取 NGINX 变量和配置；
- 字符串、时间、编解码等通用功能。

这里，我建议你同时打开 OpenResty 的 Lua API 文档，对照着其中的 [API 列表](https://github.com/openresty/lua-nginx-module/#nginx-api-for-lua)  ，看看是否能和这个分类联系起来。

OpenResty 的 API 不仅仅存在于 lua-nginx-module 项目中，也存在于 lua-resty-core 项目中，比如 ngx.ssl、ngx.base64、ngx.errlog、ngx.process、ngx.re.split、ngx.resp.add_header、ngx.balancer、ngx.semaphore、ngx.ocsp 这些 API 。

而对于不在 lua-nginx-module 项目中的 API，你需要单独 require 才能使用。举个例子，比如你想使用 split 这个字符串分割函数，就需要按照下面的方法来调用：

```
$ resty -e 'local ngx_re = require &quot;ngx.re&quot;
 local res, err = ngx_re.split(&quot;a,b,c,d&quot;, &quot;,&quot;, nil, {pos = 5})
 print(res)
 '

```

当然，这可能会给你带来一个困惑：在 lua-nginx-module 项目中，明明有 ngx.re.sub、ngx.re.find 等好几个 ngx.re 开头的 API，为什么单单是 ngx.re.split 这个 API ，需要 require 后才能使用呢？

事实上，在前面 lua-resty-core 章节中，我们也提到过，OpenResty 新的 API 都是通过 FFI 的方式在 `lua-rety-core` 仓库中实现的，所以难免就会存在这种割裂感。自然，我也很期待lua-nginx-module 和 lua-resty-core 这两个项目以后可以合并，彻底解决此类问题。

## 请求

接下来，我们具体了解下OpenResty 是如何处理终端请求和响应的。先来看下处理请求的 API，不过，以 ngx.req 开头的 API 有 20 多个，该怎么下手呢？

我们知道，HTTP 请求报文由三部分组成：请求行、请求头和请求体，所以下面我就按照这三部分来对 API 做介绍。

### 请求行

首先是请求行，HTTP 的请求行中包含请求方法、URI 和 HTTP 协议版本。在 NGINX 中，你可以通过内置变量的方式，来获取其中的值；而在 OpenResty 中对应的则是 `ngx.var.*` 这个 API。我们来看两个例子。

- `$scheme` 这个内置变量，在 NGINX 中代表协议的名字，是 “http” 或者 “https”；而在 OpenResty 中，你可以通过 `ngx.var.scheme` 来返回同样的值。
- `$request_method` 代表的是请求的方法，“GET”、“POST” 等；而在 OpenResty 中，你可以通过 `ngx.var. request_method` 来返回同样的值。

至于完整的 NGINX 内置变量列表，你可以访问 NGINX 的官方文档来获取：[http://nginx.org/en/docs/http/ngx_http_core_module.html#variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)。

那么问题就来了：既然可以通过`ngx.var.*` 这种返回变量值的方法，来得到请求行中的数据，为什么 OpenResty 还要单独提供针对请求行的 API 呢？

这其实是很多方面因素的综合考虑结果：

- 首先是对性能的考虑。`ngx.var` 的效率不高，不建议反复读取；
- 也有对程序友好的考虑，`ngx.var` 返回的是字符串，而非 Lua 对象，遇到获取 args 这种可能返回多个值的情况，就不好处理了；
- 另外是对灵活性的考虑，绝大部分的 `ngx.var` 是只读的，只有很少数的变量是可写的，比如 `$args` 和 `limit_rate`，可很多时候，我们会有修改 method、URI 和 args 的需求。

所以， OpenResty 提供了多个专门操作请求行的 API，它们可以对请求行进行改写，以便后续的重定向等操作。

我们先来看下，如何通过 API 来获取 HTTP 协议版本号。OpenResty 的 API `ngx.req.http_version` 和 NGINX 的 `$server_protocol` 变量的作用一样，都是返回 HTTP 协议的版本号。不过这个 API 的返回值是数字格式，而非字符串，可能的值是 2.0、1.0、1.1 和 0.9，如果结果不在这几个值的范围内，就会返回 nil。

再来看下获取请求行中的请求方法。刚才我们提到过，`ngx.req.get_method` 和 NGINX 的 `$request_method` 变量的作用、返回值一样，都是字符串格式的方法名。

但是，改写当前 HTTP 请求方法的 API，也就是 `ngx.req.set_method`，它接受的参数格式却并非字符串，而是内置的数字常量。比如，下面的代码，把请求方法改写为 POST：

```
ngx.req.set_method(ngx.HTTP_POST)

```

为了验证 `ngx.HTTP_POST` 这个内置常量，确实是数字而非字符串，你可以打印出它的值，看输出是否为 8：

```
$ resty -e 'print(ngx.HTTP_POST)'

```

这样一来，get 方法的返回值为字符串，而set 方法的输入值却是数字，就很容易让你在写代码的时候想当然了。如果是 set 时候传值混淆的情况还好，API 会崩溃报出 500 的错误；但如果是下面这种判断逻辑的代码：

```
if (ngx.req.get_method() == ngx.HTTP_POST) then
    -- do something
 end

```

这种代码是可以正常运行的，不会报出任何错误，甚至在 code review 时也很难发现。不幸的是，我就犯过类似的错误，对此记忆犹新：当时已经经过了两轮 code review，还有不完整的测试案例尝试覆盖，然而，最终还是因为线上环境异常才追踪到了这里。

碰到这类情况，除了自己多小心，或者再多一层封装外，并没有什么有效的方法来解决。平常你在设计自己的业务 API 时，也可以多做一些这方面的考虑，尽量保持 get、set 方法的参数格式一致，即使这会牺牲一些性能。

另外，在改写请求行的方法中，还有 `ngx.req.set_uri` 和 `ngx.req.set_uri_args` 这两个 API，可以用来改写 uri 和 args。我们来看下这个 NGINX 配置：

```
 rewrite ^ /foo?a=3? break;

```

那么，如何用等价的 Lua API 来解决呢？答案就是下面这两行代码。

```
ngx.req.set_uri_args(&quot;a=3&quot;)
ngx.req.set_uri(&quot;/foo&quot;)

```

其实，如果你看过官方文档，就会发现 `ngx.req.set_uri` 还有第二个参数：jump，默认是 false。如果设置为 true，就等同于把 rewrite 指令的 flag 设置为 `last`，而非上面示例中的 `break`。

不过，我个人并不喜欢 rewrite 指令的 flag 配置，看不懂也记不住，远没有代码来的直观和好维护。

### 请求头

再来看下和请求头有关的 API。我们知道，HTTP 的请求头是 `key : value` 格式的，比如：

```
Accept: text/css,*/*;q=0.1
Accept-Encoding: gzip, deflate, br

```

在OpenResty 中，你可以使用 `ngx.req.get_headers` 来解析和获取请求头，返回值的类型则是 table：

```
local h, err = ngx.req.get_headers()
 
  if err == &quot;truncated&quot; then
      -- one can choose to ignore or reject the current request here
  end
 
  for k, v in pairs(h) do
      ...
  end

```

这里默认返回前 100 个 header，如果请求头超过了 100 个，就会返回 `truncated` 的错误信息，由开发者自己决定如何处理。你可能会好奇为什么会有这样的处理，这一点先留个悬念，在后面安全漏洞的章节中我会提到。

不过，需要注意的是，OpenResty 并没有提供获取某一个指定请求头的 API，也就是没有 `ngx.req.header['host']` 这种形式。如果你有这样的需求，那就需要借助 NGINX 的变量 `$http_xxx` 来实现了，那么在 OpenResty 中，就是 `ngx.var.http_xxx` 这样的获取方式。

看完了获取请求头，我们再来看看应该如何改写和删除请求头，这两种操作的 API 其实都很直观：

```
ngx.req.set_header(&quot;Content-Type&quot;, &quot;text/css&quot;)
ngx.req.clear_header(&quot;Content-Type&quot;)

```

当然，官方文档中也提到了其他方法来删除请求头，比如把 header 的值设置为 nil等，但为了代码更加清晰的考虑，我还是推荐统一用 `clear_header` 来操作。

### 请求体

最后来看请求体。出于性能考虑，OpenResty 不会主动读取请求体的内容，除非你在 nginx.conf 中强制开启了 `lua_need_request_body` 指令。此外，对于比较大的请求体，OpenResty 会把内容保存在磁盘的临时文件中，所以读取请求体的完整流程是下面这样的：

```
ngx.req.read_body()
local data = ngx.req.get_body_data()
if not data then
    local tmp_file = ngx.req.get_body_file()
     -- io.open(tmp_file)
     -- ...
 end

```

这段代码中有读取磁盘文件的 IO 阻塞操作。你应该根据实际情况来调整 `client_body_buffer_size` 配置的大小（64 位系统下默认是 16 KB），尽量减少阻塞的操作；你也可以把 `client_body_buffer_size` 和 `client_max_body_size` 配置成一样的，完全在内存中来处理，当然，这取决于你内存的大小和处理的并发请求数。

另外，请求体也可以被改写，`ngx.req.set_body_data` 和 `ngx.req.set_body_file` 这两个API，分别接受字符串和本地磁盘文件做为输入参数，来完成请求体的改写。不过，这类操作并不常见，你可以查看文档来获取更详细的内容。

## 响应

处理完请求后，我们就需要发送响应返回给客户端了。和请求报文一样，响应报文也由几个部分组成，即状态行、响应头和响应体。同样的，接下来我会按照这三部分来介绍相应的API。

### 状态行

状态行中，我们主要关注的是状态码。在默认情况下，返回的 HTTP 状态码是 200，也就是 OpenResty 中内置的常量 `ngx.HTTP_OK`。但在代码的世界中，处理异常情况的代码总是占比最多的。

如果你检测了请求报文，发现这是一个恶意的请求，那么你需要终止请求:

```
 ngx.exit(ngx.HTTP_BAD_REQUEST)

```

不过，OpenResty 的 HTTP 状态码中，有一个特别的常量：`ngx.OK`。当 `ngx.exit(ngx.OK)` 时，请求会退出当前处理阶段，进入下一个阶段，而不是直接返回给客户端。

当然，你也可以选择不退出，只使用 `ngx.status` 来改写状态码，比如下面这样的写法：

```
ngx.status = ngx.HTTP_FORBIDDEN

```

如果你想了解更多的状态码常量，可以从[文档](https://github.com/openresty/lua-nginx-module/#http-status-constants)中查询到。

### 响应头

说到响应头，其实，你有两种方法来设置它。第一种是最简单的：

```
ngx.header.content_type = 'text/plain'
ngx.header[&quot;X-My-Header&quot;] = 'blah blah'
ngx.header[&quot;X-My-Header&quot;] = nil -- 删除

```

这里的 ngx.header 保存了响应头的信息，可以读取、修改和删除。

第二种设置响应头的方法是 `ngx_resp.add_header` ，来自 lua-resty-core 仓库，它可以增加一个头信息，用下面的方法来调用：

```
local ngx_resp = require &quot;ngx.resp&quot;
ngx_resp.add_header(&quot;Foo&quot;, &quot;bar&quot;)

```

与第一种方法的不同之处在于，add header 不会覆盖已经存在的同名字段。

### 响应体

最后看下响应体，在 OpenResty 中，你可以使用 `ngx.say` 和 `ngx.print` 来输出响应体：

```
ngx.say('hello, world')

```

这两个 API 的功能是一致的，唯一的不同在于， `ngx.say` 会在最后多一个换行符。

为了避免字符串拼接的低效，`ngx.say / ngx.print` 不仅支持字符串作为参数，也支持数组格式：

```
$ resty -e 'ngx.say({&quot;hello&quot;, &quot;, &quot;, &quot;world&quot;})'
 hello, world

```

这样在 Lua 层面就跳过了字符串的拼接，把这个它不擅长的事情丢给了 C 函数去处理。

## 写在最后

到此，让我们回顾下今天的内容。我们按照请求报文和响应报文的内容，依次介绍了与之相关的 OpenResty API。你可以看得出来，和 NGINX 的指令相比，OpenResty API更加灵活和强大。

那么，在你处理 HTTP 请求时，OpenResty 提供的 Lua API 是否足够满足你的需求呢？欢迎留言一起探讨，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。


