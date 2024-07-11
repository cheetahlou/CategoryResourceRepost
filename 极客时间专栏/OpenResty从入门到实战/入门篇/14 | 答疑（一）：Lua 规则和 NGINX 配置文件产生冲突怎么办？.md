<audio id="audio" title="14 | 答疑（一）：Lua 规则和 NGINX 配置文件产生冲突怎么办？" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/9c/e5/9c1350a183922cf8c6cf71127c15e1e5.mp3"></audio>

你好，我是温铭。

专栏更新到现在，OpenResty第一版块入门篇，我们就已经学完了。恭喜你没有掉队，仍然在积极学习和实践操作，并且热情地留下了你的思考。

很多留言提出的问题很有价值，大部分我都已经在app里回复过，一些手机上不方便回复的或者比较典型、有趣的问题，我专门摘了出来，作为今天的答疑内容，集中回复。另一方面，也是为了保证所有人都能不漏掉任何一个重点。

下面我们来看今天的这5个问题。

## 第一问，OpenResty的名字和语言

Q：看到现在，我还没看懂 OpenResty 这个名字的来历。另外，OpenResty 借助 Lua 语言，插上翅膀，那么为什么不借助其他脚本语言呢？比如 Shell 等。

A：事实上，OpenResty 最早是雅虎中国的一个公司项目，起步于 2007 年 10 月。当时兴起了 OpenAPI 的热潮，于是春哥想做一个类似的东西，可以支持各种 Web Service 的需求。Open 这个名字取自 OpenAPI， Resty 则是取自 rest API。最初 OpenResty 的目的，并非是做 web 服务器和开发平台，而是做类似网站这样的应用。

OpenResty 在十几年前开源的时候，支持同步非阻塞的语言凤毛麟角。即使是到了现在，后端语言可以达到 OpenResty 这种性能级别的也不多。当前，更多的开发者把 OpenResty 用在 API 网关和软 WAF 领域，这也算是开发者的自然选择了。

至于语言方面，OpenResty 并不是唯一一个把其他开发语言嵌入NGINX 的项目。比如，NGINX 官方就把 JS 嵌入了进来；同时也有一些开源项目，把 PHP 嵌入 NGINX。

通常来说，选择借助哪一门语言，会综合考虑协程、JIT和语言普及度等多种因素。对于OpenResty，在 2007 年时，Lua 确实是最佳的选择。实际上，OpenResty 在最早的版本中选择了 perl 而不是 Lua，也可以说是走了一段弯路。

## 第二问，配置文件的规则优先级

Q：当 OpenResty 中的 Lua 规则和 NGINX 配置文件产生冲突时，比如NGINX配置了rewrite规则，又同时引用了rewrite_by_lua_file，那么这两条规则的优先级是什么？

A：其实，这个具体要看 NGINX 配置的 rewrite 规则是怎么写的了，是 break 还是 last。这一点，在 OpenResty 的官方文档中有注明，并且配了一个示例代码：

```
 location /foo {
     rewrite ^ /bar;
     rewrite_by_lua 'ngx.exit(503)';
 }
 location /bar {
     ...
 }

```

在示例代码的这个配置中，ngx.exit(503) 是不会被执行的。

但是，如果你改成下面这样的写法，ngx.exit(503) 就可以被执行。

```
rewrite ^ /bar break；

```

不过，为了避免这种歧义，我还是建议都使用 OpenResty 来处理 rewrite，而不是 NGINX 的配置。说实话，NGINX 的很多配置是比较晦涩的，需要你反复查阅文档才能读懂。

## 第三问，我的代码为什么报错？

Q：在LuaJIT 扩展的table 函数中，为什么下面这两行代码用 LuaJIT 去执行，都会报错“找不到 moudule”呢？我用的LuaJIT 为 2.0.5版本。

```
local new_tab = require('table.new')
# 或者
require('table.clear')

# 执行后会报错
luajit: table_luajit.lua:1: module 'table.new' not found:

```

A：这个问题要注意，这两行代码，需要 LuaJIT 2.1 的版本才能运行, 文档在这里：[https://github.com/LuaJIT/LuaJIT/blob/v2.1/doc/extensions.html#L218](https://github.com/LuaJIT/LuaJIT/blob/v2.1/doc/extensions.html#L218)，可以了解一下。

其实，这也是你在使用 OpenResty 时需要特别留意的。OpenResty 需要特定版本的 LuaJIT 才能正常运行，前面我们也讲过，因为 OpenResty 基于 LuaJIT 2.1 的分支，并且对 LuaJIT 做了不少自己的扩展。

所以，在运行本专栏的代码时，请记得使用OpenResty 官方的安装方式，如果你在 NGINX 的基础上添加 lua-nginx-module 来编译，还是会踩不少坑的。

## 第四问，关于空值的困惑

Q：我遇到一些让人困惑的地方是`ngx.null`、`nil`、`null`和`""`。在网上搜索的时候，看到有人说`null`是`ngx.null`的一个定义。Redis 返回的时候，经常会判断返回结果是否为空，那么，判断的时候是和哪个值进行比较呢？关于这些值，有没有其他一些使用上的坑呢？一直以来我都没有一个明确的认识，想和老师确认一下。

A：在回答你的问题之前，我建议你在 lua-resty-redis 里，使用下面的代码去查找一个 key：

```
local res, err = red:get(&quot;dog&quot;)

```

如果返回值 res 是 nil，就说明函调用失败了；如果 res 是 ngx.null ，就说明redis 中不存在 dog 这个key。这是因为， Lua 的 nil 无法作为 table 的 value，所以 OpenResty 引入了 `ngx.null`，作为 table 中的空值。

我们可以用下面的代码，打印出 `ngx.null` 和它的类型：

```
# 打印ngx.null
$ resty -e  'print(ngx.null)'
null

# 打印类型
$ resty -e 'print(type(ngx.null))'
userdata

```

你可以看到， `ngx.null` 并非`nil`，而是 `userdata` 类型。

更进一步，在 OpenResty 中有很多种空值，比如 `cjson.null`、`cdata:NULL` 等等，后面我都会专门讲到。

总的来说，在 OpenResty 中只有 `nil` 和 `false` 是假值。所以，在你写类似 `if not res then`这种代码的时候，一定要慎之又慎，最好改成明确的 `if res ~= nil and res ~= false then`，用类似这样的写法，并要有对应的测试案例覆盖。

## 第五问：API 网关到底是什么？

Q：文中一直说的 API 网关是指什么？和NGINX、Tomcat、Apache这种Web服务器，又有什么区别呢？

A：API 网关其实是用来统一管理服务的网关。举个例子，像是支付、用户登录等，都是 API 形式对外提供的服务，它们都需要一个网关来做统一的安全和身份认证。

API 网关可以替代传统的 NGINX、Apache 来处理南北向流量，也可以在微服务环境下处理东西向的流量，是更加贴近业务的一种中间件，而非底层的 Web 服务器。

所以，在专栏的最后几篇文章中，我会带着你一起来看下，如何实现一个 API 网关，这是 OpenResty 当前最热门的使用场景之一。

学习是一个需要反复和刻意练习的过程，就像你高中、大学读书的时候一样，能提出问题、敢于提出问题，是吸收知识的重要步骤。希望你能够体会“把书读厚再读薄”的这个学习过程。

最后，欢迎你继续在留言区写下你的疑问，我会持续不断地解答。希望可以通过交流和答疑，帮你把所学转化为所得。也欢迎你把这篇文章转发给你的同事朋友，一起交流、一起进步。


