<audio id="audio" title="28 | test::nginx 还可以这样用？" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/3f/21/3f20a1b788027e4950de581b9bce6621.mp3"></audio>

你好，我是温铭。

在前面两个章节中，你已经掌握了 `test::nginx` 的大部分使用方法，我相信你已经能够看明白 OpenResty 项目中大部分的测试案例集了。这对于学习 OpenResty 和它的周边库而言，已经足够了。

但如果你有志于成为 OpenResty 的代码贡献者，或者你正在自己的项目中使用 `test::nginx` 来编写测试案例，那么你还需要来学习一些更高级、更复杂的用法。

今天的内容，可能会是这个专栏中最“高冷”的部分，因为这都是从来没有人分享过的内容。 以 lua-nginx-module 这个 OpenResty 中最核心的模块为例，全球一共有 70 多个贡献者，但并非每个贡献者都写过测试案例。所以，如果学完今天的课程，你在 `test::nginx` 上的理解，绝对可以进入全球 Top 100。

## 测试中的调试

首先，我们来看几个最简单、也是开发者最常用到的原语，它们在平时的调试中会被使用到。下面，我们就来依次介绍下，这几个调试相关的原语的使用场景。

### ONLY

很多时候，我们都是在原有的测试案例集基础上，新增了一个测试案例。如果这个测试文件包含了很多的测试案例，那么从头到尾跑一遍显然是比较耗时的，这在你需要反复修改测试案例的时候尤为明显。

那么，有没有什么方法只运行你指定的某一个测试案例呢？ `ONLY` 这个标记可以轻松实现这一点：

```
=== TEST 1: sanity
=== TEST 2: get
--- ONLY

```

上面这段伪码就展示了如何使用这个原语。把 `--- ONLY` 放在需要单独运行的测试案例的最后一行，那么使用 prove 来运行这个测试案例文件的时候，就会忽略其他所有的测试案例，只运行这一个测试了。

不过，这只适合在你做调试的时候使用。所以， prove 命令发现 ONLY 标记的时候，也会给出提示，告诉你不要忘记在提交代码时把它去掉。

### SKIP

与只执行一个测试案例对应的需求，就是忽略掉某一个测试案例。`SKIP` 这个标记，一般用于测试尚未实现的功能：

```
=== TEST 1: sanity
=== TEST 2: get
--- SKIP

```

从这段伪码你可以看到，它的用法和ONLY类似。因为我们是测试驱动开发，需要先编写测试案例；而在集体编码实现时，可能由于实现难度或者优先级的关系，导致某个功能需要延后实现。那么这时候，你就可以先跳过对应的测试案例集，等实现完成后，再把 SKIP 标记去掉即可。

### LAST

还有一个常用的标记是 `LAST`，它的用法也很简单，在它之前的测试案例集都会被执行，后面的就会被忽略掉：

```
=== TEST 1: sanity
=== TEST 2: get
--- LAST
=== TEST 3: set

```

你可能疑惑，ONLY和SKIP我能理解，但LAST这个功能有什么用呢？实际上，有时候你的测试案例是有依赖关系的，需要你执行完前面几个测试案例后，之后的测试才有意义。那么，在这种情况下去调试的话，LAST 就非常有用了。

## 测试计划 plan

在 `test::nginx` 所有的原语中，`plan` 是最容易让人抓狂、也是最难理解的一个。它源自于 perl 的 `Test::Plan` 模块，所以文档并不在 `test::nginx`中，找到它的解释并不容易，所以我把它放在靠前的位置来介绍。我见过好几个 OpenResty 的代码贡献者，都在这个坑里面跌倒，甚至爬不出来。

下面是一个示例，在 OpenResty 官方测试集的每一个文件的开始部分，你都能看到类似的配置：

```
plan tests =&gt; repeat_each() * (3 * blocks());

```

这里 plan 的含义是，在整个测试文件中，按照计划应该会做多少次检测项。如果最终运行的结果和计划不符，整个测试就会失败。

拿这个示例来说，如果 `repeat_each` 的值是 2，一共有 10 个测试案例，那么 plan 的值就应该是2 x 3 x 10 = 60。这里估计你唯一搞不清楚的，就是数字 3 的含义吧，看上去完全是一个 magic number！

别着急，我们继续看示例，一会儿你就能搞懂了。先来说说，你能算清楚下面这个测试案例中，plan 的正确值是多少吗？

```
=== TEST 1: sanity
--- config
    location /t {
        content_by_lua_block {
            ngx.say(&quot;hello&quot;)
        }
    }
--- request
GET /t
--- response_body
hello

```

我相信所有人都会得出 plan = 1 的结论，因为测试中只对 `response_body` 进行了校验。

