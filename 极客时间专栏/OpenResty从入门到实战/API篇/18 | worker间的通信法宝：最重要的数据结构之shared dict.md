<audio id="audio" title="18 | worker间的通信法宝：最重要的数据结构之shared dict" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/b7/a8/b7a770648c696dd5b49fac1a887615a8.mp3"></audio>

你好，我是温铭。

前面我们讲过，在 Lua 中， table 是唯一的数据结构。与之对应的一个事实是，共享内存字典shared dict，是你在 OpenResty 编程中最为重要的数据结构。它不仅支持数据的存放和读取，还支持原子计数和队列操作。

基于 shared dict，你可以实现多个 worker 之间的缓存和通信，以及限流限速、流量统计等功能。你可以把 shared dict 当作简单的 Redis 来使用，只不过 shared dict 中的数据不能持久化，所以你存放在其中的数据，一定要考虑到丢失的情况。

## 数据共享的几种方式

在编写 OpenResty Lua 代码的过程中，你不可避免地会遇到，在一个请求的不同阶段、不同 worker 之间共享数据的情况，还可能需要在 Lua 和 C 代码之间共享数据。

所以，在正式介绍 shared dict 的 API 之前，先让我们了解一下，OpenResty 中常见的几种数据共享的方法；并学会根据实际情况，选择较为合适的数据共享方式。

**第一种是 Nginx 中的变量**。它可以在 Nginx C 模块之间共享数据，自然的，也可以在 C 模块和 OpenResty 提供的 `lua-nginx-module` 之间共享数据，比如下面这段代码：

```
location /foo {
     set $my_var ''; # this line is required to create $my_var at config time
     content_by_lua_block {
         ngx.var.my_var = 123;
         ...
     }
 }

```

不过，使用 Nginx 变量这种方式来共享数据是比较慢的，因为它涉及到 hash 查找和内存分配。同时，这种方法有其局限性，只能用来存储字符串，不能支持复杂的 Lua 类型。

**第二种是`ngx.ctx`，可以在同一个请求的不同阶段之间共享数据**。它其实就是一个普通的 Lua 的 table，所以速度很快，还可以存储各种 Lua 的对象。它的生命周期是请求级别的，当一个请求结束的时候，`ngx.ctx` 也会跟着被销毁掉。

下面是一个典型的使用场景，我们用 `ngx.ctx` 来缓存 `Nginx 变量` 这种昂贵的调用，并在不同阶段都可以使用到它：

```
location /test {
     rewrite_by_lua_block {
         ngx.ctx.host = ngx.var.host
     }
     access_by_lua_block {
        if (ngx.ctx.host == 'openresty.org') then
            ngx.ctx.host = 'test.com'
        end
     }
     content_by_lua_block {
         ngx.say(ngx.ctx.host)
     }
 }

```

这时，如果你使用 curl 访问的话：

```
curl -i 127.0.0.1:8080/test -H 'host:openresty.org'

```

就会打印出 `test.com`，可以表明 `ngx.ctx` 的确是在不同阶段共享了数据。当然，你还可以自己动手修改上面的例子，保存 table 等更复杂的对象，而非简单的字符串，看看它是否满足你的预期。

不过，这里需要特别注意的是，正因为 `ngx.ctx` 的生命周期是请求级别的，所以它并不能在模块级别进行缓存。比如，我在 `foo.lua` 文件中这样使用就是错误的：

```
local ngx_ctx = ngx.ctx

local function bar()
    ngx_ctx.host =  'test.com'
end

```

我们应该在函数级别进行调用和缓存：

```
local ngx = ngx

local function bar()
    ngx_ctx.host =  'test.com'
end

```

`ngx.ctx` 还有很多的细节，后面的性能优化部分，我们再继续探讨。

接着往下看，**第三种方法是使用`模块级别的变量`，在同一个 worker 内的所有请求之间共享数据**。跟前面的 Nginx 变量和 `ngx.ctx` 不一样，这种方法有些不太好理解。不过别着急，概念抽象，代码先行，让我们先来看个例子，弄明白什么是 `模块级别的变量`：

```
-- mydata.lua
 local _M = {}

 local data = {
     dog = 3,
     cat = 4,
     pig = 5,
 }

 function _M.get_age(name)
     return data[name]
 end

 return _M

```

在 nginx.conf 的配置如下：

```
location /lua {
     content_by_lua_block {
         local mydata = require &quot;mydata&quot;
         ngx.say(mydata.get_age(&quot;dog&quot;))
     }
 }

```

在这个示例中，`mydata` 就是一个模块，它只会被 worker 进程加载一次，之后，这个 worker 处理的所有请求，都会共享 `mydata` 模块的代码和数据。

