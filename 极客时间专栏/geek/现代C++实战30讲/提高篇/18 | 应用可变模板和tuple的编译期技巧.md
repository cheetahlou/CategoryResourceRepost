<audio id="audio" title="18 | 应用可变模板和tuple的编译期技巧" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/76/e0/76181e927a84617d78a7ca04e16195e0.mp3"></audio>

你好，我是吴咏炜。

今天我们讲一个特殊的专题，如何使用可变模板和 tuple 来完成一些常见的功能，尤其是编译期计算。

## 可变模板

可变模板 [1] 是 C++11 引入的一项新功能，使我们可以在模板参数里表达不定个数和类型的参数。从实际的角度，它有两个明显的用途：

- 用于在通用工具模板中转发参数到另外一个函数
- 用于在递归的模板中表达通用的情况（另外会有至少一个模板特化来表达边界情况）

我们下面就来分开讨论一下。

### 转发用法

以标准库里的 `make_unique` 为例，它的定义差不多是下面这个样子：

```
template &lt;typename T,
          typename... Args&gt;
inline unique_ptr&lt;T&gt;
make_unique(Args&amp;&amp;... args)
{
  return unique_ptr&lt;T&gt;(
    new T(forward&lt;Args&gt;(args)...));
}

```

这样，它就可以把传递给自己的全部参数转发到模板参数类的构造函数上去。注意，在这种情况下，我们通常会使用 `std::forward`，确保参数转发时仍然保持正确的左值或右值引用类型。

稍微解释一下上面三处出现的 `...`：

- `typename... Args` 声明了一系列的类型——`class...` 或 `typename...` 表示后面的标识符代表了一系列的类型。
- `Args&amp;&amp;... args` 声明了一系列的形参 `args`，其类型是 `Args&amp;&amp;`。
- `forward&lt;Args&gt;(args)...` 会在编译时实际逐项展开 `Args` 和 `args` ，参数有多少项，展开后就是多少项。

举一个例子，如果我们需要在堆上传递一个 `vector&lt;int&gt;`，假设我们希望初始构造的大小为 100，每个元素都是 `1`，那我们可以这样写：

```
make_unique&lt;vector&lt;int&gt;&gt;(100, 1)

```

模板实例化之后，会得到相当于下面的代码：

```
template &lt;&gt;
inline unique_ptr&lt;vector&lt;int&gt;&gt;
make_unique(int&amp;&amp; arg1, int&amp;&amp; arg2)
{
  return unique_ptr&lt;vector&lt;int&gt;&gt;(
    new vector&lt;int&gt;(
      forward&lt;int&gt;(arg1),
      forward&lt;int&gt;(arg2)));
}

```

如前所述，`forward&lt;Args&gt;(args)...` 为每一项可变模板参数都以同样的形式展开。项数也允许为零，那样，我们在调用构造函数时也同样没有任何参数。

### 递归用法

我们也可以用可变模板来实现编译期递归。下面就是个小例子：

```
template &lt;typename T&gt;
constexpr auto sum(T x)
{
  return x;
}

template &lt;typename T1, typename T2,
          typename... Targ&gt;
constexpr auto sum(T1 x, T2 y,
                   Targ... args)
{
  return sum(x + y, args...);
}

```

在上面的定义里，如果 `sum` 得到的参数只有一个，会走到上面那个重载。如果有两个或更多参数，编译器就会选择下面那个重载，执行一次加法，随后你的参数数量就少了一个，因而递归总会终止到上面那个重载，结束计算。

要使用上面这个模板，我们就可以写出像下面这样的函数调用：

```
auto result = sum(1, 2, 3.5, x);

```

模板会这样依次展开：

```
sum(1 + 2, 3.5, x)
sum(3 + 3.5, x)
sum(6.5 + x)
6.5 + x

```

注意我们都不必使用相同的数据类型：只要这些数据之间可以应用 `+`，它们的类型无关紧要……

再看另一个复杂些的例子，函数的组合 [2]。如果我们有函数 $f$ 和 函数 $g$，要得到函数的联用 $g \circ f$，其满足：

$$<br>
(g \circ f)(x) = g(f(x))<br>
$$

我们能不能用一种非常简单的方式，写不包含变量 $x$ 的表达式来表示函数组合呢？答案是肯定的。

跟上面类似，我们需要写出递归的终结情况，单个函数的“组合”：

```
template &lt;typename F&gt;
auto compose(F f)
{
  return [f](auto&amp;&amp;... x) {
    return f(
      forward&lt;decltype(x)&gt;(x)...);
  };
}

```

