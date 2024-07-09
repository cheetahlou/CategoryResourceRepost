
<video poster="https://static001.geekbang.org/resource/image/65/c2/6565cf0a87645948ef66c547192db3c2.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/3d0d7df0-16ce81ba96a-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/ed7db4a5658b4ed59f225967841c31f8/e7f659c5c1864bb3bde42a4c200eab4b-d5090411129213bc3a3a2d388e0daa0b-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/ed7db4a5658b4ed59f225967841c31f8/e7f659c5c1864bb3bde42a4c200eab4b-3d9bf901ccefc4abb20c7469df8134d4-hd.m3u8" type="application/x-mpegURL"></video>

你好，我是温铭。

今天的内容，我同样会以视频的形式来讲解。老规矩，在你进行视频学习之前，先问你这么几个问题：

- 面对多个相同功能的 lua-resty 库，我们应该从哪些方面来选择？
- 如何来组织一个 lua-resty 的结构？

这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。

同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。

## 今日核心

前面我们介绍过的 lua-resty 库都是官方自带的，但在 HTTP client 这个最常用的库上，官方并没有。这时候，我们就得自己来选择一个优秀的第三方库了。

那么，如何在众多的 lua-resty HTTP client 中，选择一个最好、最适合自己的第三方库呢？

这时候，你就需要综合考虑活跃度、作者、测试覆盖度、接口封装等各方面的因素了。我最后选择的是 lua-resty-requests（[https://github.com/tokers/lua-resty-requests](https://github.com/tokers/lua-resty-requests)），它是由又拍云的工程师 tokers 贡献的，我个人很喜欢它的接口风格，也推荐给你。

在视频中我会从最简单的 get 接口入手，结合文档、测试案例和源码，来逐步展开。你可以看到一个优秀的 lua-resty 库是如何编写的，有哪些可以借鉴的地方。

## 课件参考

今天的课件已经上传到了我的GitHub上，你可以自己下载学习。

链接如下：[https://github.com/iresty/geektime-slides](https://github.com/iresty/geektime-slides)

如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。
