<audio id="audio" title="26 | 代码贡献者的拦路虎：test::nginx 简介" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/b3/64/b3fb7ed797152551cbee231d8a946864.mp3"></audio>

你好，我是温铭。

测试，是软件开发中必不可少的一个重要环节。测试驱动开发（TDD）的理念已经深入人心，几乎每家软件公司都有 QA 团队来负责测试的工作。

测试也是 OpenResty 质量稳定和好口碑的基石，不过同时，它也是 OpenResty 众多开源项目中最被人忽视的部分。很多开发者每天都在使用 lua-nginx-module，偶尔跑一跑火焰图，但有几个人会去运行测试案例呢？甚至很多基于 OpenResty 的开源项目，都是没有测试案例的。但没有测试案例和持续集成的开源项目，显然是不值得信赖的。

不过，和商业公司不同的是，大部分的开源项目都没有专职的测试工程师，那么它们是如何来保证代码质量的呢？答案很简单，就是“自动化测试”和“持续集成”，关键点在于自动和持续，而OpenResty 在这两个方面都做到了极致。

OpenResty 有 70 个开源项目，它们的单元测试、集成测试、性能测试、mock 测试、fuzz 测试等工作量，是无法靠社区的人力解决的。所以，OpenResty 一开始在自动化测试上的投入就比较大。这样做短期看起来会拖慢项目进度，但可以说是一劳永逸，长期来看在这方面的投入是非常划算的。因此，每当我和其他工程师聊起 OpenResty 在测试方面的思路和工具集时，他们都会惊叹不已。

下面，我们就先来说说OpenResty的测试理念。

## 理念

`test::nginx` 是 OpenResty 测试体系中的核心，OpenResty 本身和周边的 lua-rety 库，都是使用它来组织和编写测试集的。虽然它一个是测试框架，但它的**门槛非常高**。这是因为， `test::nginx` 和一般的测试框架不同，并非基于断言，也不使用 Lua 语言，这就要求开发者从零开始学习和使用 `test::nginx`，并得扭转自身对测试框架固有的认知。

我认识几个 OpenResty 的贡献者，他们可以流畅地给 OpenResty 提交 C 和 Lua 代码，但在使用 `test::nginx` 编写测试用例时都卡壳了，要么不知道怎么写，要么遇到测试跑不过时不知道如何解决。所以，我把 `test::nginx` 称为代码贡献者的拦路虎。

`test::nginx` **糅合了Perl、数据驱动以及 DSL（领域小语言）**。对于同一份测试案例集，通过对参数和环境变量的控制，可以实现乱序执行、多次重复、内存泄漏检测、压力测试等不同的效果。

## 安装和示例

说了这么多概念，让我们来对 `test::nginx` 有一个直观的认识吧。在使用前，我们先来看下如何安装。

