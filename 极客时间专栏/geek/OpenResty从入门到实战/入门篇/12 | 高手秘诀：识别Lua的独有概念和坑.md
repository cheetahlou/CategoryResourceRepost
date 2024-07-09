<audio id="audio" title="12 | 高手秘诀：识别Lua的独有概念和坑" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/f7/14/f738771f4d119db9326fc0607719e414.mp3"></audio>

你好，我是温铭。

上一节中，我们一起了解了 LuaJIT 中 table 相关的库函数。除了这些常用的函数外，今天我再为你介绍一些Lua 独有的或不太常用的概念，以及 OpenResty 中常见的 Lua 的坑。

## 弱表

首先是 `弱表`（weak table），它是 Lua 中很独特的一个概念，和垃圾回收相关。和其他高级语言一样，Lua 是自动垃圾回收的，你不用关心具体的实现，也不用显式 GC。没有被引用到的空间，会被垃圾收集器自动完成回收。

但简单的引用计数还不太够用，有时候我们需要一种更灵活的机制。举个例子，我们把一个 Lua 的对象 `Foo`（table 或者函数）插入到 table `tb` 中，这就会产生对这个对象 `Foo` 的引用。即使没有其他地方引用 `Foo`，`tb` 对它的引用也还一直存在，那么 GC 就没有办法回收 `Foo` 所占用的内存。这时候，我们就只有两种选择：

- 一是手工释放 `Foo`；
- 二是让它常驻内存。

比如下面这段代码：

```
$ resty -e 'local tb = {}
tb[1] = {red}
tb[2] = function() print(&quot;func&quot;) end
print(#tb) -- 2

collectgarbage()
print(#tb) -- 2

table.remove(tb, 1)
print(#tb) -- 1

```

不过，你肯定不希望，内存一直被用不到的对象占用着吧，特别是 LuaJIT 中还有 2G 内存的上限。而手工释放的时机并不好把握，也会增加代码的复杂度。

那么这时候，就轮到弱表来大显身手了。看它的名字，弱表，首先它是一个表，然后这个表里面的所有元素都是弱引用。概念总是抽象的，让我们先来看一段稍加修改后的代码：

```
$ resty -e 'local tb = {}
tb[1] = {red}
tb[2] = function() print(&quot;func&quot;) end
setmetatable(tb, {__mode = &quot;v&quot;})
print(#tb)  -- 2

collectgarbage()
print(#tb) -- 0
'

```

可以看到，没有被使用的对象都被 GC 了。这其中，最重要的就是下面这一行代码：

```
setmetatable(tb, {__mode = &quot;v&quot;})

```

是不是似曾相识？这不就是元表的操作吗！没错，当一个 table 的元表中存在 `__mode` 字段时，这个 table 就是弱表（weak table）了。

- 如果 `__mode` 的值是 `k`，那就意味着这个 table 的 `键` 是弱引用。
- 如果 `__mode` 的值是 `v`，那就意味着这个 table 的 `值` 是弱引用。
- 当然，你也可以设置为 `kv`，表明这个表的键和值都是弱引用。

这三者中的任意一种弱表，只要它的 `键` 或者 `值` 被回收了，那么对应的**整个**`键值` 对象都会被回收。

在上面的代码示例中，`__mode` 的值 `v`，而`tb` 是一个数组，数组的 `value` 则是 table 和函数对象，所以可以被自动回收。不过，如果你把`__mode` 的值改为 `k`，就不会 GC 了，比如看下面这段代码：

```
$ resty -e 'local tb = {}
tb[1] = {red}
tb[2] = function() print(&quot;func&quot;) end
setmetatable(tb, {__mode = &quot;k&quot;})
print(#tb)  -- 2

collectgarbage()
print(#tb) -- 2
'

```

请注意，这里我们只演示了 `value` 为弱引用的弱表，也就是数组类型的弱表。自然，你同样可以把对象作为 `key`，来构建哈希表类型的弱表，比如下面这样写：

```
$ resty -e 'local tb = {}
tb[{color = red}] = &quot;red&quot;
local fc = function() print(&quot;func&quot;) end
tb[fc] = &quot;func&quot;
fc = nil

setmetatable(tb, {__mode = &quot;k&quot;})
for k,v in pairs(tb) do
     print(v)
end

collectgarbage()
print(&quot;----------&quot;)
for k,v in pairs(tb) do
     print(v)
end
'

```

在手动调用 `collectgarbage()` 进行强制 GC 后，`tb` 整个 table 里面的元素，就已经全部被回收了。当然，在实际的代码中，我们大可不必手动调用 `collectgarbage()`，它会在后台自动运行，无须我们担心。

