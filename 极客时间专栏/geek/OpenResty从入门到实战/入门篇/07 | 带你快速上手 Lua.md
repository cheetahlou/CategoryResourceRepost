<audio id="audio" title="07 | 带你快速上手 Lua" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/61/ab/61cbc24102eb4f04301c96a0f2f845ab.mp3"></audio>

你好，我是温铭。

在大概了解 NGINX 的基础知识后，接下来，我们就要来进一步学习 Lua了。它是 OpenResty 中使用的编程语言，掌握它的基本语法还是很有必要的。

Lua 是一个小巧精妙的脚本语言，诞生于巴西的大学实验室，这个名字在葡萄牙语里的含义是“美丽的月亮”。从作者所在的国家来看，NGINX 诞生于俄罗斯，Lua 诞生于巴西，OpenResty 诞生于中国，这三门同样精巧的开源技术都出自金砖国家，而不是欧美，也是挺有趣的一件事。

回到Lua语言上。事实上，Lua 在设计之初，就把自己定位为一个简单、轻量、可嵌入的胶水语言，没有走大而全的路线。虽然你平常工作中可能没有直接编写 Lua 代码，但 Lua 的使用其实非常广泛。很多的网游，比如魔兽世界，都会采用 Lua 来编写插件；而键值数据库 Redis 则是内置了 Lua 来控制逻辑。

另一方面，虽然 Lua 自身的库比较简单，但它可以方便地调用 C 库，大量成熟的 C 代码都可以为其所用。比如在 OpenResty 中，很多时候都需要你调用 NGINX 和 OpenSSL 的 C 函数，而这都得益于 Lua 和 LuaJIT 这种方便调用 C 库的能力。

下面，我带你来快速熟悉下 Lua 的数据类型和语法，以便你后面更顺畅地学习 OpenResty。

## 环境和 hello world

我们不用专门去安装标准 Lua 5.1 之类的环境，因为 OpenResty 已经不再支持标准 Lua，而只支持 LuaJIT。这里我介绍的 Lua 语法，也是和 LuaJIT 兼容的部分，而不是基于最新的 Lua 5.3，这一点需要你特别注意。

在 OpenResty 的安装目录下，你可以找到 LuaJIT 的目录和可执行文件。我这里是 Mac 环境，使用 brew 安装 OpenResty，所以你本地的路径很可能和下面的不同：

```
$ ll /usr/local/Cellar/openresty/1.13.6.2/luajit/bin/luajit
 lrwxr-xr-x  1 ming  admin    18B  4  2 14:54 /usr/local/Cellar/openresty/1.13.6.2/luajit/bin/luajit -&gt; luajit-2.1.0-beta3

```

你也可以在系统的可执行文件目录中找到它：

```
$ which luajit
 /usr/local/bin/luajit

```

并查看 LuaJIT 的版本号：

```
$ luajit -v
 LuaJIT 2.1.0-beta2 -- Copyright (C) 2005-2017 Mike Pall. http://luajit.org/

```

查清楚这些信息后，你可以新建一个 `1.lua` 文件，并用 luajit 来运行其中的 hello world 代码：

```
$ cat 1.lua
print(&quot;hello world&quot;)

$ luajit 1.lua
 hello world

```

当然，你还可以使用 `resty` 来直接运行，要知道，它最终也是用 LuaJIT 来执行的：

```
$ resty -e 'print(&quot;hello world&quot;)'
 hello world

```

上述两种运行 hello world 的方式都是可行的。不顾对我来说，我更喜欢 `resty` 这种方式，因为后面很多 OpenResty 的代码，也都是通过 `resty` 来运行的。

## 数据类型

Lua 中的数据类型不多，你可以通过 `type` 函数来返回一个值的类型，比如下面这样的操作：

```
$ resty -e 'print(type(&quot;hello world&quot;)) 
 print(type(print)) 
 print(type(true)) 
 print(type(360.0))
 print(type({}))
 print(type(nil))
 '

```

会打印出如下内容：

```
 string
 function
 boolean
 number
 table
 nil

```

这几种就是 Lua 中的基本数据类型了。下面我们来简单介绍一下它们。

### 字符串

在 Lua 中，字符串是不可变的值，如果你要修改某个字符串，就等于创建了一个新的字符串。这种做法显然有利有弊：好处是即使同一个字符串出现了很多次，在内存中也只有一份；但劣势也很明显，如果你想修改、拼接字符串，会额外地创建很多不必要的字符串。

