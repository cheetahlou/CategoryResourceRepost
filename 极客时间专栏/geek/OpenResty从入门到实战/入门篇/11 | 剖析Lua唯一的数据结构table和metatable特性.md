<audio id="audio" title="11 | 剖析Lua唯一的数据结构table和metatable特性" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/37/b7/37b91fa264c2af0acabd16235e1472b7.mp3"></audio>

你好，我是温铭。今天我们一起学习下LuaJIT 中唯一的数据结构：`table`。

和其他具有丰富数据结构的脚本语言不同，LuaJIT 中只有 `table` 这一个数据结构，并没有区分开数组、哈希、集合等概念，而是揉在了一起。让我们先温习下之前提到过的一个例子：

```
local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
print(color[&quot;first&quot;])                 --&gt; output: red
print(color[1])                         --&gt; output: blue
print(color[&quot;third&quot;])                --&gt; output: green
print(color[2])                         --&gt; output: yellow
print(color[3])                         --&gt; output: nil

```

这个例子中， `color` 这个 table 包含了数组和哈希，并且可以互不干扰地进行访问。比如，你可以用 `ipairs` 函数，只遍历数组部分的内容：

```
$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
for k, v in ipairs(color) do
     print(k)
end
'

```

`table` 的操作是如此重要，以至于 LuaJIT 对标准 Lua 5.1 的 table 库做了扩展，而 OpenResty 又对 LuaJIT 的 table 库做了更进一步的扩展。下面，我们就一起来分别看下这些库函数。

## table 库函数

先来看标准table 库函数。Lua 5.1 中自带的 table 库函数并不多，我们可以大概浏览一遍。

### `table.getn` 获取元素个数

我们在 `标准 Lua 和 LuaJIT` 章节中曾经提到过，想正确地获取到 table 所有元素的个数，在 LuaJIT 中是一个老大难问题。

对于序列，你用`table.getn` 或者一元操作符 `#` ，就可以正确返回元素的个数。比如下面这个例子，就会返回我们预期中的 3。

```
$ resty -e 'local t = { 1, 2, 3 }
print(table.getn(t)) '

```

而对于不是序列的 table，就无法返回正确的值。比如第二个例子，返回的就是 1。

```
$ resty -e 'local t = { 1, a = 2 }
print(#t) '

```

不过，幸运的是，这种难以理解的函数，已经被 LuaJIT 的扩展替代，后面我们会提到。所以在 OpenResty 的环境下，除非你明确知道，你正在获取序列的长度，否则请不要使用函数 `table.getn` 和一元操作符 `#` 。

另外，`table.getn` 和一元操作符 `#` 并不是 O(1) 的时间复杂度，而是 O(n)，这也是尽量避免使用它们的另外一个理由。

### `table.remove` 删除指定元素

第二个我们来看`table.remove` 函数，它的作用是在 table 中根据下标来删除元素，也就是说只能删除 table 中数组部分的元素。我们还是来看`color`的例子：

```
$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
  table.remove(color, 1)
  for k, v in pairs(color) do
      print(v)
  end'

```

这段代码会把下标为 1 的 `blue` 删除掉。你可能会问，那该如何删除 table 中的哈希部分呢？也很简单，把 key 对应的 value 设置为 `nil` 即可。这样，`color`这个例子中，`third` 对应的`green`就被删除了。

```
$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
  color.third = nil
  for k, v in pairs(color) do
      print(v)
  end'

```

### `table.concat` 元素拼接函数

第三个我们来看`table.concat` 元素拼接函数。它可以按照下标，把 table 中的元素拼接起来。既然这里又是根据下标来操作的，那么显然还是针对 table 的数组部分。同样还是`color`这个例子：

```
$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
print(table.concat(color, &quot;, &quot;))'

```

使用`table.concat`函数后，它输出的是 `blue, yellow`，哈希的部分被跳过了。

另外，这个函数还可以指定下标的起始位置来做拼接，比如下面这样的写法：

```
$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;, &quot;orange&quot;}
print(table.concat(color, &quot;, &quot;, 2, 3))'

```

这次输出是 `yellow, orange`，跳过了 `blue`。