不过，既然提到了 `collectgarbage()` 这个函数，我就再多说几句。这个函数其实可以传入多个不同的选项，且默认是 `collect`，即完整的 GC。另一个比较有用的是 `count`，它可以返回 Lua 占用的内存空间大小。这个统计数据很有用，可以让你看出是否存在内存泄漏，也可以提醒我们不要接近 2G 的上限值。

弱表相关的代码，在实际应用中会写得比较复杂，不太容易理解，相对应的，也会隐藏更多的 bug。具体有哪些呢？不必着急，后面内容，我会专门介绍一个开源项目中，使用弱表带来的内存泄漏问题。

## 闭包和 upvalue

再来看闭包和 upvalue。前面我强调过，在 Lua 中，所有的值都是一等公民，包含函数也是。这就意味着函数可以保存在变量中，当作参数传递，以及作为另一个函数的返回值。比如在上面弱表中出现的这段示例代码：

```
tb[2] = function() print(&quot;func&quot;) end

```

其实就是把一个匿名函数，作为 table 的值给存储了起来。

在 Lua 中，下面这段代码中动两个函数的定义是完全等价的。不过注意，后者是把函数赋值给一个变量，这也是我们经常会用到的一种方式：

```
local function foo() print(&quot;foo&quot;) end
local foo = fuction() print(&quot;foo&quot;) end

```

另外，Lua 支持把一个函数写在另外一个函数里面，即嵌套函数，比如下面的示例代码：

```
$ resty -e '
local function foo()
     local i = 1
     local function bar()
         i = i + 1
         print(i)
     end
     return bar
end

local fn = foo()
print(fn()) -- 2
'

```

你可以看到， `bar` 这个函数可以读取函数 `foo` 里面的局部变量 `i`，并修改它的值，即使这个变量并不在 `bar` 里面定义。这个特性叫做词法作用域（lexical scoping）。

事实上，Lua 的这些特性正是闭包的基础。所谓`闭包` ，简单地理解，它其实是一个函数，不过它访问了另外一个函数词法作用域中的变量。

如果按照闭包的定义来看，Lua 的所有函数实际上都是闭包，即使你没有嵌套。这是因为 Lua 编译器会把 Lua 脚本外面，再包装一层主函数。比如下面这几行简单的代码段：

```
local foo, bar
local function fn()
     foo = 1
     bar = 2
end

```

在编译后，就会变为下面的样子：

```
function main(...)
     local foo, bar
     local function fn()
         foo = 1
         bar = 2
     end
end

```

而函数 `fn` 捕获了主函数的两个局部变量，因此也是闭包。

当然，我们知道，很多语言中都有闭包的概念，它并非 Lua 独有，你也可以对比着来加深理解。只有理解了闭包，你才能明白我们接下来要讲的 upvalue。

upvalue 就是 Lua 中独有的概念了。从字面意思来看，可以翻译成 `上面的值`。实际上，upvalue 就是闭包中捕获的自己词法作用域外的那个变量。还是继续看上面那段代码：

```
local foo, bar
local function fn()
     foo = 1
     bar = 2
end

```

你可以看到，函数 `fn` 捕获了两个不在自己词法作用域的局部变量 `foo` 和 `bar`，而这两个变量，实际上就是函数 `fn` 的 upvalue。

## 常见的坑

介绍了 Lua 中的几个概念后，我再来说说，在 OpenResty 开发中遇到的那些和 Lua 相关的坑。

在前面内容中，我们提到了一些 Lua 和其他开发语言不同的点，比如下标从 1 开始、默认全局变量等等。在 OpenResty 实际的代码开发中，我们还会遇到更多和 Lua、 LuaJIT 相关的问题点， 下面我会讲其中一些比较常见的。

这里要先提醒一下，即使你知道了所有的 `坑`，但不可避免的，估计还是要自己踩过之后才能印象深刻。当然，不同的是，你能够更块地从坑里面爬出来，并找到症结所在。

### 下标从 0 开始还是从 1 开始

第一个坑，Lua 的下标是从 1 开始的，这点我们之前反复提及过。但我不得不说，这并非事实的全部。

因为在 LuaJIT 中，使用 `ffi.new` 创建的数组，下标又是从 0 开始的:

```
local buf = ffi_new(&quot;char[?]&quot;, 128)

```

所以，如果你要访问上面这段代码中 `buf` 这个 cdata，请记得下标从 0 开始，而不是 1。在使用 FFI 和 C 交互的时候，一定要特别注意这个地方。

### 正则模式匹配

第二个坑，正则模式匹配问题。OpenResty 中并行着两套字符串匹配方法：Lua 自带的 `sting` 库，以及 OpenResty 提供的 `ngx.re.*` API。

