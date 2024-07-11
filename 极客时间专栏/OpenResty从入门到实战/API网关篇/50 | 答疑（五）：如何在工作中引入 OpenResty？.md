<audio id="audio" title="50 | 答疑（五）：如何在工作中引入 OpenResty？" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/03/94/03cb04deab8b531696b1a6e28408c694.mp3"></audio>

你好，我是温铭。

几个月的时间转瞬即逝，到现在，OpenResty专栏的最后一个版块微服务 API 网关篇，我们就已经学完了。恭喜你没有掉队，始终在积极学习和实践操作，并且热情地留下了你的思考。

很多留言提出的问题很有价值，大部分我都已经在App里回复过，一些手机上不方便回复的或者比较典型、有趣的问题，我专门摘了出来，作为今天的答疑内容，集中回复。另一方面，也是为了保证所有人都不漏掉任何一个重点。

下面我们来看今天的这 5 个问题。

## 问题一：OpenResty 在工作中的使用

Q：快结课了，我也基本上跟下来了，但自己的实践还是偏少（工作中目前未用）。不过，这确实是很强大的一门课。感谢温老师的持续分享，后期工作中我也会择机引入。

A：感谢这位同学的认可，关于这条留言，我想聊一聊，如何在工作中引入 OpenResty，这确实是一个值得一谈的话题。

OpenResty 基于 Nginx，并在它的基础之上加了 lua-nginx-module 的 C 模块和众多 lua-resty 库，所以 OpenResty 是可以无痛替换 Nginx 的，这是成本最低的开始使用 OpenResty 的方法。当然，这个替换过程也是有风险的，你需要注意下面这三点。

第一，确认线上 Nginx 的版本。OpenResty 的主版本号与 Nginx 保持一致，比如 OpenResty 1.15.8.1 使用的就是 Nginx 1.15.8 的内核。如果目前线上 Nginx 的版本号比 OpenResty 的最新版高，那么你最好谨慎换用 OpenResty，毕竟，OpenResty 升级的速度还是比较慢的，离 Nginx 的主线版本要落后半年到一年的时间。如果线上 Nginx 的版本和 OpenResty 的一致或者比 OpenResty 的低，那就具备了升级的前提条件。

第二，测试。测试是最主要的一个环节，使用 OpenResty 替换 Nginx 的风险很低，但肯定也存在一些风险。比如，是否有自定义的 C 模块需要编译，OpenResty 依赖的 openssl 版本，以及 OpenResty 给 Nginx 打的 patch 是否对业务会造成影响等。你需要复制一些业务的流量过来做验证。

第三，流量切换。基本的验证通过后，你还需要线上真实流量的灰度来验证，这时候为了能够快速的回滚，我们可以新开几台服务器来部署 OpenResty，而不是直接替换原有的 Nginx 服务。如果没有问题，我们可以选择二进制文件热升级的方式，或者是从 LB 中逐步摘掉和替换 Nginx 的方式来升级。

OpenResty 除了可以替代 Nginx 外，另外两个比较容易的切入点是 WAF 和 API 网关，它们都是对性能和动态有比较高要求的场景，也有对应的开源项目可以开箱即用，我在专栏中也有部分涉及到。

再继续把 OpenResty 深入到业务层面的话，就需要考虑比较多技术之外的因素了，比如是否容易招聘到 OpenResty 相关的工程师？是否能够和公司原有的技术系统进行融合等等。

总的来说，从替代 Nginx 的角度来切入，然后慢慢扩散来使用OpenResty ，是一个不错的注意。

## 问题二：OpenResty 的数据库封装

Q：根据你的指点，要尽量少用 `..`字符串拼接，特别是在代码热区。但是我在处理数据库访问时，需要动态构建SQL语句（在语句中插入变量），这应该是非常常见的使用场景。可是对于这个需求，我目前感觉，只有字符串拼接是最简单的办法，其他真的想不到既简单又高性能的办法。

A：你可以先用我们前面课程介绍过的 SystemTap 或者其他工具分析下，看 SQL 语句的拼接是否是系统的瓶颈。如果不是，自然就没有优化的必要性，毕竟，过早的优化是万恶之源。

