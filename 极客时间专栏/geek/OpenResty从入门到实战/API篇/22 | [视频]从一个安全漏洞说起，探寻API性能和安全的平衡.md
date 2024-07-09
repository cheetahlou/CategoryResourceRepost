
<video poster="https://static001.geekbang.org/resource/image/2b/04/2b19372f8c88bb89c799382bb4767504.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/55ac88a8-16ce81d8277-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/e63152456c494c38a25ef45b274c9610/f2a99770955f4dff8f68622573c395ca-37963b86cd6511e18c03ec2809815ddb-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/e63152456c494c38a25ef45b274c9610/f2a99770955f4dff8f68622573c395ca-fbd7fdcd1c8e947dad21fe0f44731b04-hd.m3u8" type="application/x-mpegURL"></video>

你好，我是温铭。

今天的内容，我同样会以视频的形式来讲解。老规矩，在你进行视频学习之前，我想先问你这么几个问题：

- 你在使用 OpenResty 的时候，是否注意到有 API 存在安全隐患呢？
- 在安全和性能之间，如何去平衡它们的关系呢？

这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。

同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。

## 今日核心

安全，是一个永恒的话题，不管你是写开发业务代码，还是做底层的架构，都离不开安全方面的考虑。

CVE-2018-9230 是与 OpenResty 相关的一个安全漏洞，但它并非 OpenResty 自身的安全漏洞。这听起来是不是有些拗口呢？没关系，接下来让我们具体看下，攻击者是如何构造请求的。

OpenResty 中的 `ngx.req.get_uri_args`、`ngx.req.get_post_args` 和 `ngx.req.get_headers`接口，默认只返回前 100 个参数。如果 WAF 的开发者没有注意到这个细节，就会被参数溢出的方式攻击。攻击者可以填入 100 个无用参数，把 payload 放在第 101 个参数中，借此绕过 WAF 的检测。

那么，应该如何处理这个 CVE 呢？

显然，OpenResty 的维护者需要考虑到向下兼容、不引入更多安全风险和不影响性能这么几个因素，并要在其中做出一个平衡的选择。

最终，OpenResty 维护者选择新增一个 err 的返回值来解决这个问题。如果输入参数超过 100 个，err 的提示信息就是 truncated。这样一来，这些 API 的调用者就必须要处理错误信息，自行判断拒绝请求还是放行。

其实，归根到底，安全是一种平衡。究竟是选择基于规则的黑名单方式，还是选择基于身份的白名单方式，抑或是两种方式兼用，都取决于你的实际业务场景。

## 课件参考

今天的课件已经上传到了我的GitHub上，你可以自己下载学习。

链接如下：[https://github.com/iresty/geektime-slides](https://github.com/iresty/geektime-slides)

如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。