其中， Lua 正则模式匹配是自己独有的格式，和 PCRE 的写法不同。下面是一个简单的示例：

```
resty -e 'print(string.match(&quot;foo 123 bar&quot;, &quot;%d%d%d&quot;))'  — 123

```

这段代码从字符串中提取了数字部分，你会发现，它和我们的熟悉的正则表达式完全不同。Lua 自带的正则匹配库，不仅代码维护成本高，而且性能低——不能被 JIT，而且被编译过一次的模式也不会被缓存。

所以，在你使用 Lua 内置的 string 库去做 find、match 等操作时，如果有类似正则这样的需求，不用犹豫，请直接使用 OpenResty 提供的 `ngx.re` 来替代。只有在查找固定字符串的时候，我们才考虑使用 plain 模式来调用 string 库。

**这里我有一个建议：在 OpenResty 中，我们总是优先使用 OpenResty 的 API，然后是 LuaJIT 的 API，使用 Lua 库则需要慎之又慎**。

### json 编码时无法区分 array 和 dict

第三个坑，json 编码时无法区分 array 和 dict。由于 Lua 中只有 table 这一个数据结构，所以在 json 对空 table 编码的时候，自然就无法确定编码为数组还是字典：

```
resty -e 'local cjson = require &quot;cjson&quot;
local t = {}
print(cjson.encode(t))
'

```

比如上面这段代码，它的输出是 `{}`，由此可见， OpenResty 的 cjson 库，默认把空 table 当做字典来编码。当然，我们可以通过 `encode_empty_table_as_object` 这个函数，来修改这个全局的默认值：

```
resty -e 'local cjson = require &quot;cjson&quot;
cjson.encode_empty_table_as_object(false)
local t = {}
print(cjson.encode(t))
'

```

这次，空 table 就被编码为了数组：`[]`。

不过，全局这种设置的影响面比较大，那能不能指定某个 table 的编码规则呢？答案自然是可以的，我们有两种方法可以做到。

第一种方法，把 `cjson.empty_array` 这个 userdata 赋值给指定 table。这样，在 json 编码的时候，它就会被当做空数组来处理：

```
$ resty -e 'local cjson = require &quot;cjson&quot;
local t = cjson.empty_array
print(cjson.encode(t))
'

```

不过，有时候我们并不确定，这个指定的 table 是否一直为空。我们希望当它为空的时候编码为数组，那么就要用到 `cjson.empty_array_mt` 这个函数，也就是我们的第二个方法。

它会标记好指定的 table，当 table 为空时编码为数组。从`cjson.empty_array_mt` 这个命名你也可以看出，它是通过 metatable 的方式进行设置的，比如下面这段代码操作：

```
$ resty -e 'local cjson = require &quot;cjson&quot;
local t = {}
setmetatable(t, cjson.empty_array_mt)
print(cjson.encode(t))
t = {123}
print(cjson.encode(t))
'

```

你可以在本地执行一下这段代码，看看输出和你预期的是否一致。

### 变量的个数限制

再来看第四个坑，变量的个数限制问题。 Lua 中，一个函数的局部变量的个数，和 upvalue 的个数都是有上限的，你可以从 Lua 的源码中得到印证：

```

/*
@@ LUAI_MAXVARS is the maximum number of local variables per function
@* (must be smaller than 250).
*/
#define LUAI_MAXVARS            200


/*
@@ LUAI_MAXUPVALUES is the maximum number of upvalues per function
@* (must be smaller than 250).
*/
#define LUAI_MAXUPVALUES        60

```

这两个阈值，分别被硬编码为 200 和 60。虽说你可以手动修改源码来调整这两个值，不过最大也只能设置为 250。

一般情况下，我们不会超过这个阈值，但写 OpenResty 代码的时候，你还是要留意这个事情，不要过多地使用局部变量和 upvalue，而是要尽可能地使用 `do .. end` 做一层封装，来减少局部变量和 upvalue 的个数。

比如我们来看下面这段伪码：

```
local re_find = ngx.re.find
  function foo() ... end
function bar() ... end
function fn() ... end

```

如果只有函数 `foo` 使用到了 `re_find`， 那么我们可以这样改造下：

```
do
     local re_find = ngx.re.find
     function foo() ... end
end
function bar() ... end
function fn() ... end

```

这样一来，在 `main` 函数的层面上，就少了 `re_find` 这个局部变量。这在单个的大的 Lua 文件中，算是一个优化技巧。

## 写在最后

从“多问几个为什么”的角度出发，Lua 中 250 这个阈值是从何而来的呢？这算是我们今天的思考题，欢迎你留言说下你的看法，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。
