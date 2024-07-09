<audio id="audio" title="20 | 超越 Web 服务器：特权进程和定时任务" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/8e/38/8e18455afada7424d471e975ed42ff38.mp3"></audio>

你好，我是温铭。

前面我们介绍了 OpenResty API、共享字典缓存和 cosocket。它们实现的功能，都还在 Nginx 和 Web 服务器的范畴之内，算是提供了开发成本更低、更容易维护的一种实现，提供了可编程的 Web 服务器。

不过，OpenResty并不满足于此。我们今天就挑选几个，OpenResty 中超越 Web 服务器的功能来介绍一下。它们分别是定时任务、特权进程和非阻塞的 ngx.pipe。

## 定时任务

在 OpenResty 中，我们有时候需要在后台定期地执行某些任务，比如同步数据、清理日志等。如果让你来设计，你会怎么做呢？最容易想到的方法，便是对外提供一个 API 接口，在接口中完成这些任务；然后用系统的 crontab 定时调用 curl，来访问这个接口，进而曲线地实现这个需求。

不过，这样一来不仅会有割裂感，也会给运维带来更高的复杂度。所以， OpenResty 提供了 `ngx.timer` 来解决这类需求。你可以把`ngx.timer` ，看作是 OpenResty 模拟的客户端请求，用以触发对应的回调函数。

其实，OpenResty 的定时任务可以分为下面两种：

- `ngx.timer.at`，用来执行一次性的定时任务；
- `ngx.time.every`，用来执行固定周期的定时任务。

还记得上节课最后我留下的思考题吗？问题是如何突破 `init_worker_by_lua` 中不能使用 cosocket 的限制，这个答案其实就是 `ngx.timer`。

下面这段代码，就是启动了一个延时为 0 的定时任务。它启动了回调函数 `handler`，并在这个函数中，用 cosocket 去访问一个网站：

```
init_worker_by_lua_block {
        local function handler()
            local sock = ngx.socket.tcp()
            local ok, err = sock:connect(“www.baidu.com&quot;, 80)
        end

        local ok, err = ngx.timer.at(0, handler)
    }

```

这样，我们就绕过了 cosocket 在这个阶段不能使用的限制。

再回到这部分开头时我们提到的的用户需求，`ngx.timer.at` 并没有解决周期性运行这个需求，在上面的代码示例中，它是一个一次性的任务。

那么，又该如何做到周期性运行呢？表面上来看，基于 `ngx.timer.at` 这个API 的话，你有两个选择：

- 你可以在回调函数中，使用一个 while true 的死循环，执行完任务后 sleep 一段时间，自己来实现周期任务；
- 你还可以在回调函数的最后，再创建另外一个新的 timer。

不过，在做出选择之前，有一点我们需要先明确下：timer 的本质是一个请求，虽然这个请求不是终端发起的；而对于请求来讲，在完成自己的任务后它就要退出，不能一直常驻，否则很容易造成各种资源的泄漏。

所以，第一种使用 while true 来自行实现周期任务的方案并不靠谱。第二种方案虽然是可行的，但递归地创建 timer ，并不容易让人理解。

那么，是否有更好的方案呢？其实，OpenResty 后面新增的 `ngx.time.every` API，就是专门为了解决这个问题而出现的，它是更加接近 crontab 的解决方案。

但美中不足的是，在启动了一个 timer 之后，你就再也没有机会来取消这个定时任务了，毕竟`ngx.timer.cancel` 还是一个 todo 的功能。

这时候，你就会面临一个问题：定时任务是在后台运行的，并且无法取消；如果定时任务的数量很多，就很容易耗尽系统资源。

所以，OpenResty 提供了 `lua_max_pending_timers` 和 `lua_max_running_timers` 这两个指令，来对其进行限制。前者代表等待执行的定时任务的最大值，后者代表当前正在运行的定时任务的最大值。

你也可以通过 Lua API，来获取当前等待执行和正在执行的定时任务的值，下面是两个示例：

```
content_by_lua_block {
            ngx.timer.at(3, function() end)
            ngx.say(ngx.timer.pending_count())
        }

```

这段代码会打印出 1，表示有 1 个计划任务正在等待被执行。

```
content_by_lua_block {
            ngx.timer.at(0.1, function() ngx.sleep(0.3) end)
            ngx.sleep(0.2)
            ngx.say(ngx.timer.running_count())
        }

```

这段代码会打印出 1，表示有 1 个计划任务正在运行中。

## 特权进程

接着来看特权进程。我们都知道 Nginx 主要分为 master 进程和 worker 进程，其中，真正处理用户请求的是 worker 进程。我们可以通过 `lua-resty-core` 中提供的 `process.type` API ，获取到进程的类型。比如，你可以用 `resty` 运行下面这个函数：

```
$ resty -e 'local process = require &quot;ngx.process&quot;
ngx.say(&quot;process type:&quot;, process.type())'

```

你会看到，它返回的结果不是 `worker`， 而是 `single`。这意味 `resty` 启动的 Nginx 只有 worker 进程，没有 master 进程。其实，事实也是如此。在 `resty` 的实现中，你可以看到，下面这样的一行配置， 关闭了 master 进程：

