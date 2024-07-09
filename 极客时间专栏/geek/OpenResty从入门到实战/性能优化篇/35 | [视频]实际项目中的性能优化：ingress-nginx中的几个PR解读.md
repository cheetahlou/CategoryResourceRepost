
<video poster="https://static001.geekbang.org/resource/image/53/80/536c067253bc7d68cfbb54f762484980.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/1a97c39b-16ce823e7a2-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/402947d480674f578b44c42e194ef714/8e5079a430654fe4a9901fad3e5a9a3a-72367b8ed74bf04d395686d3b65b78d1-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/402947d480674f578b44c42e194ef714/8e5079a430654fe4a9901fad3e5a9a3a-71ddb15e8eaa40b4179db24e849427d2-hd.m3u8" type="application/x-mpegURL"></video>

你好，我是温铭。

今天的内容，我同样会以视频的形式来讲解。老规矩，在你进行视频学习之前，先问你这么几个问题：

- 如何在开源项目中找到可能存在的性能问题？
- 在 Github 上，如何与其他开发者正确地交流？

这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。

同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。

## 今日核心

[ingress-nginx](https://github.com/kubernetes/ingress-nginx) 是 k8s 官方的一个项目，主要使用Go、 Nginx 和 lua-nginx-module 来处理入口流量。

在今天的视频中，我会为你清楚介绍，如何运用我们刚刚学习的性能优化方面的知识，来发现开源项目的性能问题。要知道，在我们给开源项目贡献 PR 时，跑通测试案例集以及与项目维护者积极沟通，都是非常重要的。

下面是 ingress-nginx 中，和 OpenResty 性能相关的两个 PR：

- [https://github.com/kubernetes/ingress-nginx/pull/3673](https://github.com/kubernetes/ingress-nginx/pull/3673)
- [https://github.com/kubernetes/ingress-nginx/pull/3674](https://github.com/kubernetes/ingress-nginx/pull/3674)

从中你也可以发现，即使是资深的开发者，对 LuaJIT 相关的优化，可能也并不是很熟悉。一方面是因为，这两个 PR 涉及到的代码，并不会对整体系统造成严重的性能下降；另一个方面，这方面的优化知识，没有人系统地总结过，开发者即使想优化也找不到方向。

事实上，很多时候，我们站在代码可读性和可维护性的角度来看，可有可无的优化是不必要的，你只要去优化那些被频繁执行的代码片段就可以了，过度优化是万恶之源。

那么，学完今天这节课后，你是否可以在其他的开源项目中，找到类似的性能优化点呢？

## 课件参考

今天的课件已经上传到了我的GitHub上，你可以自己下载学习。

链接如下：[https://github.com/iresty/geektime-slides](https://github.com/iresty/geektime-slides)

如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。
