<audio id="audio" title="27 | test::nginx 包罗万象的测试方法" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/dc/9d/dcf24f0a273be28566f2b8424e0ef39d.mp3"></audio>

你好，我是温铭。

通过上节课的学习，你已经对 `test::nginx` 有了一个初步的认识，并运行了最简单的示例。不过，在实际的开源项目中，`test::nginx` 编写的测试案例显然要比示例代码复杂得多，也更加难以掌握，不然它也就称不上是拦路虎了。

在本节课中，我会带你来熟悉下 `test::nginx` 中经常用到的指令和测试方法，目的是让你可以看明白 OpenResty 项目中大部分的测试案例集，并有能力来编写更真实的测试案例。即使你还没有给 OpenResty 贡献过代码，但熟悉了 OpenResty 的测试框架，对于你平时工作中设计和编写测试案例，还是会有不少启发的。

`test::nginx` 的测试，本质上是根据每一个测试案例的配置，先去生成 nginx.conf，并启动一个 Nginx 进程；然后，模拟客户端发起请求，其中包含指定的请求体和请求头；紧接着，测试案例中的 Lua 代码会处理请求并作出响应，这时，`test::nginx` 解析响应体、响应头、错误日志等关键信息，并和测试配置做对比。如果发现不符，就报错退出，测试失败；否则就算成功。

`test::nginx` 中提供了很多 DSL 的原语，我按照 Nginx 配置、发送请求、处理响应、检查日志这个流程，做了一个简单的分类。这  20% 的功能可以覆盖 80% 的应用场景，所以你一定要牢牢掌握。至于其他更高级的原语和使用方法，我们留到下一节再来介绍。

## Nginx 配置

我们首先来看下 Nginx 配置。`test::nginx` 的原语中带有 `config` 这个关键字的，就和 Nginx 配置相关，比如上一节中提到的 `config`、`stream_config`、`http_config` 等。

它们的作用都是一样的，即在 Nginx 的不同上下文中，插入指定的 Nginx 配置。这些配置可以是 Nginx 指令，也可以是 `content_by_lua_block` 封装起来的 Lua 代码。

