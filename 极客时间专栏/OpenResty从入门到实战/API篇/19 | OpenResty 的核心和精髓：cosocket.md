<audio id="audio" title="19 | OpenResty 的核心和精髓：cosocket" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/b3/39/b33be3f616df6012d81f7605d30a5e39.mp3"></audio>

你好，我是温铭，今天我们来学习下 OpenResty 中的核心技术：cosocket。

其实在前面的课程中，我们就已经多次提到过它了，cosocket 是各种 `lua-resty-*` 非阻塞库的基础，没有 cosocket，开发者就无法用 Lua 来快速连接各种外部的网络服务。

在早期的 OpenResty 版本中，如果你想要去与 Redis、memcached 这些服务交互的话，需要使用 `redis2-nginx-module`、`redis-nginx-module` 和 `memc-nginx-module`这些 C 模块.这些模块至今仍然在 OpenResty 的发行包中。

不过，cosocket 功能加入以后，它们都已经被 `lua-resty-redis` 和 `lua-resty-memcached` 替代，基本上没人再去使用 C 模块连接外部服务了。

## 什么是 cosocket？

那究竟什么是cosocket 呢？事实上，cosocket是 OpenResty 中的专有名词，是把协程和网络套接字的英文拼在一起形成的，即 cosocket = coroutine + socket。所以，你可以把 cosocket 翻译为“协程套接字”。

cosocket 不仅需要 Lua 协程特性的支持，也需要 Nginx 中非常重要的事件机制的支持，这两者结合在一起，最终实现了非阻塞网络 I/O。另外，cosocket 支持 TCP、UDP 和 Unix Domain Socket。

如果我们在 OpenResty 中调用一个 cosocket 相关函数，内部实现便是下面这张图的样子：

<img src="https://static001.geekbang.org/resource/image/80/06/80d16e11d2750d6e4127445c126c9f06.png" alt="">

记性比较好的同学应该发现了，在前面 OpenResty 原理和基本概念的那节课里，我也用过这张图。从图中你可以看到，用户的 Lua 脚本每触发一个网络操作，都会有协程的 yield 以及 resume。

遇到网络 I/O 时，它会交出控制权（yield），把网络事件注册到 Nginx 监听列表中，并把权限交给 Nginx；当有 Nginx 事件达到触发条件时，便唤醒对应的协程继续处理（resume）。

OpenResty 正是以此为蓝图，封装实现 connect、send、receive 等操作，形成了我们如今见到的 cosocket API。下面，我就以处理 TCP 的 API 为例来介绍一下。处理 UDP 和 Unix Domain Socket ，与TCP 的接口基本是一样的。

## cosocket API 和指令简介

TCP 相关的 cosocket API 可以分为下面这几类。

- 创建对象：ngx.socket.tcp。
- 设置超时：tcpsock:settimeout 和 tcpsock:settimeouts。
- 建立连接：tcpsock:connect。
- 发送数据：tcpsock:send。
- 接受数据：tcpsock:receive、tcpsock:receiveany 和 tcpsock:receiveuntil。
- 连接池：tcpsock:setkeepalive。
- 关闭连接：tcpsock:close。

我们还要特别注意下，这些 API 可以使用的上下文：

```
rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*_

```

这里我还要强调一点，归咎于 Nginx 内核的各种限制，cosocket API 在 `set_by_lua*`， `log_by_lua*`， `header_filter_by_lua*` 和 `body_filter_by_lua*` 中是无法使用的。而在 `init_by_lua*` 和 `init_worker_by_lua*` 中暂时也不能用，不过 Nginx 内核对这两个阶段并没有限制，后面可以增加对这它们的支持。

此外，与这些 API 相关的，还有 8 个 `lua_socket_` 开头的 Nginx 指令，我们简单来看一下。

- `lua_socket_connect_timeout`：连接超时，默认 60 秒。
- `lua_socket_send_timeout`：发送超时，默认 60 秒。
- `lua_socket_send_lowat`：发送阈值（low water），默认为 0。
- `lua_socket_read_timeout`： 读取超时，默认 60 秒。
- `lua_socket_buffer_size`：读取数据的缓存区大小，默认 4k/8k。
- `lua_socket_pool_size`：连接池大小，默认 30。
- `lua_socket_keepalive_timeout`：连接池 cosocket 对象的空闲时间，默认 60 秒。
- `lua_socket_log_errors`：cosocket 发生错误时，是否记录日志，默认为 on。

这里你也可以看到，有些指令和 API 的功能一样的，比如设置超时时间和连接池大小等。不过，如果两者有冲突的话，API 的优先级高于指令，会覆盖指令设置的值。所以，一般来说，我们都推荐使用 API 来做设置，这样也会更加灵活。

接下来，我们一起来看一个具体的例子，弄明白到底如何使用这些 cosocket API。下面这段代码的功能很简单，是发送 TCP 请求到一个网站，并把返回的内容打印出来：

```
$ resty -e 'local sock = ngx.socket.tcp()
        sock:settimeout(1000)  -- one second timeout
        local ok, err = sock:connect(&quot;www.baidu.com&quot;, 80)
        if not ok then
            ngx.say(&quot;failed to connect: &quot;, err)
            return
        end

        local req_data = &quot;GET / HTTP/1.1\r\nHost: www.baidu.com\r\n\r\n&quot;
        local bytes, err = sock:send(req_data)
        if err then
            ngx.say(&quot;failed to send: &quot;, err)
            return
        end

        local data, err, partial = sock:receive()
        if err then
            ngx.say(&quot;failed to receive: &quot;, err)
            return
        end

        sock:close()
        ngx.say(&quot;response is: &quot;, data)'

```