自然，`mydata` 模块中的 `data` 这个变量，就是 `模块级别的变量`，它位于模块的 top level，也就是模块最开始的位置，所有函数都可以访问到它。

所以，你可以把需要在请求间共享的数据，放在模块的 top level 变量中。不过，需要特别注意的是，一般我们只用这种方式来保存**只读的数据**。如果涉及到写操作，你就要非常小心了，因为可能会有 **race condition**，这是**非常难以定位的 bug**。

我们可以通过下面这个最简化的例子来体会下：

```
-- mydata.lua
 local _M = {}

 local data = {
     dog = 3,
     cat = 4,
     pig = 5,
 }

 function _M.incr_age(name)
     data[name]  = data[name] + 1
    return data[name]
 end

 return _M

```

在模块中，我们增加了 `incr_age` 这个函数，它会对 data 这个表的数据进行修改。

然后，在调用的代码中，我们增加了最关键的一行 `ngx.sleep(5)`，这个 sleep 是一个 yield 操作：

```
location /lua {
     content_by_lua_block {
         local mydata = require &quot;mydata&quot;
         ngx.say(mydata. incr_age(&quot;dog&quot;))
         ngx.sleep(5) -- yield API
         ngx.say(mydata. incr_age(&quot;dog&quot;))
     }
 }

```

如果没有这行 sleep 代码（也可以是其他的非阻塞 IO 操作，比如访问 Redis 等），就不会有 yield 操作，也就不会产生竞争，那么，最后输出的数字就是顺序的。

但当我们加了这行代码后，哪怕只是在 sleep 的 5 秒钟内，也很可能就有其他请求调用了`mydata. incr_age` 函数，修改了变量的值，从而导致最后输出的数字不连续。要知道，在实际的代码中，逻辑不会这么简单，bug 的定位也一定会困难得多。

所以，除非你很确定这中间没有 yield 操作，不会把控制权交给 Nginx 事件循环，否则，我建议你还是保持对模块级别变量的只读。

**第四种，也是最后一种方法，用 shared dict 来共享数据，这些数据可以在多个 worker 之间共享。**

这种方法是基于红黑树实现的，性能很好，但也有自己的局限性——你必须事先在 Nginx 的配置文件中，声明共享内存的大小，并且这不能在运行期更改：

```
lua_shared_dict dogs 10m;

```

shared dict 同样只能缓存字符串类型的数据，不支持复杂的 Lua 数据类型。这也就意味着，当我需要存放 table 等复杂的数据类型时，我将不得不使用 json 或者其他的方法，来序列化和反序列化，这自然会带来不小的性能损耗。

总之，还是那句话，这里并没有银弹，不存在一种完美的数据共享方式，你需要根据需求和场景，来组合多个方法来使用。

## 共享字典

上面数据共享的部分，我们花了很多的篇幅来学，有的人可能纳闷儿：它们看上去和 shared dict 没有直接关系，是不是有些文不对题呢？

事实并非如此，你可以自己想一下，为什么 OpenResty 中要有 shared dict 的存在呢？

回忆一下刚刚讲的几种方法，前面三种数据共享的范围都是在请求级别，或者单个 worker 级别。所以，在当前的 OpenResty 的实现中，只有 shared dict 可以完成 worker 间的数据共享，并借此实现 worker 之间的通信，这也是它存在的价值。

在我看来，明白一个技术为何存在，并弄清楚它和别的类似技术之间的差异和优势，远比你只会熟练调用它提供的 API 更为重要。这种技术视野，会给你带来一定程度的远见和洞察力，这也可以说是工程师和架构师的一个重要区别。

回到共享字典本身，它对外提供了 20多个 Lua API，不过所有的这些 API 都是原子操作，你不用担心多个 worker 和高并发的情况下的竞争问题。