但，事实并非如此！正确的答案是， plan = 2。为什么呢？因为 `test::nginx` 中隐含了一个校验，也就是`--- error_code: 200`，它默认检测  HTTP 的 response code 是否为 200。

所以，上面的 magic number 3，真实含义是在每一个测试中都显式地检测了两次，比如 body 和 error log；同时，隐式地检测了 response code。

由于这个地方太容易出错，所以，我的建议是，推荐你用下面的方法，直接关闭掉 plan：

```
use Test::Nginx::Socket 'no_plan';

```

如果无法关闭，比如在 OpenResty 的官方测试集中遇到 plan 不准确的情况，建议你也不要去深究原因，直接在 plan 的表达式中增加或者减少数字即可：

```
plan tests =&gt; repeat_each() * (3 * blocks()) + 2;

```

这也是官方会使用到的方法。

## 预处理器

我们知道，在同一个测试文件的不同测试案例之间，可能会有一些共同的设置。如果在每一个测试案例中都重复设置，就会让代码显得冗余，后面修改起来也比较麻烦。

这时候，你就可以使用 `add_block_preprocessor` 指令，来增加一段 perl 代码，比如下面这样来写：

```
add_block_preprocessor(sub {
    my $block = shift;

    if (!defined $block-&gt;config) {
        $block-&gt;set_value(&quot;config&quot;, &lt;&lt;'_END_');
    location = /t {
        echo $arg_a;
    }
    _END_
    }
});

```

这个预处理器，就会为所有的测试案例，都增加一段 config 的配置，而里面的内容就是 `location /t`。这样，在你后面的测试案例里，就都可以省略掉 config，直接访问即可：

```
=== TEST 1:
--- request
    GET /t?a=3
--- response_body
3

=== TEST 2:
--- request
    GET /t?a=blah
--- response_body
blah

```

## 自定义函数

除了在预处理器中增加 perl 代码之外，你还可以在 `run_tests` 原语之前，随意地增加 perl 函数，也就是我们所说的自定义函数。

下面是一个示例，它增加了一个读取文件的函数，并结合 `eval` 指令，一起实现了 POST 文件的功能：

```
sub read_file {
    my $infile = shift;
    open my $in, $infile
        or die &quot;cannot open $infile for reading: $!&quot;;
    my $content = do { local $/; &lt;$in&gt; };
    close $in;
    $content;
}

our $CONTENT = read_file(&quot;t/test.jpg&quot;);

run_tests;

__DATA__

=== TEST 1: sanity
--- request eval
&quot;POST /\n$::CONTENT&quot;

```

## 乱序

除了上面几点外，`test::nginx` 还有一个鲜为人知的坑：默认乱序、随机来执行测试案例，而非按照测试案例的前后顺序和编号来执行。

它的初衷是想测试出更多的问题。毕竟，每一个测试案例运行完后，都会关闭 Nginx 进程，并启动新的 Nginx 来执行，结果不应该和顺序相关才对。

对于底层的项目而言，确实如此。但是，对于应用层的项目来说，外部存在数据库等持久化存储。这时候的乱序执行，就会导致错误的结果。由于每次都是随机的，所以可能报错，也可能不报错，每次的报错还可能不同。这显然会给开发者带来困惑，就连我都在这里跌倒过好多次。

所以，我的忠告就是：请关闭掉这个特性。你可以用下面这两行代码来关闭：

```
no_shuffle();
run_tests;

```

其中，`no_shuffle` 原语就是用来禁用随机，让测试严格按照测试案例的前后顺序来运行。

## reindex

最后，让我们聊一个不烧脑的、轻松一点儿的话题。OpenResty 的测试案例集，对格式有着严格的要求。每个测试案例之间都需要有 3 个换行来分割，测试案例的编号也要严格保持自增长。

幸好，我们有对应的自动化工具 `reindex` 来做这些繁琐的事情，它隐藏在 [[ openresty-devel-utils]](https://github.com/openresty/openresty-devel-utils)  项目中，因为没有文档来介绍，知道的人很少。

有兴趣的同学，可以尝试着把测试案例的编号打乱，或者增删分割的换行个数，然后用这个工具来整理下，看看是否可以还原。

## 写在最后

关于 `test::nginx` 的介绍就到此结束了。当然，它的功能其实还有更多，我们只讲了最核心最重要的一些。授人以鱼不如授人以渔，学习测试的基本方法和注意点我都已经教给你了，剩下的就需要你自己去官方的测试案例集中去挖掘了。

最后给你留一个问题。在你的项目开发中，是否有测试？你又是使用什么框架来测试的呢？欢迎留言和我交流这个问题，也欢迎你把这篇文章分享给更多的人，一起交流和学习。


