<audio id="audio" title="44 | OpenResty 的杀手锏：动态" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/b1/44/b175c2d393a0cdabfec9081ce1afbc44.mp3"></audio>

你好，我是温铭。

到目前为止，和 OpenResty 性能相关的内容，我就差不多快要介绍完了。我相信，掌握并灵活运用好这些优化技巧，一定可以让你的代码性能提升一个数量级。今天，在性能优化的最后一个部分，我来讲一讲 OpenResty 中被普遍低估的一种能力：动态。

让我们先来看下什么是动态，以及它和性能之间有什么样的关系。

这里的动态，指的是程序可以在运行时、在不重新加载的情况下，去修改参数、配置，乃至修改自身的代码。具体到 Nginx 和 OpenResty 领域，你去修改上游、SSL 证书、限流限速阈值，而不用重启服务，就属于实现了动态。至于动态和性能之间的关系，很显然，如果这几类操作不能动态地完成，那么频繁的 reload Nginx 服务，自然就会带来性能的损耗。

不过，我们知道，开源版本的 Nginx 并不支持动态特性，所以，你要对上游、SSL 证书做变更，就必须通过修改配置文件、重启服务的方式才能生效。而商业版本的 Nginx Plus 提供了部分动态的能力，你可以用 REST API 来完成更新，但这最多算是一个不够彻底的改良。

但是，在 OpenResty 中，这些桎梏都是不存在的，动态可以说就是 OpenResty 的杀手锏。你可能纳闷儿，为什么基于 Nginx 的 OpenResty 却可以支持动态呢？原因也很简单，Nginx 的逻辑是通过 C 模块来完成的，而 OpenResty 是通过脚本语言 Lua 来完成的——脚本语言的一大优势，便是运行时可以去做动态地改变。

## 动态加载代码

下面我们就来看看，如何在 OpenResty 中动态地加载 Lua 代码：

```
resty -e 'local s = [[ngx.say(&quot;hello world&quot;)]]
local func, err = loadstring(s)
func()'

```

你没有看错，只要短短的两三行代码，就可以把一个字符串变为一个 Lua 函数，并运行起来。我们进一步仔细看下这几行代码，我来简单解读一下：

- 首先，我们声明了一个字符串，它的内容是一段合法的 Lua 代码，把 `hello world` 打印出来；
- 然后，使用 Lua 中的 `loadstring` 函数，把字符串对象转为函数对象`func`；
- 最后，在函数名的后面加上括号，把 `func` 执行起来，打印出 `hello world` 来。

当然，在这段代码的基础之上，我们还可以扩展出更多好玩和实用的功能。接下来，我就带你一起来“尝尝鲜”。

## 功能一：FaaS

首先是函数即服务，这是近年来很热门的技术方向，我们看下在 OpenResty 中如何实现。在刚刚的代码中，字符串是一段 Lua 代码，我们还可以把它改成一个 Lua 函数：

```
local s = [[
 return function()
     ngx.say(&quot;hello world&quot;)
end
]]

```

我们讲过，函数在 Lua 中是一等公民，这段代码便是返回了一个匿名函数。在执行这个匿名函数时，我们使用 `pcall` 做了一层保护。`pcall` 会在保护模式下运行函数，并捕获其中的异常，如果正常就返回 true 和执行的结果，如果失败就返回 false 和错误信息，也就是下面这段代码：

```
local func1, err = loadstring(s)
local ret, func = pcall(func1)

```

自然，把上面的两部分结合起来，就会得到完整的、可运行的示例：

```
resty -e 'local s = [[
 return function()
    ngx.say(&quot;hello world&quot;)
end
]]
local  func1 = loadstring(s)
local ret, func = pcall(func1)
func()'

```

