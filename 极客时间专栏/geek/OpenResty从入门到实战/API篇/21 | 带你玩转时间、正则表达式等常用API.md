<audio id="audio" title="21 | 带你玩转时间、正则表达式等常用API" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/3f/7b/3fbd10a52dad9e660748b05f114fde7b.mp3"></audio>

你好，我是温铭。在前面几节课中，你已经熟悉了不少OpenResty 中重要的 Lua API 了，今天我们再来了解下其他一些通用的 API，主要和正则表达式、时间、进程等相关。

## 正则

先来看下最常用，也是最重要的正则。在 OpenResty 中，我们应该一直使用 `ngx.re.*` 提供的一系列 API，来处理和正则表达式相关的逻辑，而不是用 Lua 自带的模式匹配。这不仅是出于性能方面的考虑，还因为Lua 自带的正则是自成体系的，并非 PCRE 规范，这对于绝大部分开发者来说都是徒增烦恼。

在前面的课程中，你已经多多少少接触过一些 `ngx.re.*` 的 API了，文档也写得非常详细，我就不再一一列举了。这里，我再单独强调两个内容。

### ngx.re.split

第一个是`ngx.re.split`。字符串切割是很常见的功能，OpenResty 也提供了对应的 API，但在社区的 QQ 交流群中，很多开发者都找不到这样的函数，只能选择自己手写。

为什么呢？其实， `ngx.re.split` 这个 API 并不在 lua-nginx-module 中，而是在 lua-resty-core 里面；并且它也不在 lua-resty-core 首页的文档中，而是在 `lua-resty-core/lib/ngx/re.md` 这个第三级目录的文档中出现的。多种原因，导致很多开发者完全不知道这个 API 的存在。

类似这种“藏在深闺无人识“的 API，还有我们前面提到过的 `ngx_resp.add_header`、`enable_privileged_agent` 等等。那么怎么来最快地解决这种问题呢？除了阅读 lua-resty-core 首页文档外，你还需要把 `lua-resty-core/lib/ngx/` 这个目录下的 `.md` 格式的文档也通读一遍才行。

我们前面夸了很多 OpenResty 文档做得好的地方，不过，这一点上，也就是在一个页面能够查询到完整的 API 列表，确实还有很大的改进空间。

### lua_regex_match_limit

第二个，我想介绍一下`lua_regex_match_limit`。我们之前并没有花专门的篇幅，来讲 OpenResty 提供的 Nginx 指令，因为大部分情况下我们使用默认值就足够了，它们也没有在运行时去修改的必要性。不过，我们今天要讲的这个，和正则表达式相关的`lua_regex_match_limit` 指令，却是一个例外。

我们知道，如果我使用的正则引擎是基于回溯的 NFA 来实现的，那么就有可能出现灾难性回溯（Catastrophic Backtracking），即正则在匹配的时候回溯过多，造成 CPU 100%，正常服务被阻塞。

一旦发生灾难性回溯，我们就需要用 gdb 分析 dump，或者 systemtap 分析线上环境才能定位，而且事先也不容易发现，因为只有特别的请求才会触发。这显然就给攻击者带来了可趁之机，ReDoS（RegEx Denial of Service）就是指的这类攻击。

