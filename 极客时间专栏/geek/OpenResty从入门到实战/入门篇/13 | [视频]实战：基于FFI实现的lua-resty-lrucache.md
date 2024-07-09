
<video poster="https://static001.geekbang.org/resource/image/6a/f7/6ada085b44eddf37506b25ad188541f7.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/30d99c0d-16d14089303-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/2ce11b32e3e740ff9580185d8c972303/a01ad13390fe4afe8856df5fb5d284a2-f2f547049c69fa0d4502ab36d42ea2fa-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/2ce11b32e3e740ff9580185d8c972303/a01ad13390fe4afe8856df5fb5d284a2-2528b0077e78173fd8892de4d7b8c96d-hd.m3u8" type="application/x-mpegURL"></video>

你好，我是温铭。

今天的内容，我同样会以视频的形式来讲解。不过，在你进行视频学习之前，我想先问你这么几个问题：

- lua-resty-lrucache 内部最重要的数据结构是什么？
- lua-resty-lrucache 有两种 FFI 的实现，我们今天讲的这一种更适合什么场景？

这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。

同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。

## 今日核心

[lua-resty-lrucache](https://github.com/openresty/lua-resty-lrucache) 是一个使用 LuaJIT FFI 实现的 LRU 缓存库，可以在 worker 内缓存各种类型的数据。功能与之类似的是 shared dict，但 shared dict 只能存储字符串类型的数据。在大多数实际情况下，这两种缓存是配合在一起使用的——lrucache 作为一级缓存，shared dict 作为二级缓存。

lrucache 的实现，并没有涉及到 OpenResty 的 Lua API。所以，即使你以前没有用过OpenResty，也可以通过这个项目来学习如何使用 LuaJIT 的 FFI。

lrucache 仓库中包含了两种实现方案，一种是使用 Lua table 来实现缓存，另外一种则是使用 hash 表来实现。前者更适合命中率高的情况，后者适合命中率低的情况。两个方案没有哪个更好，要看你的线上环境更适合哪一个。

通过今天这个项目，你可以弄清楚要如何使用 FFI，并了解一个完整的 lua-resty 库应该包括哪些必要的内容。当然，我顺道也会介绍下 travis 的使用。

最后，还是想强调一点，在你面对一个陌生的开源项目时，文档和测试案例永远是最好的上手方式。而你后期如果要阅读源码，也不要先去抠细节，而是应该先去看主要的数据结构，围绕重点逐层深入。

## 课件参考

今天的课件已经上传到了我的GitHub上，你可以自己下载学习。

链接如下：[https://github.com/iresty/geektime-slides](https://github.com/iresty/geektime-slides)

如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。