你可能觉得这些操作还挺简单的，不过，我要说的是，函数不可貌相，海水不可。千万不要小看这个看上去没有太大用处的函数，在做性能优化时，它却会有意想不到的作用，也是我们后面性能优化章节中的主角之一。

### `table.insert` 插入一个元素

最后我们来看`table.insert` 函数。它可以下标插入一个新的元素，自然，影响的还是 table 的数组部分。还是用`color`例子来说明：

```
$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
table.insert(color, 1,  &quot;orange&quot;)
print(color[1])
'

```

你可以看到， color 的第一个元素变为了 orange。当然，你也可以不指定下标，这样就会默认插入队尾。

这里我必须说明的是，`table.insert` 虽然是一个很常见的操作，但性能并不乐观。如果你不是根据指定下标来插入元素，那么每次都需要调用 LuaJIT 的 `lj_tab_len` 来获取数组的长度，以便插入队尾。正如我们在 `table.getn` 中提到的，获取 table 长度的时间复杂度为 O(n) 。

所以，对于`table.insert` 操作，我们应该尽量避免在热代码中使用，比如：

```
local t = {}
for i = 1, 10000 do
     table.insert(t, i)
end

```

## LuaJIT 的 table 扩展函数

接下来我们来看LuaJIT 的 table 扩展函数。LuaJIT 在标准 Lua 的基础上，扩展了两个很有用的 table 函数，分别用来新建和清空一个 table，下面我具体来介绍一下。

### `table.new(narray, nhash)` 新建 table

第一个是`table.new(narray, nhash)` 函数。这个函数，会预先分配好指定的数组和哈希的空间大小，而不是在插入元素时自增长，这也是它的两个参数 `narray` 和 `nhash` 的含义。自增长是一个代价比较高的操作，会涉及到空间分配、`resize` 和 `rehash` 等，我们应该尽量避免。