如果你对如何自动化发现和彻底解决这个问题感兴趣，可以参考我之前在公众号写的一篇文章：[如何彻底避免正则表达式的灾难性回溯](https://mp.weixin.qq.com/s/K9d60kjDdFn6ZwIdsLjqOw)？

今天在这里，我主要给你介绍下，如何在 OpenResty 中简单有效地规避，也就是使用下面这行代码：

```
lua_regex_match_limit 100000;

```

`lua_regex_match_limit` ，就是用来限制 PCRE 正则引擎的回溯次数的。这样，即使出现了灾难性回溯，后果也会被限制在一个范围内，不会导致你的 CPU 满载。

这里我简单说一下，这个指令的默认值是 0，也就是不做限制。如果你没有替换 OpenResty 自带的正则引擎，并且还涉及到了比较多的复杂的正则表达式，你可以考虑重新设置这个 Nginx 指令的值。

## 时间 API

接下来我们说说时间 API。OpenResty 提供了 10 个左右和时间相关的 API，从这个数量你也可见它的重要性。一般来说，最常用的时间 API就是 `ngx.now`，它可以打印出当前的时间戳，比如下面这行代码：

```
resty -e 'ngx.say(ngx.now())'

```

从打印的结果可以看出，`ngx.now` 包括了小数部分，所以更加精准。而与之相关的 `ngx.time` 则只返回了整数部分的值。至于其他的 `ngx.localtime`、`ngx.utctime`、`ngx.cookie_time` 和 `ngx.http_time` ，主要是返回和处理时间的不同格式。具体用到的话，你可以查阅文档，本身并不难理解，我就没有必要专门来讲了。

不过，值得一提的是，**这些返回当前时间的 API，如果没有非阻塞网络 IO 操作来触发，便会一直返回缓存的值，而不是像我们想的那样，能够返回当前的实时时间**。可以看看下面这个示例代码：

```
$ resty -e 'ngx.say(ngx.now())
os.execute(&quot;sleep 1&quot;)
ngx.say(ngx.now())'

```

在两次调用 `ngx.now` 之间，我们使用 Lua 的阻塞函数 sleep 了 1 秒钟，但从打印的结果来看，这两次返回的时间戳却是一模一样的。

那么，如果换成是非阻塞的 sleep 函数呢？比如下面这段新的代码：

```
$ resty -e 'ngx.say(ngx.now())
ngx.sleep(1)
ngx.say(ngx.now())'

```

显然，它就会打印出不同的时间戳了。这里顺带引出了 `ngx.sleep` ，这个非阻塞的 sleep 函数。这个函数除了可以休眠指定的时间外，还有另外一个特别的用处。

举个例子，比如你有一段正在做密集运算的代码，需要花费比较多的时间，那么在这段时间内，这段代码对应的请求就会一直占用着 worker 和 CPU 资源，导致其他请求需要排队，无法得到及时的响应。这时，我们就可以在其中穿插 `ngx.sleep(0)`，使这段代码让出控制权，让其他请求也可以得到处理。

## worker 和进程 API

再来看worker 和进程相关的API。OpenResty 提供了 `ngx.worker.*` 和 `ngx.process.*` 这些 API， 来获取 worker 和进程相关的信息。其中，前者和 Nginx worker 进程有关，后者则是泛指所有的 Nginx 进程，不仅有 worker 进程，还有 master 进程和特权进程等等。

事实上，`ngx.worker.*` 由 lua-nginx-module 提供，而`ngx.process.*` 则是由 lua-resty-core 提供。还记得上节课我们留的作业题吗，如何保证在多 worker 的情况下，只启动一个 timer？其实，这就需要用到 `ngx.worker.id` 这个 API 了。你可以在启动 timer 之前，先做一个简单的判断：

```
if ngx.worker.id == 0 then
    start_timer()
end

```

这样，我们就能实现只启动一个 timer的目的了。这里注意，worker id 是从 0 开始返回的，这和 Lua 中数组下标从 1 开始并不相同，千万不要混淆了。

至于其他 worker 和 process 相关的 API，并没有什么特别需要注意的地方，就交给你自己去学习和练习了。

## 真值和空值

最后我们来看看，真值和空值的问题。在 OpenResty 中，真值与空值的判断，一直是个让人头痛、也比较混乱的点。

我们先看来下 Lua 中真值的定义：**除了 nil 和 false 之外，都是真值。**

所以，真值也就包括了：0、空字符串、空表等等。

再来看下 Lua 中的空值（nil），它是未定义的意思，比如你申明了一个变量，但还没有初始化，它的值就是 nil：

```
$ resty -e 'local a
ngx.say(type(a))'

```

而 nil 也是 Lua 中的一种数据类型。

明白了这两点后，我们现在就来具体看看，基于这两个定义，衍生出来的其他坑。

### ngx.null

第一个坑是`ngx.null`。因为 Lua 的 nil 无法作为 table 的 value，所以 OpenResty 引入了 `ngx.null`，作为 table 中的空值：

```
$ resty -e  'print(ngx.null)'
null

```

```
$ resty -e 'print(type(ngx.null))'
userdata

```

从上面两段代码你可以看出，`ngx.null` 被打印出来是 null，而它的类型是 userdata。那么，可以把它当作假值吗？当然不行，事实上，`ngx.null` 的布尔值为真：

```
$ resty -e 'if ngx.null then
ngx.say(&quot;true&quot;)
end'

```

所以，要谨记，**只有 nil 和 false 是假值**。如果你遗漏了这一点，就很容易踩坑，比如你在使用 lua-resty-redis 的时候，做了下面这个判断：

```
local res, err = red:get(&quot;dog&quot;)
if not res then
    res = res + &quot;test&quot;
end 

```

如果返回值 res 是 nil，就说明函数调用失败了；如果 res 是 ngx.null，就说明 redis 中不存在 `dog` 这个key。那么，在 `dog` 这个 key 不存在的情况下，这段代码就 500 崩溃了。

### cdata:NULL

第二个坑是`cdata:NULL`。当你通过 LuaJIT FFI 接口去调用 C 函数，而这个函数返回一个 NULL 指针，那么你就会遇到另外一种空值，即`cdata:NULL` 。

```
$ resty -e 'local ffi = require &quot;ffi&quot;
local cdata_null = ffi.new(&quot;void*&quot;, nil)
if cdata_null then
    ngx.say(&quot;true&quot;)
end'

```

和 `ngx.null` 一样，`cdata:NULL` 也是真值。但更让人匪夷所思的是，下面这段代码，会打印出 true，也就是说`cdata:NULL` 是和 `nil` 相等的：

```
$ resty -e 'local ffi = require &quot;ffi&quot;
local cdata_null = ffi.new(&quot;void*&quot;, nil)
ngx.say(cdata_null == nil)'

```

那么我们应该如何处理 `ngx.null` 和 `cdata:NULL` 呢？显然，让应用层来关心这些闹心事儿是不现实的，最好是做一个二层封装，不要让调用者知道这些细节即可。

### cjson.null

最后，我们再来看下 cjson 中出现的空值。cjson 库会把 json 中的 NULL，解码为 Lua 的 `lightuserdata`，并用 `cjson.null` 来表示：

```
$ resty -e 'local cjson = require &quot;cjson&quot;
local data = cjson.encode(nil)
local decode_null = cjson.decode(data)
ngx.say(decode_null == cjson.null)'

```

Lua 中的 nil，被 json encode 和 decode 一圈儿之后，就变成了 `cjson.null`。你可以想得到，它引入的原因和 `ngx.null` 是一样的，因为 nil 无法在 table 中作为 value。

到现在为止，看了这么多 OpenResty 中的空值，不知道你蒙圈儿了没？不要慌张，这部分内容多看几遍，自己梳理一下，就不至于晕头转向分不清了。当然，你以后在写类似 `if not foo then` 的时候，就要多想想，这个条件到底能不能成立了。

## 写在最后

学完今天这节课后，OpenResty 中常用的 Lua API 我们就都介绍过了，不知道你是否都清楚了呢？

最后，留一个思考题给你：在 `ngx.now` 的示例中，为什么在没有 yield 操作的时候，它的值不会修改呢？欢迎留言分享你的看法，也欢迎你把这篇文章分享出去，我们一起交流，一起进步。