在做单元测试的时候，`config` 是最常用的原语，我们会在其中加载 Lua 库，并调用函数来做白盒测试。下面是节选的一段测试代码，并不能完整运行。它来自一个真实的开源项目，如果你对此有兴趣，可以点击[链接](https://github.com/iresty/apisix/blob/master/t/plugin/key-auth.t#L11)查看完整的测试，也可以尝试在本机运行。

```
=== TEST 1: sanity
--- config
    location /t {
        content_by_lua_block {
            local plugin = require(&quot;apisix.plugins.key-auth&quot;)
            local ok, err = plugin.check_schema({key = 'test-key'})
            if not ok then
                ngx.say(err)
            end
            ngx.say(&quot;done&quot;)
        }
    }

```

这个测试案例的目的，是为了测试代码文件 `plugins.key-auth` 中， `check_schema` 这个函数能否正常工作。它在`location /t` 中使用 `content_by_lua_block` 这个 Nginx 指令，require 需要测试的模块，并直接调用需要检查的函数。

这就是在 `test::nginx` 进行白盒测试的通用手段。不过，只有这段配置自然是无法完成测试的，下面我们继续看下，如何发起客户端的请求。

## 发送请求

模拟客户端发送请求，会涉及到不少的细节，所以，我们就先从最简单的发送单个请求入手吧。

### request

还是继续上面的测试案例，如果你想要单元测试的代码被运行，那就要发起一个 HTTP 请求，访问的地址是 config 中注明的 `/t`，正如下面的测试代码所示：

```
--- request
GET /t

```

这段代码在 `request` 原语中，发起了一个 GET 请求，地址是 `/t`。这里，我们并没有注明访问的 ip 地址、域名和端口，也没有指定是 HTTP 1.0 还是 HTTP 1.1，这些细节都被 `test::nginx` 隐藏了，你不用去关心。这就是 DSL 的好处之一——你只需要关心业务逻辑，不用被各种细节所打扰。

同时，这也提供了部分的灵活性。比如默认是 HTTP 1.1 的协议，如果你想测试 HTTP 1.0，也可以单独指定：

```
--- request
GET /t  HTTP/1.0


```

除了 GET 方法之外，POST 方法也是需要支持的。下面这个示例，可以 POST `hello world` 这个字符串到指定的地址：

```
--- request
POST /t  
hello world

```

同样的， `test::nginx` 在这里为你自动计算了请求体长度，并自动增加了 `host` 和 `connection` 这两个请求头，以保证这是一个正常的请求。

当然，出于可读性的考虑，你可以在其中增加注释。以 `#` 开头的，就会被识别为代码注释：

```
--- request
   # post request
POST /t  
hello world

```

`request` 还支持更为复杂和灵活的模式，那就是配合 `eval` 这个 filter，直接嵌入 perl 代码，毕竟 `test::nginx` 就是perl 编写的。这种做法，类似于在 DSL 之外开了一个后门，如果当前的 DSL 原语都不能满足你的需求，那么 `eval` 这种直接执行 perl 代码的方法，就可以说是“终极武器”了。

关于 `eval`的用法，这里我们先看几个简单的例子，其他更复杂的，我们下节课继续介绍：

```
--- request eval
&quot;POST /t
hello\x00\x01\x02
world\x03\x04\xff&quot;

```

第一个例子中，我们用 `eval` 来指定不可打印的字符，这也是它的用处之一。双引号之间的内容，会被当做 perl 的字符串来处理后，再传给 `request` 来作为参数。

下面是一个更有趣的例子：

```
--- request eval
&quot;POST /t\n&quot; . &quot;a&quot; x 1024

```

不过，要看懂这个例子，需要懂一些 perl 的字符串知识，这里我简单提两句：

- 在 perl 中，我们用一个点号来表示字符串拼接，这是不是和 Lua 的两个点号有些类似呢？
- 用小写的 x 来表示字符的重复次数。比如上面的 `"a" x 1024`，就表示字符 a 重复 1024 次。

所以，第二个例子的含义是，用 POST 方法，向 `/t` 地址，发送包含 1024 个字符 a 的请求。

### pipelined_requests

了解完如何发送单个请求后，我们再来看下如何发送多个请求。在 `test::nginx` 中，你可以使用 `pipelined_requests` 这个原语，在同一个 `keep-alive` 的连接里面，依次发送多个请求：

```
--- pipelined_requests eval
[&quot;GET /hello&quot;, &quot;GET /world&quot;, &quot;GET /foo&quot;, &quot;GET /bar&quot;]

```

比如这个示例就会在同一个连接中，依次访问这 4 个接口。这样做会有两个好处：

- 第一是可以省去不少重复的测试代码，把 4 个测试案例压缩到一个测试案例中完成；
- 第二也是最重要的原因，你可以用流水线的请求，来检测代码逻辑在多次访问的情况下，是否会有异常。

你可能会奇怪，我依次写多个测试案例，那么执行的时候，代码也会被多次执行，不也可以覆盖上面的第二个问题吗？

其实，这就涉及到 `test::nginx` 的执行模式了，它并非像你想象中的那样去运转。事实上，在执行完每一个测试案例后， `test::nginx` 都会关闭当前的 Nginx 进程，自然的，内存中所有数据也都随之消失了。当运行下一个测试案例时，又会重新生成 `nginx.conf`，并启动新的 Nginx worker。这种机制是为了保证测试案例之间不会互相影响。

所以，当你要测试多个请求时，就需要用到 `pipelined_requests` 这个原语了。基于它，你可以模拟出限流、限速、限并发等多种情况，用更真实和复杂的场景来检测你的系统是否正常。这一点，我们也留在下节课继续拆解，因为它会涉及到多个指令和原语的配合。

### repeat_each

刚才我们提到了测试多个请求的情况，那么应该如何对同一个测试执行多次呢？

针对这个问题，`test::nginx` 提供了一个全局的设置：`repeat_each`。它其实是一个 perl 函数，默认情况下是 `repeat_each(1)`，表示测试案例只运行一次。所以之前的测试案例中，我们都没有去单独设置它。

自然，你可以在 `run_test()` 函数之前来设置它，比如将参数改为2：

```
repeat_each(2);
run_tests();

```

那么，每个测试案例就都会被运行两次，以此类推。

### more_headers

聊完了请求体，我们再来看下请求头。上面我们提到，`test::nginx` 在发送请求的时候，默认会带上 `host` 和 `connection` 这两个请求头。那么其他的请求头如何设置呢？

其实，`more_headers` 就是专门做这件事儿的：

```
--- more_headers
X-Foo: blah

```

你可以用它来设置各种自定义的头。如果想设置多个头，那设置多行就可以了：

```
--- more_headers
X-Foo: 3
User-Agent: openresty

```

## 处理响应

发送完请求后，`test::nginx` 中最重要的部分就来了，那就是处理响应，我们会在这里判断响应是否符合预期。这里我们分为 4 个部分依次介绍，分别是响应体、响应头、响应码和日志。

### response_body

与 `request` 原语对应的就是 `response_body`，下面是它们两个配置使用的例子：

```
=== TEST 1: sanity
--- config
    location /t {
        content_by_lua_block {
            ngx.say(&quot;hello&quot;)
        }
    }
--- request
GET /t
--- response_body
hello


```

这个测试案例，在响应体是 `hello` 的情况下会通过，其他情况就会报错。但如何返回体很长，我们怎么检测才合适呢？别着急，`test::nginx` 已经为你考虑好了，它支持用用正则表达式来检测响应体，比如下面这样的写法：

```
--- response_body_like
^he\w+$

```

这样你就可以对响应体进行非常灵活的检测了。不仅如此，`test::nginx` 还支持 unlike 的操作：

```
--- response_body_unlike
^he\w+$

```

这时候，如果响应体是`hello`，测试就不能通过了。

同样的思路，了解完单个请求的检测后，我们再来看下多个请求的检测。下面是配合 `pipelined_requests` 一起使用的示例：

```
--- pipelined_requests eval
[&quot;GET /hello&quot;, &quot;GET /world&quot;, &quot;GET /foo&quot;, &quot;GET /bar&quot;]
--- response_body eval
[&quot;hello&quot;, &quot;world&quot;, &quot;oo&quot;, &quot;bar&quot;]

```

当然，这里需要注意的是，你发送了多少个请求，就需要有多少个响应来对应。

### response_headers

第二个我们来说说响应头。响应头和请求头类似，每一行对应一个 header 的 key 和 value：

```
--- response_headers
X-RateLimit-Limit: 2
X-RateLimit-Remaining: 1

```

和响应体的检测一样，响应头也支持正则表达式和 unlike 操作，分别是 `response_headers_like` 、`raw_response_headers_like` 和 `raw_response_headers_unlike`。

### error_code

第三个来看响应码。响应码的检测支持直接的比较，同时也支持 like 操作，比如下面两个示例：

```
--- error_code: 302


```

```
--- error_code_like: ^(?:500)?$

```

而对于多个请求的情况，`error_code` 自然也需要检测多次：

```
--- pipelined_requests eval
[&quot;GET /hello&quot;, &quot;GET /hello&quot;, &quot;GET /hello&quot;, &quot;GET /hello&quot;]
--- error_code eval
[200, 200, 503, 503]

```

### error_log

最后一个检测项，就是错误日志了。在大部分的测试案例中，都不会产生错误日志。我们可以用 `no_error_log` 来检测：

```
--- no_error_log
[error]

```

在上面的例子中，如果 Nginx 的错误日志 error.log 中，出现 `[error]` 这个字符串，测试就会失败。这是一个很常用的功能，建议在你正常的测试中，都加上对错误日志的检测。

自然，另一方面，我们也需要编写很多异常的测试案例，以便验证在出错的情况下，我们的代码是否正常处理。这种情况下，我们就需要错误日志中出现指定的字符串，这就是 `error_log` 的用武之地了：

```
--- error_log
hello world

```

上面这段配置，其实就在检测 error.log 中是否出现了 `hello world`。当然，你可以在其中，用 `eval` 嵌入 perl 代码的方式，来实现正则表达式的检测，比如下面这样的写法：

```
--- error_log eval
qr/\[notice\] .*?  \d+ hello world/

```

## 写在最后

今天，我们学习的是如何在 `test::nginx` 中发送请求和检测响应，包含了 body、header、响应码和错误日志等。通过这些原语的组合，你可以实现比较完整的测试案例集。

最后，给你留一个思考题：`test::nginx` 这种抽象一层的 DSL，你觉得有什么优势和劣势吗？欢迎留言和我探讨，也欢迎你把这篇文章分享出去，一起交流和思考。