上面我们仅返回一个泛型 lambda 表达式，保证参数可以转发到 `f`。记得我们在[[第 16 讲]](https://time.geekbang.org/column/article/184018) 讲过泛型 lambda 表达式，本质上就是一个模板，所以我们按转发用法的可变模板来理解上面的 `...` 部分就对了。

下面是正常有组合的情况：

```
template &lt;typename F,
          typename... Args&gt;
auto compose(F f, Args... other)
{
  return [f,
          other...](auto&amp;&amp;... x) {
    return f(compose(other...)(
      forward&lt;decltype(x)&gt;(x)...));
  };
}

```

在这个模板里，我们返回一个 lambda 表达式，然后用 `f` 捕捉第一个函数对象，用 `args...` 捕捉后面的函数对象。我们用 `args...` 继续组合后面的部分，然后把结果传到 `f` 里面。

上面的模板定义我实际上已经有所简化，没有保持值类别。完整的包含完美转发的版本，请看参考资料 [3] 中的 functional.h 实现。

下面我们来试验一下使用这个 `compose` 函数。我们先写一个对输入范围中每一项都进行平方的函数对象：

```
auto square_list =
  [](auto&amp;&amp; container) {
    return fmap(
      [](int x) { return x * x; },
      container);
  };

```

我们使用了[[第 13 讲]](https://time.geekbang.org/column/article/181608) 中给出的 `fmap`，而不是标准库里的 `transform`，是因为后者接口非函数式，无法组合——它要求参数给出输出位置的迭代器，会修改迭代器指向的内容，返回结果也只是单个的迭代器；函数式的接口则期望不修改参数的内容，结果完全在返回值中。

我们这儿用了泛型 lambda 表达式，是因为组合的时候不能使用模板，只能是函数对象或函数（指针）——如果我们定义一个 `square_list` 模板的话，组合时还得显式实例化才行（写成 `square_list&lt;const vector&lt;int&gt;&amp;&gt;` 的样子），很不方便。

我们再写一个求和的函数对象：

```
auto sum_list =
  [](auto&amp;&amp; container) {
    return accumulate(
      container.begin(),
      container.end(), 0);
  };

```

那先平方再求和，就可以这样简单定义了：

```
auto squared_sum =
  compose(sum_list, square_list);

```

我们可以验证这个定义是可以工作的：

```
vector v{1, 2, 3, 4, 5};
cout &lt;&lt; squared_sum(v) &lt;&lt; endl;

```

我们会得到：

> 
`55`


## tuple

上面的写法虽然看起来还不错，但实际上有个缺陷：被 compose 的函数除了第一个（最右边的），其他的函数只能接收一个参数。要想进一步推进类似的技巧，我们得首先解决这个问题。

在 C++ 里，要通用地用一个变量来表达多个值，那就得看多元组——`tuple` 模板了 [4]。`tuple` 算是 C++98 里的 `pair` 类型的一般化，可以表达任意多个固定数量、固定类型的值的组合。下面这段代码约略地展示了其基本用法：

```
#include &lt;algorithm&gt;
#include &lt;iostream&gt;
#include &lt;string&gt;
#include &lt;tuple&gt;
#include &lt;vector&gt;

using namespace std;

// 整数、字符串、字符串的三元组
using num_tuple =
  tuple&lt;int, string, string&gt;;

ostream&amp;
operator&lt;&lt;(ostream&amp; os,
           const num_tuple&amp; value)
{
  os &lt;&lt; get&lt;0&gt;(value) &lt;&lt; ','
     &lt;&lt; get&lt;1&gt;(value) &lt;&lt; ','
     &lt;&lt; get&lt;2&gt;(value);
  return os;
}

int main()
{
  // 阿拉伯数字、英文、法文
  vector&lt;num_tuple&gt; vn{
    {1, "one",   "un"},
    {2, "two",   "deux"},
    {3, "three", "trois"},
    {4, "four",  "quatre"}};
  // 修改第 0 项的法文
  get&lt;2&gt;(vn[0]) = "une";
  // 按法文进行排序
  sort(vn.begin(), vn.end(),
       [](auto&amp;&amp; x, auto&amp;&amp; y) {
         return get&lt;2&gt;(x) &lt;
                get&lt;2&gt;(y);
       });
  // 输出内容
  for (auto&amp;&amp; value : vn) {
    cout &lt;&lt; value &lt;&lt; endl;
  }
  // 输出多元组项数
  constexpr auto size = \
    tuple_size_v&lt;num_tuple&gt;;
  cout &lt;&lt; "Tuple size is " &lt;&lt; size &lt;&lt; endl;
}

```

输出是：

> 
<p>`2,two,deux`<br>
`4,four,quatre`<br>
`3,three,trois`<br>
`1,one,une`<br>
`Tuple size is 3`</p>


我们可以看到：

- `tuple` 的成员数量由尖括号里写的类型数量决定。
- 可以使用 `get` 函数对 `tuple` 的内容进行读和写。（当一个类型在 `tuple` 中出现正好一次时，我们也可以传类型取内容，即，对我们上面的三元组，`get&lt;int&gt;` 是合法的，`get&lt;string&gt;` 则不是。）
- 可以用 `tuple_size_v` （在编译期）取得多元组里面的项数。

如果我们要用一个三项的 `tuple` 去调用一个函数，我们可以写类似这样的代码：

```
template &lt;class F, class Tuple&gt;
constexpr decltype(auto) apply(
  F&amp;&amp; f, Tuple&amp;&amp; t)
{
  return f(
    get&lt;0&gt;(forward&lt;Tuple&gt;(t)),
    get&lt;1&gt;(forward&lt;Tuple&gt;(t)),
    get&lt;2&gt;(forward&lt;Tuple&gt;(t)));
}

```

这似乎已经挺接近我们需要的形式了，但实际调用函数的参数项数会变啊……

我们已经有了参数的项数（使用 `tuple_size_v`），所以我们下面要做的是生成从 0 到项数减一之间的整数序列。标准库里已经定义了相关的工具，我们需要的就是其中的 `make_index_sequence` [5]，其简化实现如下所示：

```
template &lt;class T, T... Ints&gt;
struct integer_sequence {};

template &lt;size_t... Ints&gt;
using index_sequence =
  integer_sequence&lt;size_t, Ints...&gt;;

template &lt;size_t N, size_t... Ints&gt;
struct index_sequence_helper {
  typedef
    typename index_sequence_helper&lt;
      N - 1, N - 1, Ints...&gt;::type
      type;
};

template &lt;size_t... Ints&gt;
struct index_sequence_helper&lt;
  0, Ints...&gt; {
  typedef index_sequence&lt;Ints...&gt;
    type;
};

template &lt;size_t N&gt;
using make_index_sequence =
  typename index_sequence_helper&lt;
    N&gt;::type;

```

正如一般的模板代码，它看起来还是有点绕的。其要点是，如果我们给出 `make_index_sequence&lt;N&gt;`，则结果是 `integer_sequence&lt;size_t, 0, 1, 2, …, N - 1&gt;`（一下子想不清楚的话，可以拿纸笔来模拟一下模板的展开过程）。而有了这样一个模板的帮助之后，我们就可以写出下面这样的函数（同样，这是标准库里的 `apply` 函数模板 [6] 的简化版本）：

```
template &lt;class F, class Tuple,
          size_t... I&gt;
constexpr decltype(auto)
apply_impl(F&amp;&amp; f, Tuple&amp;&amp; t,
           index_sequence&lt;I...&gt;)
{
  return f(
    get&lt;I&gt;(forward&lt;Tuple&gt;(t))...);
}

template &lt;class F, class Tuple&gt;
constexpr decltype(auto)
apply(F&amp;&amp; f, Tuple&amp;&amp; t)
{
  return apply_impl(
    forward&lt;F&gt;(f),
    forward&lt;Tuple&gt;(t),
    make_index_sequence&lt;
      tuple_size_v&lt;
        remove_reference_t&lt;
          Tuple&gt;&gt;&gt;{});
}

```

我们如果有一个三元组 `t`，类型为 `tuple&lt;int, string, string&gt;`，去 `apply` 到一个函数 `f`，展开后我们得到 `apply_impl(f, t, index_sequence&lt;0, 1, 2&gt;{})`，再展开后我们就得到了上面那个有 `get&lt;0&gt;`、`get&lt;1&gt;`、`get&lt;2&gt;` 的函数调用形式。换句话说，我们利用一个计数序列的类型，可以在编译时展开 `tuple` 里的各个成员，并用来调用函数。

## 数值预算

上面的代码有点复杂，而且似乎并没有完成什么很重要的功能。我们下面看一个源自实际项目的例子。需求是，我们希望快速地计算一串二进制数中 1 比特的数量。举个例子，如果我们有十进制的 31 和 254，转换成二进制是 00011111 和 11111110，那我们应该得到 5 + 7 = 12。

显然，每个数字临时去数肯定会慢，我们应该预先把每个字节的 256 种情况记录下来。因而，如何得到这些计数值是个问题。在没有编译期编程时，我们似乎只能用另外一个程序先行计算，然后把结果填进去——这就很不方便很不灵活了。有了编译期编程，我们就不用写死，而让编译器在编译时帮我们计算数值。

利用 constexpr 函数，我们计算单个数值完全没有问题。快速定义如下：

```
constexpr int
count_bits(unsigned char value)
{
  if (value == 0) {
    return 0;
  } else {
    return (value &amp; 1) +
           count_bits(value &gt;&gt; 1);
  }
}

```

可 256 个，总不见得把计算语句写上 256 遍吧？这就需要用到我们上面讲到的 `index_sequence` 了。我们定义一个模板，它的参数是一个序列，在初始化时这个模板会对参数里的每一项计算比特数，并放到数组成员里。

```
template &lt;size_t... V&gt;
struct bit_count_t {
  unsigned char
    count[sizeof...(V)] = {
      static_cast&lt;unsigned char&gt;(
        count_bits(V))...};
};

```

注意上面用 `sizeof...(V)` 可以获得参数的个数（在 `tuple_size_v` 的实现里实际也用到它了）。如果我们模板参数传 `0, 1, 2, 3`，结果里面就会有个含 4 项元素的数组，数值分别是对 0、1、2、3 的比特计数。

然后，我们当然就可以利用 `make_index_sequence` 来展开计算了，想产生几项就可以产生几项。不过，要注意到 `make_index_sequence` 的结果是个类型，不能直接用在 `bit_count_t` 的构造中。我们需要用模板匹配来中转一下：

```
template &lt;size_t... V&gt;
constexpr bit_count_t&lt;V...&gt;
get_bit_count(index_sequence&lt;V...&gt;)
{
  return bit_count_t&lt;V...&gt;();
}

auto bit_count = get_bit_count(
  make_index_sequence&lt;256&gt;());

```

得到 `bit_count` 后，我们要计算一个序列里的比特数就只是轻松查表相加了，此处不再赘述。

## 内容小结

今天我们讨论了在编译期处理不确定数量的参数和类型的基本语言特性，可变模板，以及可以操控可变模板的重要工具——`tuple` 和 `index_sequence`。用好这些工具，可以让我们轻松地完成一些编译期计算的工作。

## 课后思考

请考虑一下：

1. 我展示了 `compose` 带一个或更多参数的情况。你觉得 `compose` 不带任何参数该如何定义？它有意义吗？
1. 有没有可能不用 `index_sequence` 来初始化 `bit_count`？如果行，应该如何实现？
1. 作为一个挑战，你能自行实现出 `make_integer_sequence` 吗？

期待你的答案。

## 参考资料

[1] cppreference.com, “Parameter pack”. [https://en.cppreference.com/w/cpp/language/parameter_pack](https://en.cppreference.com/w/cpp/language/parameter_pack) 

[1a] cppreference.com, “形参包”. [https://zh.cppreference.com/w/cpp/language/parameter_pack](https://zh.cppreference.com/w/cpp/language/parameter_pack) 

[2] Wikipedia, “Function composition”. [https://en.wikipedia.org/wiki/Function_composition](https://en.wikipedia.org/wiki/Function_composition) 

[2a] 维基百科, “复合函数”. [https://zh.wikipedia.org/zh-cn/复合函数](https://zh.wikipedia.org/zh-cn/%E5%A4%8D%E5%90%88%E5%87%BD%E6%95%B0) 

[3] 吴咏炜, nvwa. [https://github.com/adah1972/nvwa](https://github.com/adah1972/nvwa) 

[4] cppreference.com, “std::tuple”. [https://en.cppreference.com/w/cpp/utility/tuple](https://en.cppreference.com/w/cpp/utility/tuple) 

[4a] cppreference.com, “std::tuple”. [https://zh.cppreference.com/w/cpp/utility/tuple](https://zh.cppreference.com/w/cpp/utility/tuple) 

[5] cppreference.com, “std::integer_sequence”. [https://en.cppreference.com/w/cpp/utility/integer_sequence](https://en.cppreference.com/w/cpp/utility/integer_sequence) 

[5a] cppreference.com, “std::integer_sequence”. [https://zh.cppreference.com/w/cpp/utility/integer_sequence](https://zh.cppreference.com/w/cpp/utility/integer_sequence) 

[6] cppreference.com, “std::apply”. [https://en.cppreference.com/w/cpp/utility/apply](https://en.cppreference.com/w/cpp/utility/apply) 

[6a] cppreference.com, “std::apply”. [https://zh.cppreference.com/w/cpp/utility/apply](https://zh.cppreference.com/w/cpp/utility/apply) 
