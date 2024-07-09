<audio id="audio" title="09 | 为什么 lua-resty-core 性能更高一些？" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/9d/03/9df43767fc424ac63ecff81a69748703.mp3"></audio>

你好，我是温铭。

前面两节课我们说了，Lua 是一种嵌入式开发语言，核心保持了短小精悍，你可以在 Redis、NGINX 中嵌入 Lua，来帮助你更灵活地完成业务逻辑。同时，Lua 也可以调用已有的 C 函数和数据结构，避免重复造轮子。

在 Lua 中，你可以用 Lua C API 来调用 C 函数，而在 LuaJIT 中还可以使用 FFI。对 OpenResty 而言：

- 在核心的 `lua-nginx-module` 中，调用 C 函数的 API，都是使用 Lua C API 来完成的；
- 而在 `lua-resty-core` 中，则是把 `lua-nginx-module` 已有的部分 API，使用 FFI 的模式重新实现了一遍。

看到这里你估计纳闷了：为什么要用 FFI 重新实现一遍？

别着急，让我们以 [ngx.base64_decode](https://github.com/openresty/lua-nginx-module#ngxdecode_base64) 这个很简单的 API 为例，一起看下 Lua C API 和 FFI 的实现有何不同之处，这样你也可以对它们的性能有个直观的认识。

## Lua CFunction

我们先来看下， `lua-nginx-module` 中用 Lua C API 是如何实现的。我们在项目的代码中搜索 `decode_base64`，可以找到它的代码实现在 `ngx_http_lua_string.c` 中：

```
lua_pushcfunction(L, ngx_http_lua_ngx_decode_base64);
lua_setfield(L, -2, &quot;decode_base64&quot;);

```

上面的代码看着就头大，不过还好，我们不用深究那两个 `lua_` 开头的函数，以及它们参数的具体作用，只需要知道一点——这里注册了一个 CFunction：`ngx_http_lua_ngx_decode_base64`， 而它与 `ngx.base64_decode` 这个对外暴露的 API 是对应关系。

我们继续“按图索骥”，在这个 C 文件中搜索 `ngx_http_lua_ngx_decode_base64`，它定义在文件的开始位置：

```
static int ngx_http_lua_ngx_decode_base64(lua_State *L);

```

对于那些能够被 Lua 调用的 C 函数来说，它的接口必须遵循 Lua 要求的形式，也就是 `typedef int (*lua_CFunction)(lua_State* L)`。它包含的参数是 `lua_State` 类型的指针 L ；它的返回值类型是一个整型，表示返回值的数量，而非返回值自身。

它的实现如下（这里我已经去掉了错误处理的代码）：

```
static int
 ngx_http_lua_ngx_decode_base64(lua_State *L)
 {
     ngx_str_t p, src;
 
    src.data = (u_char *) luaL_checklstring(L, 1, &amp;src.len);
 
     p.len = ngx_base64_decoded_length(src.len);
 
     p.data = lua_newuserdata(L, p.len);
 
     if (ngx_decode_base64(&amp;p, &amp;src) == NGX_OK) {
         lua_pushlstring(L, (char *) p.data, p.len);
 
     } else {
         lua_pushnil(L);
     }
 
     return 1;
 }

```

这段代码中，最主要的是 `ngx_base64_decoded_length` 和 `ngx_decode_base64`， 它们都是 NGINX 自身提供的 C 函数。

我们知道，用 C 编写的函数，无法把返回值传给 Lua 代码，而是需要通过栈，来传递 Lua 和 C 之间的调用参数和返回值。这也是为什么，会有很多我们一眼无法看懂的代码。同时，这些代码也不能被 JIT 跟踪到，所以对于 LuaJIT 而言，这些操作是处于黑盒中的，没法进行优化。

## LuaJIT FFI

而 FFI 则不同。FFI 的交互部分是用 Lua 实现的，这部分代码可以被 JIT 跟踪到，并进行优化；当然，代码也会更加简洁易懂。

我们还是以 `base64_decode`为例，它的 FFI 实现分散在两个仓库中： `lua-resty-core` 和 `lua-nginx-module`。我们先来看下前者里面[实现的代码](https://github.com/openresty/lua-resty-core/blob/master/lib/resty/core/base64.lua#L72)：

```
ngx.decode_base64 = function (s)
     local slen = #s
     local dlen = base64_decoded_length(slen)
     
     local dst = get_string_buf(dlen)
     local pdlen = get_size_ptr()
     local ok = C.ngx_http_lua_ffi_decode_base64(s, slen, dst, pdlen)
     if ok == 0 then
         return nil
     end
     return ffi_string(dst, pdlen[0])
 end

```

你会发现，相比 CFunction，FFI 实现的代码清爽了很多，它具体的实现是 `lua-nginx-module` 仓库中的`ngx_http_lua_ffi_decode_base64`，如果你对这里感兴趣，可以自己去查看这个函数的实现，特别简单，这里我就不贴代码了。

不过，细心的你，是否从上面的代码片段中，发现函数命名的一些规律了呢？

没错，OpenResty 中的函数都是有命名规范的，你可以通过命名推测出它的用处。比如：

- `ngx_http_lua_ffi_` ，是用 FFI 来处理 NGINX HTTP 请求的 Lua 函数；
- `ngx_http_lua_ngx_` ，是用 Cfunction 来处理 NGINX HTTP 请求的 Lua 函数；
- 其他 `ngx_` 和 `lua_` 开头的函数，则分别属于 NGINX 和 Lua 的内置函数。

更进一步，OpenResty 中的 C 代码，也有着严格的代码规范，这里我推荐阅读[官方的 C 代码风格指南](https://openresty.org/cn/c-coding-style-guide.html)。对于有意学习 OpenResty 的 C 代码并提交 PR 的开发者来说，这是必备的一篇文档。否则，即使你的 PR 写得再好，也会因为代码风格问题被反复评论并要求修改。

关于 FFI 更多的 API 和细节，推荐你阅读 LuaJIT [官方的教程](http://luajit.org/ext_ffi_tutorial.html) 和 [文档](http://luajit.org/ext_ffi_api.html)。技术专栏并不能代替官方文档，我也只能在有限的时间内帮你指出学习的路径，少走一些弯路，硬骨头还是需要你自己去啃的。

## LuaJIT FFI GC

使用 FFI 的时候，我们可能会迷惑：在 FFI 中申请的内存，到底由谁来管理呢？是应该我们在 C 里面手动释放，还是 LuaJIT 自动回收呢？

这里有个简单的原则：LuaJIT 只负责由自己分配的资源；而 `ffi.C` 是 C 库的命名空间，所以，使用 `ffi.C` 分配的空间不由 LuaJIT 负责，需要你自己手动释放。

举个例子，比如你使用 `ffi.C.malloc` 申请了一块内存，那你就需要用配对的 `ffi.C.free` 来释放。LuaJIT 的官方文档中有一个对应的示例：

```
local p = ffi.gc(ffi.C.malloc(n), ffi.C.free)
 ...
 p = nil -- Last reference to p is gone.
 -- GC will eventually run finalizer: ffi.C.free(p)

```

这段代码中，`ffi.C.malloc(n)` 申请了一段内存，同时 `ffi.gc` 就给它注册了一个析构的回调函数 `ffi.C.free`。这样一来，`p` 这个 `cdata` 在被 LuaJIT GC 的时候，就会自动调用 `ffi.C.free`，来释放 C 级别的内存。而 `cdata` 是由 LuaJIT 负责 GC的 ，所以上述代码中的 `p` 会被 LuaJIT 自动释放。

这里要注意，如果你要在 OpenResty 中申请大块的内存，我更推荐你用 `ffi.C.malloc` 而不是 `ffi.new`。原因也很明显：

1. `ffi.new` 返回的是一个 `cdata`，这部分内存由 LuaJIT 管理；
1. LuaJIT GC 的管理内存是有上限的，OpenResty 中的 LuaJIT 并未开启 GC64 选项，所以**单个 worker 内存的上限只有2G**。一旦超过 LuaJIT 的内存管理上限，就会导致报错。

**在使用 FFI 的时候，我们还需要特别注意内存泄漏的问题**。不过，凡人皆会犯错，只要是人写的代码，百密一疏，总会出现 bug。那么，有没有什么工具可以检测内存泄漏呢？

这时候，OpenResty 强大的周边测试和调试工具链就派上用场了。

我们先来说说测试。在 OpenResty 体系中，我们使用 Valgrind 来检测内存泄漏问题。

前面课程我们提到过的测试框架 `test::nginx`，有专门的内存泄漏检测模式去运行单元测试案例集，你只需要设置环境变量 `TEST_NGINX_USE_VALGRIND=1` 即可。OpenResty 的官方项目在发版本之前，都会在这个模式下完整回归，后面的测试章节中我们再详细介绍。

而 OpenResty 的 CLI `resty` 也有 `--valgrind` 选项，方便你单独运行某段 Lua 代码，即使你没有写测试案例也是没问题的。

再来看调试工具。

OpenResty 提供[基于 systemtap 的扩展](https://github.com/openresty/stapxx)，来对 OpenResty 程序进行活体的动态分析。你可以在这个项目的工具集中，搜索 `gc` 这个关键字，会看到 `lj-gc` 和 `lj-gc-objs` 这两个工具。

而对于 core dump 这种离线分析，OpenResty 提供了 [GDB 的工具集](https://github.com/openresty/openresty-gdb-utils)，同样你可以在里面搜索 `gc`，找到 `lgc`、`lgcstat` 和 `lgcpath` 三个工具。

这些调试工具的具体用法，我们会在后面的调试章节中详细介绍，你先有个印象即可。这样，你遇到内存问题就不会“病急乱投医“，毕竟OpenResty 有专门的工具集，帮你定位和解决这些问题。

## lua-resty-core

从上面的比较中，我们可以看到，FFI 的方式不仅代码更简洁，而且可以被 LuaJIT 优化，显然是更优的选择。其实现实也是如此，实际上，CFunction 的实现方式已经被 OpenResty 废弃，相关的实现也从代码库中移除了。现在新的 API，都通过 FFI 的方式，在 `lua-resty-core` 仓库中实现。

在 OpenResty 2019 年 5 月份发布的 1.15.8.1 版本前，`lua-resty-core` 默认是不开启的，而这不仅会带来性能损失，更严重的是会造成潜在的 bug。所以，我强烈推荐还在使用历史版本的用户，都手动开启 `lua-resty-core`。你只需要在 `init_by_lua` 阶段，增加一行代码就可以了：

```
require &quot;resty.core&quot;

```

当然，姗姗来迟的 1.15.8.1 版本中，已经增加了 `lua_load_resty_core` 指令，默认开启了 `lua-resty-core`。我个人感觉，OpenResty 对于 `lua-resty-core` 的开启还是过于谨慎了，开源项目应该尽早把类似的功能设置为默认开启。

`lua-resty-core` 中不仅重新实现了部分 lua-nginx-module 项目中的 API，比如 `ngx.re.match`、`ngx.md5` 等，还实现了不少新的 API，比如 ngx.ssl、ngx.base64、ngx.errlog、ngx.process、ngx.re.split、ngx.resp.add_header、ngx.balancer、ngx.semaphore 等等，我们在后面的 OpenResty API 章节中会介绍到。

## 写在最后

讲了这么多内容，最后我还是想说，FFI 虽然好，却也并不是性能银弹。它之所以高效，主要原因就是可以被 JIT 追踪并优化。如果你写的 Lua 代码不能被 JIT，而是需要在解释模式下执行，那么 FFI 的效率反而会更低。

那么到底有哪些操作可以被 JIT，哪些不能呢？怎样才可以避免写出不能被 JIT 的代码呢？下一节我来揭晓这个问题。

最后，给你留一个需要动手的作业题：你可以找一两个lua-nginx-module 和 lua-resty-core 中都存在的 API，然后性能测试比较一下两者的差异吗？你可以看下 FFI 的性能提升到底有多大。

欢迎留言和我分享你的思考、收获，也欢迎你把这篇文章分享给你的同事、朋友，一起交流，一起进步。