更深入一步，我们还可以把 `s` 这个包含函数的字符串，改成可以由用户指定的形式，并加上执行它的条件，这样其实就是 FaaS 的原型了。这里，我提供了一个完整的[实现](https://github.com/apache/incubator-apisix/blob/master/apisix/plugins/serverless.lua)，如果你对FaaS感兴趣，想要继续研究，推荐你通过这个链接深入学习。

## 功能二：边缘计算

OpenResty 的动态不仅可以用于 FaaS，让脚本语言的动态细化到函数级别，还可以在边缘计算上发挥动态的优势。

得益于 Nginx 和 LuaJIT 良好的多平台支持特性，OpenResty 不仅能运行在 X86 架构下，对于 ARM 的支持也很完善。同时， OpenResty 支持七层和四层的代理，这样一来，常见的各种协议都可以被 OpenResty 解析和代理，这其中也包括了 IoT 中的几种协议。

因为这些优势，我们便可以把 OpenResty 的触角，从 API 网关、WAF、web 服务器等服务端的领域，伸展到物联网设备、CDN 边缘节点、路由器等最靠近用户的边缘节点上去。

这并非只是一种畅想，事实上，OpenResty 已经在上述领域中被大量使用了。以 CDN 的边缘节点为例，OpenResty 的最大使用者 CloudFlare 很早就借助 OpenResty 的动态特性，实现了对于 CDN 边缘节点的动态控制。

CloudFlare 的做法和上面动态加载代码的原理是类似的，大概可以分为下面几个步骤：

- 首先，从键值数据库集群中获取到有变化的代码文件，获取的方式可以是后台 timer 轮询，也可以是用“发布-订阅”的模式来监听；
- 然后，用更新的代码文件替换本地磁盘的旧文件，然后使用 `loadstring` 和 `pcall`的方式，来更新内存中加载的缓存；

这样，下一个被处理的终端请求，就会走更新后的代码逻辑。

当然，实际的应用要比上面的步骤考虑更多的细节，比如版本的控制和回退、异常的处理、网络的中断、边缘节点的重启等，但整体的流程是不变的。

如果把 CloudFlare 的这套做法，从 CDN 边缘节点挪移到其他边缘的场景下，那我们就可以把很多计算能力动态地赋予边缘节点的设备。这不仅可以充分利用边缘节点的计算能力，也可以让用户请求得到更快速的响应。因为边缘节点会把原始数据处理过后，再汇总给远端的服务器，这就大大减少了数据的传输量。

不过，要把 FaaS 和边缘计算做好，OpenResty 的动态只是一个良好的基础，你还需要考虑自身周边生态的完善和厂商的加入，这就不仅仅是技术的范畴了。

## 动态上游

现在，让我们把思绪拉回到 OpenResty 上来，一起来看如何实现动态上游。`lua-resty-core` 提供了 `ngx.balancer` 这个库来设置上游，它需要放到 OpenResty 的 `balancer` 阶段来运行：

```
balancer_by_lua_block {
    local balancer = require &quot;ngx.balancer&quot;
    local host = &quot;127.0.0.2&quot;
    local port = 8080

    local ok, err = balancer.set_current_peer(host, port)
    if not ok then
        ngx.log(ngx.ERR, &quot;failed to set the current peer: &quot;, err)
        return ngx.exit(500)
    end
}

```

我来简单解释一下。`set_current_peer` 函数，就是用来设置上游的 IP 地址和端口的。不过要注意，这里并不支持域名，你需要使用 `lua-resty-dns` 库来为域名和 IP 做一层解析。

不过，`ngx.balancer` 还比较底层，虽然它有设置上游的能力，但动态上游的实现远非如此简单。所以，在 `ngx.balancer` 前面还需要两个功能：

- 一是上游的选择算法，究竟是一致性哈希，还是 roundrobin；
- 二是上游的健康检查机制，这个机制需要剔除掉不健康的上游，并且需要在不健康的上游变健康的时候，重新把它加入进来。

而OpenResty 官方的 `lua-resty-balancer` [这个库](https://github.com/openresty/lua-resty-balancer)中，则包含了 `resty.chash` 和 `resty.roundrobin` 两类算法来完成第一个功能，并且有 `lua-resty-upstream-healthcheck` 来尝试完成第二个功能。

不过，这其中还是有两个问题。

第一点，缺少最后一公里的完整实现。把 `ngx.balancer`、`lua-resty-balancer` 和 `lua-resty-upstream-healthcheck` 整合并实现动态上游的功能，还是需要一些工作量的，这就拦住了大部分的开发者。

第二点，`lua-resty-upstream-healthcheck` 的实现并不完整，只有被动的健康检查，而没有主动的健康检查。

简单解释一下，这里的被动健康检查，是指由终端的请求触发，进而分析上游的返回值来作为健康与否的判断条件。如果没有终端请求，那么上游是否健康就无从得知了。而主动健康检查就可以弥补这个缺陷，它使用 `ngx.timer` 定时去轮询指定的上游接口，来检测健康状态。

所以，在实际的实践中，我们通常推荐使用 `lua-resty-healthcheck` 这个[库](https://github.com/Kong/lua-resty-healthcheck)，来完成上游的健康检查。它的优点是包含了主动和被动的健康检查，而且在多个项目中都经过了验证，可靠性更高。

更进一步，新兴的微服务 API 网关APISIX，在 `lua-resty-healthcheck` 的基础之上，对动态上游做了完整的实现。我们可以参考它的[实现](https://github.com/iresty/apisix/blob/master/lua/apisix/http/balancer.lua)，总共只有 200 多行代码，你可以很轻松地把它剥离出来，放到你的自己的项目中使用。

## 写在最后

讲了这么多，最后，给你留一个思考题。关于OpenResty 的动态，你觉得还可以在哪些领域和场景来发挥它的这种优势呢？提醒一下，这个章节中介绍的每部分的内容，你都可以展开来做更详细和深入的分析。

欢迎留言和我讨论，也欢迎你把这篇文章分享出去，和更多的人一起学习、进步。