我们来具体分析下这段代码。

- 首先，通过 `ngx.socket.tcp()` ，创建  TCP 的 cosocket 对象，名字是 sock。
- 然后，使用 `settimeout()` ，把超时时间设置为 1 秒。注意这里的超时没有区分 connect、receive，是统一的设置。
- 接着，使用 `connect()` 去连接指定网站的 80 端口，如果失败就直接退出。
- 连接成功的话，就使用 `send()` 来发送构造好的数据，如果发送失败就退出。
- 发送数据成功的话，就使用 `receive()` 来接收网站返回的数据。这里 `receive()` 的默认参数值是 `*l`，也就是只返回第一行的数据；如果参数设置为了`*a`，就是持续接收数据，直到连接关闭；
- 最后，调用 `close()` ，主动关闭 socket 连接。

你看，短短几步就可以完成，使用 cosocket API 来做网络通信，就是这么简单。不过，不能满足于此，接下来，我们对这个示例再做一些调整。

**第一个动作，对 socket 连接、发送和读取这三个动作，分别设置超时时间。**

我们刚刚用的`settimeout()` ，作用是把超时时间统一设置为一个值。如果要想分开设置，就需要使用 `settimeouts()` 函数，比如下面这样的写法：

```
sock:settimeouts(1000, 2000, 3000) 

```

这行代码表示连接超时为 1 秒，发送超时为 2 秒，读取超时为 3 秒。

在OpenResty 和 lua-resty 库中，大部分和时间相关的 API 的参数，都以毫秒为单位，但也有例外，需要你在调用的时候特别注意下。

**第二个动作，receive接收指定大小的内容。**

刚刚说了，`receive()` 接口可以接收一行数据，也可以持续接收数据。不过，如果你只想接收 10K 大小的数据，应该怎么设置呢？

这时，`receiveany()` 闪亮登场。它就是专为满足这种需求而设计的，一起来看下面这行代码：

```
local data, err, partial = sock:receiveany(10240)

```

这段代码就表示，最多只接收 10K 的数据。

当然，关于receive，还有另一个很常见的用户需求，那就是一直获取数据，直到遇到指定字符串才停止。

`receiveuntil()` 专门用来解决这类问题，它不会像 `receive()` 和 `receiveany()` 一样返回字符串，而会返回一个迭代器。这样，你就可以在循环中调用它来分段读取匹配到的数据，当读取完毕时，就会返回 nil。下面就是一个例子：

```
 local reader = sock:receiveuntil(&quot;\r\n&quot;)

 while true do
     local data, err, partial = reader(4)
     if not data then
         if err then
             ngx.say(&quot;failed to read the data stream: &quot;, err)
             break
         end

         ngx.say(&quot;read done&quot;)
         break
     end
     ngx.say(&quot;read chunk: [&quot;, data, &quot;]&quot;)
 end

```

这段代码中的 `receiveuntil` 会返回 `\r\n` 之前的数据，并通过迭代器每次读取其中的 4 个字节，也就实现了我们想要的功能。

**第三个动作，不直接关闭 socket，而是放入连接池中。**

我们知道，没有连接池的话，每次请求进来都要新建一个连接，就会导致 cosocket 对象被频繁地创建和销毁，造成不必要的性能损耗。

为了避免这个问题，在你使用完一个 cosocket 后，可以调用 `setkeepalive()` 放到连接池中，比如下面这样的写法：

```
local ok, err = sock:setkeepalive(2 * 1000, 100)
if not ok then
    ngx.say(&quot;failed to set reusable: &quot;, err)
end

```

这段代码设置了连接的空闲时间为 2 秒，连接池的大小为 100。这样，在调用 `connect()` 函数时，就会优先从连接池中获取 cosocket 对象。

不过，关于连接池的使用，有两点需要我们注意一下。

- 第一，不能把发生错误的连接放入连接池，否则下次使用时，就会导致收发数据失败。这也是为什么我们需要判断每一个 API 调用是否成功的一个原因。
- 第二，要搞清楚连接的数量。连接池是 worker 级别的，每个 worker 都有自己的连接池。所以，如果你有 10 个 worker，连接池大小设置为 30，那么对于后端的服务来讲，就等于有 300 个连接。

## 写在最后

总结一下，今天我们学习了cosocket 的基本概念，以及相关的指令和 API，并通过一个实际的例子，熟悉了TCP 相关的 API 应该如何使用。而UDP 和 Unix Domain Socket的使用类似于TCP，弄明白今天所学，你基本上都能迎刃而解了。

从中你应该也能感受到，cosocket 用起来还是比较容易上手的，而且用好它，你就可以去连接各种外部的服务了，可以说是给 OpenResty 插上了想象的翅膀。

最后，给你留两个作业题。

第一问，在今天的例子中，`tcpsock:send` 发送的是字符串，如果我们需要发送一个由字符串构成的 table，又该怎么处理呢？

第二问，你也看到了，cosocket 在很多阶段中不能使用，那么，你能否想到一些绕过的方式呢？

欢迎留言和我分享，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。


