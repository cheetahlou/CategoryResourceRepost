
<video poster="https://static001.geekbang.org/resource/image/0b/02/0bf3ba61bafc3a97514996c701c99e02.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/1d56d7e0-16ce8221564-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/98c47d3154564eb08b624deb1aa4e260/d32353463ce34201b74a0b3807160f6b-cf761b25fd08f06e963332465b9b0f4c-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/98c47d3154564eb08b624deb1aa4e260/d32353463ce34201b74a0b3807160f6b-335f34e72825be78cd1db071594c6c08-hd.m3u8" type="application/x-mpegURL"></video>

你好，我是温铭。

今天是我们专栏中的最后一节视频课了，后面内容仍然以图文形式呈现。老规矩，为了更有针对性地学习，在你进行视频学习之前，我想先问你这么几个问题：

- 你测试过 OpenResty 程序的性能吗？如何才能科学地找到性能瓶颈？
- 如何看懂火焰图的信息，并与 Lua 代码相对应呢？

这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。

同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。

## 今日核心

今天的视频课，我会用一个开源的小项目来演示一下，如何通过 wrk 和火焰图来优化代码，这个项目地址为：[https://github.com/iresty/lua-performance-demo](https://github.com/iresty/lua-performance-demo)。

视频中的环境是 Ubuntu 16.04，其中的 systemtap 和 wrk 工具，都是使用 apt-get 来安装的，不推荐你用源码来安装。

这里的demo 有几个不同的版本，我会用 wrk 来压测每一个版本的 qps。同时，在压测过程中，我都会使用 stapxx 来生成火焰图，并用火焰图来指导我们去优化哪一个函数和代码块。

最后的结果是，我们会看到一个性能提升 10 倍以上的版本，当然，这其中的优化方式，都是在专栏前面课程中提到过的。建议你可以 clone 这个 demo 项目，来复现我在视频中的操作，加深对 wrk、火焰图和性能优化的理解。

要知道，性能优化并不是感性和直觉的判断，而是需要科学的数据来做指导的。这里的数据，不仅仅是指 qps 等最终的性能指标，也包括了用数据来定位具体的瓶颈。

## 课件参考

今天的课件已经上传到了我的GitHub上，你可以自己下载学习。

链接如下：[https://github.com/iresty/geektime-slides](https://github.com/iresty/geektime-slides)

如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。