这些 API 都有官方详细的[文档](https://github.com/openresty/lua-nginx-module#ngxshareddict)，我就不再一一赘述了。这里我想再强调一下，任何技术课程的学习，都不能代替对官方文档的仔细研读。这些耗时的笨功夫，每个人都省不掉的。

继续看shared dict 的 API，这些 API可以分为下面三个大类，也就是字典读写类、队列操作类和管理类这三种。

### 字典读写类

首先来看字典读写类。在最初的版本中，只有字典读写类的 API，它们也是共享字典最常用的功能。下面是一个最简单的示例：

```
$ resty --shdict='dogs 1m' -e 'local dict = ngx.shared.dogs
                               dict:set(&quot;Tom&quot;, 56)
                               print(dict:get(&quot;Tom&quot;))'

```

除了 set 外，OpenResty 还提供了 `safe_set`、`add`、`safe_add`、`replace` 这四种写入的方法。这里`safe` 前缀的含义是，在内存占满的情况下，不根据 LRU 淘汰旧的数据，而是写入失败并返回 `no memory` 的错误信息。

除了 get 外，OpenResty 还提供了 `get_stale` 的读取数据的方法，相比 `get` 方法，它多了一个过期数据的返回值：

```
value, flags, stale = ngx.shared.DICT:get_stale(key)

```

你还可以调用 `delete` 方法来删除指定的 key，它和 `set(key, nil)` 是等价的。

### 队列操作类

再来看队列操作，它是 OpenResty 后续新增的功能，提供了和 Redis 类似的接口。队列中的每一个元素，都用 `ngx_http_lua_shdict_list_node_t` 来描述：

```
typedef struct { 
    ngx_queue_t queue; 
    uint32_t value_len; 
    uint8_t value_type; 
    u_char data[1]; 
} ngx_http_lua_shdict_list_node_t;

```

我把这些队列操作 API 的 [PR](https://github.com/openresty/lua-nginx-module/pull/586/files)  贴在了文章中，如果你对此感兴趣，可以跟着文档、测试案例和源码，来分析具体的实现。

不过，下面这 5 个队列 API，在文档中并没有对应的代码示例，这里我简单介绍一下：

- lpush/rpush，表示在队列两端增加元素；
- lpop/rpop，表示在队列两端弹出元素；
- llen，表示返回队列的元素数量。

别忘了我们上节课讲过的另一个利器——测试案例。如果文档中没有，我们通常可以在测试案例中找到对应的代码。队列相关的测试，正是在 `145-shdict-list.t` 这个文件中：

```
=== TEST 1: lpush &amp; lpop
--- http_config
    lua_shared_dict dogs 1m;
--- config
    location = /test {
        content_by_lua_block {
            local dogs = ngx.shared.dogs

            local len, err = dogs:lpush(&quot;foo&quot;, &quot;bar&quot;)
            if len then
                ngx.say(&quot;push success&quot;)
            else
                ngx.say(&quot;push err: &quot;, err)
            end

            local val, err = dogs:llen(&quot;foo&quot;)
            ngx.say(val, &quot; &quot;, err)

            local val, err = dogs:lpop(&quot;foo&quot;)
            ngx.say(val, &quot; &quot;, err)

            local val, err = dogs:llen(&quot;foo&quot;)
            ngx.say(val, &quot; &quot;, err)

            local val, err = dogs:lpop(&quot;foo&quot;)
            ngx.say(val, &quot; &quot;, err)
        }
    }
--- request
GET /test
--- response_body
push success
1 nil
bar nil
0 nil
nil nil
--- no_error_log
[error]

```

### 管理类

最后要说的管理类 API 也是后续新增的，属于社区呼声比较高的需求。其中，共享内存的使用情况就是最典型的例子。比如，用户申请了 100M 的空间作为 shared dict，那么这 100M 是否够用呢？里面存放了多少 key？具体是哪些 key 呢？这几个都是非常现实的问题。

对于这类问题，OpenResty 的官方态度，是希望用户使用火焰图来解决，即非侵入式，保持代码基的高效和整洁，而不是提供侵入式的 API 来直接返回结果。

但站在使用者友好角度来考虑，这些管理类 API 还是非常有必要的。毕竟开源项目是用来解决产品需求的，并不是展示技术本身的。所以，下面我们就来了解一下，这几个后续增加的管理类 API。

首先是 `get_keys(max_count?)`，它默认也只返回前 1024 个 key；如果你把 `max_count` 设置为 0，那就返回所有 key。

然后是 `capacity` 和 `free_space`，这两个 API 都属于 lua-resty-core 仓库，所以需要你 require 后才能使用：

```
require &quot;resty.core.shdict&quot;

 local cats = ngx.shared.cats
 local capacity_bytes = cats:capacity()
 local free_page_bytes = cats:free_space()

```

它们分别返回的，是共享内存的大小（也就是 `lua_shared_dict` 中配置的大小）和空闲页的字节数。因为 shared dict 是按照页来分配的，即使 `free_space` 返回为 0，在已经分配的页面中也可能存在空间，所以它的返回值并不能代表共享内存实际被占用的情况。

## 写在最后

在实际的开发中，我们经常会用到多级缓存，OpenResty 的官方项目中也有对缓存的封装。你能找出来是哪几个项目吗？或者你知道一些其他缓存封装的 lua-resty 库吗？

欢迎留言和我分享，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。