这里注意，`table.new` 的文档并没有出现在 LuaJIT 的官网，而是深藏在 GitHub 项目的[扩展文档](https://github.com/openresty/luajit2/blob/v2.1-agentzh/doc/extensions.html)中，即使你用谷歌也难觅其踪迹，所以知道的工程师并不多。

下面是一个简单的例子，我来带你看下它该怎么用。首先要说明，这个函数是扩展出来的，所以在使用它之前，你需要先 `require` 一下：

```
local new_tab = require &quot;table.new&quot;
local t = new_tab(100, 0)
for i = 1, 100 do
   t[i] = i
end

```

你可以看到，这段代码新建了一个 table，里面包含 100 个数组元素和 0 个哈希元素。当然，你也可以根据实际需要，新建一个同时包含 100 个数组元素和 50 个 哈希元素的 table，这都是合法的：

```
local t = new_tab(100, 50)

```

另外，超出预设的空间大小，也可以正常使用，只不过性能会退化，也就失去了使用 `table.new` 的意义。

比如下面这个例子，我们预设大小为 100，而实际上却使用了 200：

```
local new_tab = require &quot;table.new&quot;
local t = new_tab(100, 0)
for i = 1, 200 do
   t[i] = i
end

```

所以，你需要根据实际场景，来预设好 `table.new` 中数组和哈希空间的大小，这样才能在性能和内存占用上找到一个平衡点。

### `table.clear()` 清空 table

第二个我们来看清空函数`table.clear()` 。它用来清空某个 table 里的所有数据，但并不会释放数组和哈希部分占用的内存。所以，它在循环利用 Lua table 时非常有用，可以避免反复创建和销毁 table 的开销。

```
$ resty -e 'local clear_tab =require &quot;table.clear&quot;
local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
clear_tab(color)
for k, v in pairs(color) do
     print(k)
end'

```

不过，事实上，能使用这个函数的场景并不算多，大多数情况下，我们还是应该把这个任务交给 LuaJIT GC 去完成。

## OpenResty 的 table 扩展函数

开头我提到过，OpenResty 自己维护的 LuaJIT 分支，也对 table 做了扩展，它[新增了几个 API](https://github.com/openresty/luajit2/#new-api)：`table.isempty`、`table.isarray`、 `table.nkeys` 和 `table.clone`。

需要注意的是，在使用这几个新增的 API 前，请记住检查你使用的 OpenResty 的版本，这些API 大都只能在 OpenResty 1.15.8.1 之后的版本中使用。这是因为， OpenResty 在 1.15.8.1 版本之前，已经有一年左右没有发布新版本了，而这些 API 是在这个发布间隔中新增的。

文章中我已经附上了链接，这里我就只用 `table.nkeys` 来举例说明下，其他的三个 API 从命名上来说都非常容易理解，你自己翻阅 GitHub 上的文档就可以明白了。不得不说，OpenResty 的文档质量非常高，其中包含了代码示例、能否被 JIT、需要注意的事项等，比起 Lua 和 LuaJIT 的文档，着实高了好几个数量级。

好的，回到`table.nkeys`函数上，它的命名可能会让你迷惑，不过，它实际上是获取 table 长度的函数，返回的是 table 的元素个数，包括数组和哈希部分的元素。因此，我们可以用它来替代 `table.getn`，比如下面这样来用：

```
local nkeys = require &quot;table.nkeys&quot;

print(nkeys({}))  -- 0
print(nkeys({ &quot;a&quot;, nil, &quot;b&quot; }))  -- 2
print(nkeys({ dog = 3, cat = 4, bird = nil }))  -- 2
print(nkeys({ &quot;a&quot;, dog = 3, cat = 4 }))  -- 3

```

## 元表

讲完了table函数，我们再来看下由 `table` 引申出来的 `元表`（metatable）。元表是 Lua 中独有的概念，在实际项目中的使用非常广泛。不夸张地说，在几乎所有的 `lua-resty-*` 库中，你都能看到它的身影。

元表的表现行为类似于操作符重载，比如我们可以重载 `__add`，来计算两个 Lua 数组的并集；或者重载 `__tostring`，来定义转换为字符串的函数。

而Lua 提供了两个处理元表的函数：

- 第一个是`setmetatable(table, metatable)`, 用于为一个 table 设置元表；
- 第二个是`getmetatable(table)`，用于获取 table 的元表。

介绍了这么半天，你可能更关心它的作用，我们接着就来看下元表具体有什么用处。下面是一段真实项目里的代码：

```
$ resty -e ' local version = {
  major = 1,
  minor = 1,
  patch = 1
  }
version = setmetatable(version, {
    __tostring = function(t)
      return string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch)
    end
  })
  print(tostring(version))
'

```

我们首先定义了一个 名为 `version`的table ，你可以看到，这段代码的目的，是想把 `version` 中的版本号打印出来。但是，我们并不能直接打印 `version`，你可以试着操作一下，就会发现，直接打印的话，只会输出这个 table 的地址。

```
print(tostring(version))

```

所以，我们需要自定义这个 table 的字符串转换函数，也就是 `__tostring`，到这一步也就是元表的用武之地了。我们用 `setmetatable` ，重新设置 `version` 这个 table 的 `__tostring` 方法，就可以打印出版本号: 1.1.1。

其实，除了 `__tostring` 之外，在实际项目中，我们还经常重载元表中的以下两个元方法（metamethod）。

**其中一个是`__index`**。我们在 table 中查找一个元素时，首先会直接从 table 中查询，如果没有找到，就继续到元表的 `__index` 中查询。

比如下面这个例子，我们把 `patch` 从 `version` 这个 table 中去掉：

```
$ resty -e ' local version = {
  major = 1,
  minor = 1
  }
version = setmetatable(version, {
     __index = function(t, key)
         if key == &quot;patch&quot; then
             return 2
         end
     end,
     __tostring = function(t)
      return string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch)
    end
  })
  print(tostring(version))
'

```

这样的话，`t.patch` 其实获取不到值，那么就会走到 `__index` 这个函数中，结果就会打印出 1.1.2。

事实上，`__index` 不仅可以是一个函数，也可以是一个 table。你试着运行下面这段代码，就会看到，它们实现的效果是一样的。

```
$ resty -e ' local version = {
  major = 1,
  minor = 1
  }
version = setmetatable(version, {
     __index = {patch = 2},
     __tostring = function(t)
      return string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch)
    end
  })
  print(tostring(version))
'

```

**另一个元方法则是`__call`**。它类似于仿函数，可以让 table 被调用。

我们还是基于上面打印版本号的代码来做修改，看看如何调用一个 table：

```
$ resty -e '
local version = {
  major = 1,
  minor = 1,
  patch = 1
  }

local function print_version(t)
     print(string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch))
end

version = setmetatable(version,
     {__call = print_version})

  version()
'

```

这段代码中，我们使用 `setmetatable`，给 `version` 这个 table 增加了元表，而里面的 `__call` 元方法指向了函数 `print_version` 。那么，如果我们尝试把 `version` 当作函数调用，这里就会执行函数 `print_version`。

而 `getmetatable` 是和 `setmetatable` 配对的操作，可以获取到已经设置的元表，比如下面这段代码：

```
$ resty -e ' local version = {
  major = 1,
  minor = 1
  }
version = setmetatable(version, {
     __index = {patch = 2},
     __tostring = function(t)
      return string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch)
    end
  })
  print(getmetatable(version).__index.patch)
'

```

自然，除了今天讲到的这三个元方法外，还有一些不经常使用的元方法，你可以在遇到的时候再去查阅[文档](http://lua-users.org/wiki/MetamethodsTutorial)了解。

## 面向对象

最后我们来聊聊面向对象。你可能知道，Lua 并不是一个面向对象（Object Orientation）的语言，但我们可以使用 metatable 来实现 OO。

我们来看一个实际的例子。[lua-resty-mysql](https://github.com/openresty/lua-resty-mysql/blob/master/lib/resty/mysql.lua)  是 OpenResty 官方的 MySQL 客户端，里面就使用元表**模拟**了类和类方法，它的使用方式如下所示：

```
$ resty -e 'local mysql = require &quot;resty.mysql&quot; -- 先引用 lua-resty 库
local db, err = mysql:new() -- 新建一个类的实例
db:set_timeout(1000) -- 调用类的方法'

```

你可以直接用 `resty` 命令行来执行上述代码。这几行代码很好理解，唯一可能给你造成困扰的是：

**在调用类方法的时候，为什么是冒号而不是点号呢？**

其实，在这里冒号和点号都是可以的，`db:set_timeout(1000)` 和 `db.set_timeout(db, 1000)` 是完全等价的。冒号是 Lua 中的一个语法糖，可以省略掉函数的第一个参数 `self`。

众所周知，源码面前没有秘密，让我们来看看上述几行代码所对应的具体实现，以便你更好理解，如何用元表来模拟面向对象：

```
local _M = { _VERSION = '0.21' } -- 使用 table 模拟类
local mt = { __index = _M } -- mt 即 metatable 的缩写，__index 指向类自身

-- 类的构造函数
function _M.new(self) 
     local sock, err = tcp()
     if not sock then
         return nil, err
     end
     return setmetatable({ sock = sock }, mt) -- 使用 table 和 metatable 模拟类的实例
end
 
-- 类的成员函数
 function _M.set_timeout(self, timeout) -- 使用 self 参数，获取要操作的类的实例
     local sock = self.sock
     if not sock then
        return nil, &quot;not initialized&quot;
     end

    return sock:settimeout(timeout)
end

```

你可以看到，`_M` 这个 table 模拟了一个类，初始化时，它只有 `_VERSION` 这一个成员变量，并在随后定义了 `_M.set_timeout` 等成员函数。在 `_M.new(self)` 这个构造函数中，我们返回了一个 table，这个 table 的元表就是 `mt`，而 `mt` 的 `__index` 元方法指向了 `_M`，这样，返回的这个 table 就模拟了类 `_M` 的实例。

## 写在最后

好的，到这里，今天的主要内容就结束了。事实上，table 和 metatable 会大量地用在 OpenResty 的 `lua-resty-*` 库以及基于 OpenResty 的开源项目中，我希望通过这节课的学习，可以让你更容易地读懂这些源代码。

自然，除了 table 外，Lua 中还有其他一些常用的函数，我们下节课再一起来学习。

最后，我想给你留一个思考题。为什么 `lua-resty-mysql` 库要模拟 OO 来做一层封装呢？欢迎在留言区一起讨论这个问题，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。