如果瓶颈确实是  SQL 语句的拼接，那么我们可以利用数据库的 `prepare` 语句来做优化，也可以用数组的方式来做拼接。但 `lua-resrty-mysql` 对 `prepare` 的支持一直处于 TODO 状态，所以只剩下数组拼接的方式了。这也是一些 lua-resty 库的通病，实现了大部分的功能，处于能用的状态，但更新得并不够及时。除了数据库的 `prepare` 语句外，`lua-resty-redis` 对 `cluster` 也一直没有支持。

字符串拼接，包括 lua-resty 库的这类问题，OpenResty 是希望用 DSL 来彻底解决的——使用编译器的技术自动生成数组来拼接字符串，把这些细节隐藏起来，上层的用户不用感知；使用小语言 wirelang 来自动生成各种 lua-resty 网络通信库，不再需要手写。

这听上去很美好吧？但有一个问题必须正视，那就是自动生成的代码对人类是不友好的。如果你要学习或者修改生成的代码，就必须再学习编译器技术以及一门可能不会开源的 DSL，这会让参与社区的门槛越来越高。

## 问题三：OpenResty 的 Web 框架

Q：我现在想用 OpenResty 做一个Web项目，但做起来很痛苦，主要是没找到成熟的框架，需要自己造很多轮子，就比如说上面的数据库操作问题（没找到可以动态构建SQL语句、连贯操作的类库）。所以想问下老师，在Web框架上有什么好的可以推荐吗？

A：在 `awesome-resty` 这个仓库中，我们可以看到有专门的 [W](https://github.com/bungle/awesome-resty#web-frameworks)[eb 框架分类](https://github.com/bungle/awesome-resty#web-frameworks)，有 20 个 开源项目，不过大部分项目都处于停滞的状态。其中，Lapis、lor 和香草这三个项目你可以尝试下，看看哪一个更适合。

确实，由于没有强大的 Web 框架作为支撑，OpenResty 在处理大项目的时候就会力不从心，这也是很少有人用 OpenResty 做业务系统的原因之一。

## 问题四：修改了响应体，怎么修改响应头中的content-length？

Q：如果需要修改respones body的内容，就只能在body filter里做修改，但这样会引起body长度与 content-length 长度不一致，应该如何处理呢？

A：在这种情况下，我们需要在 body filter 之前的 header filter 阶段中，把 content length 这个响应头置为 nil，不再返回，改为流式输出。

下面是一段示例代码：

```
server {
    listen 8080;

    location /test {
            proxy_pass http://www.baidu.com;
            header_filter_by_lua_block {
                     ngx.header.content_length = nil
            }
            body_filter_by_lua_block {
                    ngx.arg[1] = ngx.arg[1] .. &quot;abc&quot;
            }
     }
}

```

通过这段代码你可以看到，在 body filter 阶段中，`ngx.arg[1]` 代表的就是响应体。如果我们在它后面增加了字符串 `abc`，响应头 content length 就不准确了，所以，我们在 header filter 阶段直接把它禁用掉就可以了。

另外，从这个示例中，我们还可以看到 OpenResty 的各个阶段之间是如何来配合工作的，这一点也希望你注意并思考。

## 问题五：Lua 代码的查找路径

Q：`lua_package_path` 似乎配置的是Lua依赖的搜索路径。对于`content_by_lua_file`，我试验发现，它只在prefix下根据指令提供的文件相对路径去搜索，而不会到 `lua_package_path` 下搜索。不知道我的理解对不对？

A：这位同学自己动手试验和思考的精神非常值得肯定，并且这个理解也是对的。`lua_package_path` 这个指令是用来加载 Lua 模块而使用的，比如我们在调用 `require 'cjson'` 时，就会到`lua_package_path` 中的指定目录中，去查找 cjson 这个模块。而 `content_by_lua_file` 则不同，它后面跟随的是磁盘中的一个文件路径：

```
location /test {
     content_by_lua_file /path/test.lua;
 }

```

而且，如果这里不是绝对路径而是相对路径：

```
 content_by_lua_file path/test.lua;

```

那么就会使用 OpenResty 启动时指定的 `-p` 目录，来做一个拼接，从而得到绝对路径。

今天主要解答这几个问题。最后，欢迎你继续在留言区写下你的疑问，我会持续不断地解答。希望可以通过交流和答疑，帮你把所学转化为所得。也欢迎你把这篇文章转发出去，我们一起交流、一起进步。