我们举一个例子，来说明这个弊端。下面这段代码，是把 1 到 10 这些数字当作字符串拼接起来。对了，在 Lua 中，我们使用两个点号来表示字符串的相加：

```
$ resty -e 'local s  = &quot;&quot;
 for i = 1, 10 do
     s = s .. tostring(i)
 end
 print(s)'

```

这里我们循环了 10 次，但只有最后一次是我们想要的，而中间新建的 9 个字符串都是无用的。它们不仅占用了额外的空间，也消耗了不必要的 CPU 运算。

当然，在后面的性能优化章节，我们会有对应的方法来解决它。

另外，在 Lua 中，你有三种方式可以表达一个字符串：单引号、双引号，以及长括号（`[[]]`）。前面两种都比较好理解，别的语言一般也这么用，那么长括号有什么用处呢？

我们看一个具体的示例：

```
$ resty -e 'print([[string has \n and \r]])'
 string has \n and \r

```

你可以看到，长括号中的字符串不会做任何的转义处理。

你也许会问另外一个问题：如果上面那段字符串中包括了长括号本身，又该怎么处理呢？答案很简单，就是在长括号中间增加一个或者多个 `=` 符号：

```
$ resty -e 'print([=[ string has a [[]]. ]=])'
  string has a [[]].

```

### 布尔值

这个很简单，true 和 false。但在 Lua 中，只有 nil 和 false 为假，其他都为真，包括 0 和空字符串也为真。我们可以用下面的代码印证一下：

```
$ resty -e 'local a = 0
 if a then
   print(&quot;true&quot;)
 end
 a = &quot;&quot;
 if a then
   print(&quot;true&quot;)
 end'

```

这种判断方式和很多常见的开发语言并不一致，所以，为了避免在这种问题上出错，你可以显式地写明比较的对象，比如下面这样：

```
$ resty -e 'local a = 0
 if a == false then
   print(&quot;true&quot;)
 end
 '


```

### 数字

Lua 的 number 类型，是用双精度浮点数来实现的。值得一提的是，LuaJIT 支持 `dual-number`（双数）模式，也就是说， LuaJIT 会根据上下文来用整型来存储整数，而用双精度浮点数来存放浮点数。

此外，LuaJIT 还支持`长长整型`的大整数，比如下面的例子：

```
$ resty -e 'print(9223372036854775807LL - 1)'
9223372036854775806LL

```

### 函数

函数在 Lua 中是一等公民，你可以把函数存放在一个变量中，也可以当作另外一个函数的入参和出参。

比如，下面两个函数的声明是完全等价的：

```
function foo()
 end

```

和

```
foo = function ()
 end

```

### table

table 是 Lua 中唯一的数据结构，自然非常重要，所以后面我会用专门的章节来介绍它。我们可以先来看一个简单的示例代码：

```
$ resty -e 'local color = {first = &quot;red&quot;}
print(color[&quot;first&quot;])'
 red

```

### 空值

在 Lua 中，空值就是 nil。如果你定义了一个变量，但没有赋值，它的默认值就是 nil：

```
$ resty -e 'local a
 print(type(a))'
 nil

```

当你真正进入 OpenResty 体系中后，会发现很多种空值，比如 `ngx.null` 等等，我们后面再细聊。

Lua的数据类型，我主要就介绍这么多，先给你打个基础。一些需要重点掌握的内容，后面的文章中我们都会继续学习。在练习、使用中学习，永远是吸收新知识最便捷的方式。

## 常用标准库

很多时候，我们学习一门语言，其实就是在学习它的标准库。

Lua 比较小巧，内置的标准库并不多。而且，在 OpenResty 的环境中，Lua 标准库的优先级是很低的。对于同一个功能，我更推荐你优先使用 OpenResty 的 API 来解决，然后是 LuaJIT 的库函数，最后才是标准 Lua 的函数。

`OpenResty的API &gt; LuaJIT的库函数 &gt; 标准Lua的函数`，这个优先级后面会被反复提及，它不仅关系到是否好用这一点，更会对性能产生非常大的影响。

不过，尽管如此，在实际的项目开发中，我们还是不可避免会用到一些 Lua 库。这里，我挑选了几个比较常用的标准库做下介绍，如果你想要了解更多内容，可以查阅 Lua 的官方文档。

### string 库

字符串操作是我们最常用到的，也是坑最多的地方。有一个简单的原则，那就是如果涉及到正则表达式的，请一定要使用 OpenResty 提供的 `ngx.re.*` 来解决，不要用 Lua 的 `string.*` 处理。这是因为，Lua 的正则独树一帜，不符合 PCRE 的规范，我相信绝大部分工程师是玩不转的。

