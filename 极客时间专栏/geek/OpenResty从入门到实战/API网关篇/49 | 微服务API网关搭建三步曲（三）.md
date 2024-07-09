<audio id="audio" title="49 | 微服务API网关搭建三步曲（三）" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/d4/c7/d4b1aa87b212a3023fc52be2116562c7.mp3"></audio>

你好，我是温铭。

今天这节课，微服务 API 网关搭建就到了最后的环节了。让我们用一个最小的示例来把之前选型的组件，按照设计的蓝图，拼装运行起来吧！

## Nginx 配置和初始化

我们知道，API 网关是用来处理流量入口的，所以我们首先需要在 Nginx.conf 中做简单的配置，让所有的流量都通过网关的 Lua 代码来处理。

```
server {
    listen 9080;

    init_worker_by_lua_block {
        apisix.http_init_worker()
    }

    location / {
        access_by_lua_block {
            apisix.http_access_phase()
        }
        header_filter_by_lua_block {
            apisix.http_header_filter_phase()
        }
        body_filter_by_lua_block {
            apisix.http_body_filter_phase()
        }
        log_by_lua_block {
            apisix.http_log_phase()
        }
    }
}

```

这里我们使用开源 API 网关 [APISIX](https://github.com/apache/apisix)  为例，所以上面的代码示例中带有 `apisix` 的关键字。在这个示例中，我们监听了 9080 端口，并通过 `location /` 的方式，把这个端口的所有请求都拦截下来，并依次通过 `access`、`rewrite`、`header filter`、`body filter` 和 `log` 这几个阶段进行处理，在每个阶段中都会去调用对应的插件函数。其中， `rewrite` 阶段便是在 `apisix.http_access_phase` 函数中合并处理的。

而对于系统初始化的工作，我们放在了 `init_worker` 阶段来处理，这其中包含了读取各项配置参数、预制 etcd 中的目录、从 etcd 中获取插件列表、对于插件按照优先级进行排序等。我这里列出了关键部分的代码并进行讲解，当然，你可以在 GitHub 上看到更完整的[初始化函数](https://github.com/apache/apisix/blob/master/lua/apisix.lua#L47)。

```
function _M.http_init_worker()
-- 分别初始化路由、服务和插件这三个最重要的部分
    router.init_worker()
    require(&quot;apisix.http.service&quot;).init_worker()
    require(&quot;apisix.plugin&quot;).init_worker()
end

```

通过阅读这段代码，你可以发现，`router` 和 `plugin` 这两部分的初始化相对复杂一些，主要涉及到读取配置参数，并根据参数的不同做一些选择。因为这里会涉及到从 etcd 中读取数据，所以我们使用的是 `ngx.timer` 的方式，来绕过“不能在 `init_worker` 阶段使用 cosocket”的这个限制。如果你对这部分很感兴趣并且学有余力，建议一定要去读读源码，加深理解。

## 匹配路由

在最开始的 `access` 阶段里面，我们首先需要做的就是匹配路由，根据请求中携带 uri、host、args、cookie 等，来和已经设置好的路由规则进行匹配：

```
router.router_http.match(api_ctx)

```

对外暴露的，其实只有上面一行代码，这里的`api_ctx` 中存放的就是 uri、host、args、cookie 这些请求的信息。而具体的 `match` 函数的[实现](https://github.com/apache/apisix/blob/master/apisix/http/router/radixtree_uri.lua)，就用到了我们前面提到过的 `lua-resty-radixtree`。如果没有命中，就说明这个请求并没有设置与之对应的上游，就会直接返回 404。

```
local router = require(&quot;resty.radixtree&quot;)

local match_opts = {}

function _M.match(api_ctx)
    -- 从 ctx 中获取请求的参数，作为路由的判断条件
    match_opts.method = api_ctx.var.method
    match_opts.host = api_ctx.var.host
    match_opts.remote_addr = api_ctx.var.remote_addr
    match_opts.vars = api_ctx.var
    -- 调用路由的判断函数 
    local ok = uri_router:dispatch(api_ctx.var.uri, match_opts, api_ctx)
    -- 没有命中路由就直接返回 404 
    if not ok then
        core.log.info(&quot;not find any matched route&quot;)
        return core.response.exit(404)
    end

    return true
end

```

## 加载插件

当然，如果路由可以命中，就会走到过滤插件和加载插件的步骤，这也是 API 网关的核心所在。我们先来看下面这段代码：

```
local plugins = core.tablepool.fetch(&quot;plugins&quot;, 32, 0)
-- etcd 中的插件列表和本地配置文件中的插件列表进行交集运算 
api_ctx.plugins = plugin.filter(route, plugins)

-- 依次运行插件在 rewrite 和 access 阶段挂载的函数 
run_plugin(&quot;rewrite&quot;, plugins, api_ctx)
run_plugin(&quot;access&quot;, plugins, api_ctx)

```

在这段代码中，我们首先通过 table pool 的方式，申请了一个长度为 32 的 table，这是我们之前介绍过的性能优化技巧。然后便是插件的过滤函数。你可能疑惑，为什么需要这一步呢？在插件的 `init worker` 阶段，我们不是已经从 etcd 中获取插件列表并完成排序了吗？

事实上，这里的过滤是和本地配置文件来做对比的，主要有下面两个原因。

- 第一，新开发的插件需要灰度来发布，这时候新插件在 etcd 的列表中存在，但只在部分网关节点中处于开启状态。所以，我们需要额外做一次交集的运算。
- 第二，为了支持 debug 模式。终端的请求经过了哪些插件的处理？这些插件的加载顺序是什么？这些信息在调试的时候会很有用，所以在过滤函数中也会判断其是否处于 debug 模式，并在响应头中记录下这些信息。

因此，在 access 阶段的最后，我们会把这些过滤好的插件，按照优先级逐个运行，如下面这段代码所示：

```
local function run_plugin(phase, plugins, api_ctx)
    for i = 1, #plugins, 2 do
        local phase_fun = plugins[i][phase]
        if phase_fun then
            -- 最核心的调用代码 
            phase_fun(plugins[i + 1], api_ctx)
        end
    end

    return api_ctx
end

```

你可以看到，在遍历插件的时候，我们是以 `2` 为间隔进行的，这是因为每个插件都会有两个部分组成：插件对象和插件的配置参数。现在，我们来看上面示例代码中最核心的那一行代码：

```
phase_fun(plugins[i + 1], api_ctx)

```

单独看这行代码会有些抽象，我们用一个具体的 `limit_count` 插件来替换一下，就会清楚很多：

```
limit_count_plugin_rewrite_function(conf_of_plugin, api_ctx)

```

到这里，API 网关的整体流程，我们就实现得差不多了。这些代码都在同一个代码[文件](https://github.com/apache/apisix/blob/master/apisix/init.lua)中，它里面有 400 多行代码，但核心的代码就是我们上面所介绍的这短短几十行。

## 编写插件

现在，距离一个完整的 demo 还差一件事情，那就是编写一个插件，让它可以跑起来。我们以 `limit-count` 这个限制请求数的插件为例，它的[完整实现](https://github.com/apache/apisix/blob/master/apisix/plugins/limit-count.lua)只有 60 多行代码，你可以点击链接查看。下面，我来详细讲解下其中的关键代码。

首先，我们要引入 `lua-resty-limit-traffic` ，作为限制请求数的基础库：

```
local limit_count_new = require(&quot;resty.limit.count&quot;).new

```

然后，使用 rapidjson 中的 json schema ，来定义这个插件的参数有哪些：

```
local schema = {
    type = &quot;object&quot;,
    properties = {
        count = {type = &quot;integer&quot;, minimum = 0},
        time_window = {type = &quot;integer&quot;, minimum = 0},
        key = {type = &quot;string&quot;,
        enum = {&quot;remote_addr&quot;, &quot;server_addr&quot;},
        },
        rejected_code = {type = &quot;integer&quot;, minimum = 200, maximum = 600},
    },
    additionalProperties = false,
    required = {&quot;count&quot;, &quot;time_window&quot;, &quot;key&quot;, &quot;rejected_code&quot;},
}

```

插件的这些参数，和大部分 `resty.limit.count` 的参数是对应的，其中包含了限制的 key、时间窗口的大小、限制的请求数。另外，插件中增加了一个参数: `rejected_code`，在请求被限速的时候返回指定的状态码。

最后一步，我们把插件的处理函数挂载到 `rewrite` 阶段：

```
function _M.rewrite(conf, ctx)
    -- 从缓存中获取 limit count 的对象，如果没有就使用 `create_limit_obj` 函数新建并缓存 
    local lim, err = core.lrucache.plugin_ctx(plugin_name, ctx,  create_limit_obj, conf)

    -- 从 ctx.var 中获取 key 的值，并和配置类型和配置版本号一起组成新的 key 
    local key = (ctx.var[conf.key] or &quot;&quot;) .. ctx.conf_type .. ctx.conf_version

    --  进入限制的判断函数
    local delay, remaining = lim:incoming(key, true)
    if not delay then
        local err = remaining
        -- 如果超过阈值，就返回指定的状态码 
        if err == &quot;rejected&quot; then
            return conf.rejected_code
        end

        core.log.error(&quot;failed to limit req: &quot;, err)
        return 500
    end

    -- 如果没有超过阈值，就放行，并设置对应响应头 
    core.response.set_header(&quot;X-RateLimit-Limit&quot;, conf.count,
                             &quot;X-RateLimit-Remaining&quot;, remaining)
end

```

上面的代码中，进行限制判断的逻辑只有一行，其他的都是来做准备工作和设置响应头的。如果没有超过阈值，就会继续按照优先级运行下一个插件。

## 写在最后

今天这节课，通过整体框架和插件的编写，我们就完成了一个 API 网关的 Demo。更进一步，利用本专栏学到的 OpenResty 知识，你可以在上面继续添砖加瓦，搭建更丰富的功能。

最后，给你留一个思考题。我们知道，API 网关不仅可以处理七层的流量，也可以处理四层的流量，基于此，你能想到它的一些使用场景吗？欢迎留言说说你的看法，也欢迎你把这篇文章分享出去，和更多的人一起学习、交流。