```
master_process off;

```

而OpenResty 在 Nginx 的基础上进行了扩展，增加了特权进程：privileged agent。特权进程很特别：

- 它不监听任何端口，这就意味着不会对外提供任何服务；
- 它拥有和 master 进程一样的权限，一般来说是 `root` 用户的权限，这就让它可以做很多 worker 进程不可能完成的任务；
- 特权进程只能在 `init_by_lua` 上下文中开启；
- 另外，特权进程只有运行在 `init_worker_by_lua` 上下文中才有意义，因为没有请求触发，也就不会走到`content`、`access` 等上下文去。

下面，我们来看一个开启特权进程的示例：

```
init_by_lua_block {
    local process = require &quot;ngx.process&quot;

    local ok, err = process.enable_privileged_agent()
    if not ok then
        ngx.log(ngx.ERR, &quot;enables privileged agent failed error:&quot;, err)
    end
}

```

通过这段代码开启特权进程后，再去启动 OpenResty 服务，我们就可以看到，Nginx 的进程中多了特权进程的身影：

```
nginx: master process
nginx: worker process
nginx: privileged agent process

```

不过，如果特权只在 `init_worker_by_lua` 阶段运行一次，显然不是一个好主意，那我们应该怎么来触发特权进程呢？

没错，答案就藏在刚刚讲过的知识里。既然它不监听端口，也就是不能被终端请求触发，那就只有使用我们刚才介绍的 `ngx.timer` ，来周期性地触发了：

```
init_worker_by_lua_block {
    local process = require &quot;ngx.process&quot;

    local function reload(premature)
        local f, err = io.open(ngx.config.prefix() .. &quot;/logs/nginx.pid&quot;, &quot;r&quot;)
        if not f then
            return
        end
        local pid = f:read()
        f:close()
        os.execute(&quot;kill -HUP &quot; .. pid)
    end

    if process.type() == &quot;privileged agent&quot; then
         local ok, err = ngx.timer.every(5, reload)
        if not ok then
            ngx.log(ngx.ERR, err)
        end
    end
}

```

上面这段代码，实现了每 5 秒给 master 进程发送 HUP 信号量的功能。自然，你也可以在此基础上实现更多有趣的功能，比如轮询数据库，看是否有特权进程的任务并执行。因为特权进程是 root 权限，这显然就有点儿“后门”程序的意味了。

## 非阻塞的 ngx.pipe

最后我们来看非阻塞的 ngx.pipe。刚刚讲过的这个代码示例中，我们使用了 Lua 的标准库，来执行外部命令行，把信号发送给了 master 进程：

```
os.execute(&quot;kill -HUP &quot; .. pid) 

```

这种操作自然是会阻塞的。那么，在 OpenResty 中，是否有非阻塞的方法来调用外部程序呢？毕竟，要知道，如果你是把 OpenResty 当做一个完整的开发平台，而非 Web 服务器来使用的话，这就是你的刚需了。

为此，`lua-resty-shell` 库应运而生，使用它来调用命令行就是非阻塞的：

```
$ resty -e 'local shell = require &quot;resty.shell&quot;
local ok, stdout, stderr, reason, status =
    shell.run([[echo &quot;hello, world&quot;]])
    ngx.say(stdout)

```

这段代码可以算是 hello world 的另外一种写法了，它调用系统的 `echo` 命令来完成输出。类似的，你可以用 `resty.shell` ，来替代 Lua 中的 `os.execute` 调用。

我们知道，`lua-resty-shell` 的底层实现，依赖了 `lua-resty-core` 中的 [[ngx.pipe](https://github.com/openresty/lua-resty-core/blob/master/lib/ngx/pipe.md)] API，所以，这个使用 `lua-resty-shell` 打印出 `hello wrold` 的示例，改用 `ngx.pipe` ，可以写成下面这样：

```
$ resty -e 'local ngx_pipe = require &quot;ngx.pipe&quot;
local proc = ngx_pipe.spawn({&quot;echo&quot;, &quot;hello world&quot;})
local data, err = proc:stdout_read_line()
ngx.say(data)'

```

这其实也就是 `lua-resty-shell` 底层的实现代码了。你可以去查看 `ngx.pipe` 的文档和测试案例，来获取更多的使用方法，这里我就不再赘述了。

## 写在最后

到此，今天的主要内容我就讲完了。从上面的几个功能，我们可以看出，OpenResty 在做一个更好用的 Nginx 的前提下，也在尝试往通用平台的方向上靠拢，希望开发者能够尽量统一技术栈，都用 OpenResty 来解决开发需求。这对于运维来说是相当友好的，因为只要部署一个 OpenResty 就可以了，维护成本更低。

最后，给你留一个思考题。由于可能会存在多个 Nginx worker，那么 timer 就会在每个 worker 中都运行一次，这在大多数场景下都是不能接受的。我们应该如何保证 timer 只能运行一次呢？

欢迎留言说说你的解决方法，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。