其中 `string.byte(s [, i [, j ]])`，是比较常用到的一个 string 库函数，它返回字符 s[i]、s[i + 1]、s[i + 2]、······、s[j] 所对应的 ASCII 码。i 的默认值为 1，即第一个字节，j 的默认值为 i。

下面我们来看一段示例代码：

```
$ resty -e 'print(string.byte(&quot;abc&quot;, 1, 3))
 print(string.byte(&quot;abc&quot;, 3)) -- 缺少第三个参数，第三个参数默认与第二个相同，此时为 3
 print(string.byte(&quot;abc&quot;))    -- 缺少第二个和第三个参数，此时这两个参数都默认为 1
 '

```

它的输出为：

```
 979899
 99
 97

```

### table 库

在 OpenResty 的上下文中，对于Lua 自带的 table 库，除了 `table.concat` 、`table.sort` 等少数几个函数，大部分我都不推荐使用。至于它们的细节，我们留在 LuaJIT 章节中专门来讲。

这里我简单提一下`table.concat` 。`table.concat`一般用在字符串拼接的场景下，比如下面这个例子。它可以避免生成很多无用的字符串。

```
$ resty -e 'local a = {&quot;A&quot;, &quot;b&quot;, &quot;C&quot;}
 print(table.concat(a))'

```

### math 库

Lua math 库由一组标准的数学函数构成。数学库的引入，既丰富了 Lua 编程语言的功能，同时也方便了程序的编写。

在 OpenResty 的实际项目中，我们很少用 Lua 去做数学方面的运算，不过其中和随机数相关的 `math.random()` 和 `math.randomseed()` 两个函数，倒是比较常用，比如下面的这段代码，它可以在指定的范围内，随机地生成两个数字。

```
$ resty -e 'math.randomseed (os.time()) 
print(math.random())
 print(math.random(100))'

```

## 虚变量

了解了这些常见的标准库，接下来，我们再来学习一个新的概念——虚变量。

设想这么一个场景，当一个函数返回多个值的时候，有些返回值我们并不需要，这时候，应该怎么接收这些值呢？

不知道你是怎么看待这件事的，起码对我来说，要想法设法给这些用不到的变量，去赋予有意义的名字，着实是一件很折磨人的事情。

还好， Lua 中可以完美地解决这一点。Lua 提供了一个虚变量（dummy variable）的概念， 按照惯例以一个下划线来命名，用来表示丢弃不需要的数值，仅仅起到占位的作用。

下面我们以 `string.find` 这个标准库函数为例，来看虚变量的用法。这个标准库函数会返回两个值，分别代表开始和结束的下标。

如果我们只需要获取开始的下标，那么很简单，只声明一个变量来接收 `string.find` 的返回值即可：

```
$ resty -e 'local start = string.find(&quot;hello&quot;, &quot;he&quot;)
 print(start)'
 1

```

但如果你只想获取结束的下标，那就必须使用虚变量了：

```
$ resty -e 'local  _, end_pos = string.find(&quot;hello&quot;, &quot;he&quot;)
 print(end_pos)'
 2

```

除了在返回值里使用，虚变量还经常用于循环中，比如下面这个例子：

```
$ resty -e 'for _, v in ipairs({4,5,6}) do
     print(v)
 end'
 4
 5
 6

```

而当有多个返回值需要忽略时，你可以重复使用同一个虚变量。这里我就不举例子了，你可以试着自己写一个这样的示例代码吗？欢迎你把代码贴在留言区里和我分享、交流。

## 写在最后

今天，我们一起快速地学习了标准 Lua 的数据结构和语法，相信你对这门简单精巧的语言已经有了初步的了解。下节课，我会带你了解  Lua 和 LuaJIT 的关系，LuaJIT 更是 OpenResty 中的重头戏，值得我们深入挖掘。

最后，我想再为你留下一道思考题。

还记得这节课讲math库时，学过的这段代码吗？它可以在指定范围内，随机生成两个数字。

```
$ resty -e 'math.randomseed (os.time()) 
print(math.random())
 print(math.random(100))'

```

不过，你可能注意到了，这段代码是用当前时间戳作为种子的，那么这种方法是否有问题呢？又该如何生成好的种子呢？要知道，很多时候我们生成的随机数其实并不随机，并且有很大的安全隐患。

欢迎在留言区来说说你的看法，也欢迎你把这篇文章转发给你的同事、朋友。我们一起交流、一起进步。