关于 OpenResty 体系内软件的安装，只有官方 CI 中的安装方法才是最及时和有效的，其他方式的安装总是会遇到各种各样的问题。所以，我总是推荐你去参考它在 travis 中的[方法](https://github.com/openresty/lua-resty-core/blob/master/.travis.yml)。

`test::nginx` 的安装和使用也不例外，在 travis 中，它可以分为 4 步。

**1. **先安装 Perl 的包管理器 cpanminus。<br>
**2. **然后，通过 cpanm 来安装 `test::nginx`：

```
sudo cpanm --notest Test::Nginx IPC::Run &gt; build.log 2&gt;&amp;1 || (cat build.log &amp;&amp; exit 1)

```

**3. **再接着， clone 最新的源码：

```
git clone https://github.com/openresty/test-nginx.git

```

**4. **最后，通过 Perl 的 `prove` 命令来加载 test-nginx 的库，并运行 `/t` 目录下的测试案例集：

```
prove -Itest-nginx/lib -r t

```

安装完以后，让我们看下 `test::nginx` 中最简单的测试案例。下面这段代码改编自[官方文档](https://metacpan.org/pod/Test::Nginx::Socket)，我已经把个性化的控制参数都去掉了：

```
use Test::Nginx::Socket 'no_plan';


run_tests();

__DATA__

=== TEST 1: set Server
--- config
    location /foo {
        echo hi;
        more_set_headers 'Server: Foo';
    }
--- request
    GET /foo
--- response_headers
Server: Foo
--- response_body
hi

```

虽然 `test::nginx` 是用 Perl 编写的，并且是其中的一个模块，但从上面的测试中，你是不是完全看不到，Perl 或者其他任何其他语言的影子呀？有这个感觉这就对了。因为，`test::nginx` 本身就是作者自己用 Perl 实现的 DSL（小语言），是专门针对 Nginx 和 OpenResty 的测试而抽象出来的。

所以，当你第一次看到这种测试的时候，大概率是看不懂的。不过不用着急，让我来为“你庖丁解牛”，分析以下上面的测试案例吧。

首先是 `use Test::Nginx::Socket;`，这是 Perl 里面引用库的方式，就像 Lua 里面 require 一样。这也在提醒我们，`test::nginx`  是一个 Perl 程序。

第二行的`run_tests();` ，是 `test::nginx`  中的一个 Perl 函数，它是测试框架的入口函数。如果你还想调用 `test::nginx`  中其他的 Perl 函数，都要放在 `run_tests` 之前才有效。

第三行的 `__DATA__` 是一个标记，表示它下面的都是测试数据。Perl 函数都应该在这个标记之前完成。

接下来的 `=== TEST 1: set Server`，是测试案例的标题，是为了注明这个测试的目的，它里面的数字编号有工具可以自动排列。

`--- config` 是 Nginx 配置段。在上面的案例中，我们用的都是 Nginx 的指令，没有涉及到 Lua。如果你要添加 Lua 代码，也是在这里用类似 content_by_lua 的指令完成的。

`--- request` 用于模拟终端来发送一个请求，下面紧跟的 `GET /foo` ，则指明了请求的方法和 URI。

`--- response_headers`，是用来检测响应头的。下面的 `Server: Foo` 表示在响应头中必须出现的 header 和 value，如果没有出现，测试就会失败。

最后的`--- response_body`，是用来检测相应体的。下面的 `hi` 则是响应体中必须出现的字符串，如果没有出现，测试就会失败；

好了，到这里，最简单的测试案例就分析完了，你看明白了吗？如果哪里还不清楚，一定要及时留言提问暴露出来，毕竟，能够看懂测试案例，是完成 OpenResty 相关开发工作的前提。

## 编写自己的测试案例

光说不练假把式，接下来，我们就该进入动手试验环节了。还记得上节课中，我们是如何测试 memcached server 的吗？没错，我们是用 `resty` 来手动发送请求的，也就是用下面这段代码表示：

```
$ resty -e 'local memcached = require &quot;resty.memcached&quot;
    local memc, err = memcached:new()

    memc:set_timeout(1000) -- 1 sec
    local ok, err = memc:connect(&quot;127.0.0.1&quot;, 11212)
    local ok, err = memc:set(&quot;dog&quot;, 32)
    if not ok then
        ngx.say(&quot;failed to set dog: &quot;, err)
        return
    end

    local res, flags, err = memc:get(&quot;dog&quot;)
    ngx.say(&quot;dog: &quot;, res)'

```

不过，是不是觉得手动发送还不够智能呢？没关系，在学习完 `test::nginx`  之后，我们就可以尝试把手动的测试变为自动化的了，比如下面这段代码：

```
use Test::Nginx::Socket::Lua::Stream;

run_tests();

__DATA__
  
=== TEST 1: basic get and set
--- config
        location /test {
            content_by_lua_block {
                local memcached = require &quot;resty.memcached&quot;
                local memc, err = memcached:new()
                if not memc then
                    ngx.say(&quot;failed to instantiate memc: &quot;, err)
                    return
                end

                memc:set_timeout(1000) -- 1 sec
                local ok, err = memc:connect(&quot;127.0.0.1&quot;, 11212)

                local ok, err = memc:set(&quot;dog&quot;, 32)
                if not ok then
                    ngx.say(&quot;failed to set dog: &quot;, err)
                    return
                end

                local res, flags, err = memc:get(&quot;dog&quot;)
                ngx.say(&quot;dog: &quot;, res)
            }
        }

--- stream_config
    lua_shared_dict memcached 100m;

--- stream_server_config
    listen 11212;
    content_by_lua_block {
        local m = require(&quot;memcached-server&quot;)
        m.go()
    }

--- request
GET /test
--- response_body
dog: 32
--- no_error_log
[error]

```

在这个测试案例中，我新增了 `--- stream_config`、`--- stream_server_config`、`--- no_error_log` 这些配置项，但它们的本质上都是一样的，即：

**通过抽象好的原语（也可以看做配置），把测试的数据和检测进行剥离，让可读性和扩展性变得更好。**

这就是 `test::nginx` 和其他测试框架的根本不同之处。这种 DSL 是一把双刃剑，它可以让测试逻辑变得清晰和方便扩展，但同时也提高了学习的门槛，你需要重新学习新的语法和配置才能开始编写测试案例。

## 写在最后

不得不说，`test::nginx`  虽然强大，但很多时候，它可能不一定适合你的场景。杀鸡焉用宰牛刀？在 OpenResty 中，你也选择使用断言风格的测试框架 `busted`。`busted`结合 `resty` 这个命令行工具，也可以满足不少测试的需求。

最后，给你留一个作业题，你可以在本地把 memcached 的这个测试跑起来吗？如果你能新增一个测试案例，那就更棒了。

欢迎在留言区记录你的操作和心得，也可以写下你今天学习的疑惑地方。同时，欢迎你把这篇文章分享给更多对OpenResty感兴趣的人，我们一起交流和探讨。


