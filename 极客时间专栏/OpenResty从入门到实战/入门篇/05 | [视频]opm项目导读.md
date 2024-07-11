
<video poster="https://static001.geekbang.org/resource/image/b8/a2/b8e479499551550984792f338043a8a2.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/32bb5df8-16d13f123cf-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/71f11392644a4bd19a04901804f3faa2/b54cb30cb2b648d1a3e6392ecc351626-e6d62449b88ad07255d04f874d679181-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/71f11392644a4bd19a04901804f3faa2/b54cb30cb2b648d1a3e6392ecc351626-32e46e96bebe336fa79a9419934ca38a-hd.m3u8" type="application/x-mpegURL"></video>

你好，我是温铭。

今天的内容，我特意安排成了视频的形式来讲解。不过，在你看视频之前，我想先问你这么几个问题：

- 在真实的项目中，你会配置 nginx.conf，以便和 Lua 代码联动吗？
- 你清楚 OpenResty 的代码结构该如何组织吗？

这两个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。

同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。

## 今日核心

[opm](https://github.com/openresty/opm/) 是 OpenResty 中为数不多的网站类项目，而里面的代码，基本上是由 OpenResty 的作者亲自操刀完成的。

很多 OpenResty 的使用者并不清楚，如何在真实的项目中去配置 nginx.conf， 以及如何组织 Lua 的代码结构。确实，在这方面可以参考的开源项目并不多，给学习使用带了不小的阻力。

不过，借助今天的这个项目，你就可以克服这一点了。你将会熟悉一个OpenResty 项目的结构和开发流程，还能看到 OpenResty 的作者是如何编写业务类 Lua 代码的。

opm 还涉及到数据库的操作，它后台数据的储存，使用的是PostgreSQL ，你可以顺便了解下 OpenResty 和数据库是如何交互的。

除此之外，这个项目还涉及到一些简单的性能优化，也是为了后面专门设立的性能优化内容做个铺垫。

最后，浏览完 opm 这个项目后，你可以自行看下另外一个类似的项目，那就是 OpenResty 的官方网站：[https://github.com/openresty/openresty.org](https://github.com/openresty/openresty.org)。

## 课件参考

今天的课件已经上传到了我的GitHub上，你可以自己下载学习。

链接如下：[https://github.com/iresty/geektime-slides](https://github.com/iresty/geektime-slides)

如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。
