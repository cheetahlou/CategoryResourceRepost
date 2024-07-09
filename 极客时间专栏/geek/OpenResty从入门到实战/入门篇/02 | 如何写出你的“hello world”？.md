<audio id="audio" title="02 | 如何写出你的“hello world”？" controls="" preload="none"><source id="mp3" src="https://static001.geekbang.org/resource/audio/0d/0b/0df176a972a52a2555d75a11d2152e0b.mp3"></audio>

你好，我是温铭。今天起，就要开始我们的正式学习之旅。

每当我们开始学习一个新的开发语言或者平台，都会从最简单的`hello world`开始，OpenResty 也不例外。让我们先跳过安装的步骤，直接看下，最简单的 OpenResty 程序是怎么编写和运行的：

```
$ resty -e &quot;ngx.say('hello world')&quot;
hello world

```

这应该是你见过的最简单的那种 hello world 代码写法，和 Python 类似：

```
$ python -c 'print(&quot;hello world&quot;)'
hello world

```

这背后其实是 OpenResty 哲学的一种体现，代码要足够简洁，也好让你打消“从入门到放弃“的念头。我们今天的内容，就专门围绕着这行代码来展开聊一聊。

上一节我们讲过，OpenResty 是基于 NGINX 的。那你现在是不是有一个疑问：为什么这里看不到 NGINX 的影子？别着急，我们加一行代码，看看 `resty`背后真正运行的是什么：

```
resty -e &quot;ngx.say('hello world'); ngx.sleep(10)&quot; &amp;

```

我们加了一行 sleep 休眠的代码，让 resty 运行的程序打印出字符串后，并不退出。这样，我们就有机会一探究竟：

```
$ ps -ef | grep nginx
501 25468 25462   0  7:24下午 ttys000    0:00.01 /usr/local/Cellar/openresty/''1.13.6.2/nginx/sbin/nginx -p /tmp/resty_AfNwigQVOB/ -c conf/nginx.conf

```

终于看了熟悉的 NGINX 进程。看来，`resty` 本质上是启动了一个 NGINX 服务，那么`resty` 又是一个什么程序呢？我先卖个关子，咱后面再讲。

你的机器上可能还没有安装 OpenResty，所以，接下来，我们先回到开头跳过的安装步骤，把 OpenResty 安装完成后再继续。

## OpenResty 的安装

和其他的开源软件一样，OpenResty 的安装有多种方法，比如使用操作系统的包管理器、源码编译或者 docker 镜像。我推荐你优先使用 yum、apt-get、brew 这类包管理系统，来安装 OpenResty。这里我们使用 Mac 系统来做示例：

```
brew tap openresty/brew
brew install openresty

```

使用其他操作系统也是类似的，先要在包管理器中添加 OpenResty 的仓库地址，然后用包管理工具来安装。具体步骤，你可以参考[官方文档](https://openresty.org/en/linux-packages.html)。

不过，这看似简单的安装背后，其实有两个问题：

<li>
为什么我不推荐使用源码来安装呢？
</li>
<li>
为什么不能直接从操作系统的官方仓库安装，而是需要先设置另外一个仓库地址？
</li>

对于这两个问题，你不妨先自己想一想。

这里我想补充一句。在这门课程里面，我会在表象背后提出很多的“为什么”，希望你可以一边学新东西一边思考，结果是否正确并不重要。独立思考在技术领域也是稀缺的，由于每个人技术领域和深度的不同，在任何课程中老师都会不可避免地带有个人观点以及知识的错漏。只有在学习过程中多问几个为什么，融会贯通，才能逐渐形成自己的技术体系。

很多工程师都有源码的情节，多年前的我也是一样。在使用一个开源项目的时候，我总是希望能够自己手工从源码开始 configure 和 make，并修改一些编译参数，感觉这样做才能最适合这台机器的环境，才能把性能发挥到极致。

但现实并非如此，每次源码编译，我都会遇到各种诡异的环境问题，磕磕绊绊才能安装好。现在我想明白了，我们的最初目的其实是用开源项目来解决业务需求，不应该浪费时间和环境鏖战，更何况包管理器和容器技术，正是为了帮我们解决这些问题。

言归正传，给你说说我的看法。使用 OpenResty 源码安装，不仅仅步骤繁琐，需要自行解决 PCRE、OpenSSL 等外部依赖，而且还需要手工对 OpenSSL 打上对应版本的补丁。不然就会在处理 SSL session 时，带来功能上的缺失，比如像`ngx.sleep`这类会导致 yield 的 Lua API 就没法使用。这部分内容如果你还想深入了解，可以参考[[官方文档](https://github.com/openresty/lua-nginx-module#ssl_session_fetch_by_lua_block)]来获取更详细的信息。

从 OpenResty 自己维护的 OpenSSL [[打包脚本](https://github.com/openresty/openresty-packaging/blob/master/rpm/SPECS/openresty-openssl.spec)]中，就可以看到这些补丁。而在 OpenResty 升级 OpenSSL 版本时，都需要重新生成对应的补丁，并进行完整的回归测试。

```
Source0: https://www.openssl.org/source/openssl-%{version}.tar.gz

Patch0: https://raw.githubusercontent.com/openresty/openresty/master/patches/openssl-1.1.0d-sess_set_get_cb_yield.patch
Patch1: https://raw.githubusercontent.com/openresty/openresty/master/patches/openssl-1.1.0j-parallel_build_fix.patch

```

同时，我们可以看下 OpenResty 在 CentOS 中的[[打包脚本]](https://github.com/openresty/openresty-packaging/blob/master/rpm/SPECS/openresty.spec)，看看是否还有其他隐藏的点：

```
BuildRequires: perl-File-Temp
BuildRequires: gcc, make, perl, systemtap-sdt-devel
BuildRequires: openresty-zlib-devel &gt;= 1.2.11-3
BuildRequires: openresty-openssl-devel &gt;= 1.1.0h-1
BuildRequires: openresty-pcre-devel &gt;= 8.42-1
Requires: openresty-zlib &gt;= 1.2.11-3
Requires: openresty-openssl &gt;= 1.1.0h-1
Requires: openresty-pcre &gt;= 8.42-1

```

从这里可以看出，OpenResty 不仅维护了自己的 OpenSSL 版本，还维护了自己的 zlib 和 PCRE 版本。不过后面两个只是调整了编译参数，并没有维护自己的补丁。

所以，综合这些因素，我不推荐你自行源码编译 OpenResty，除非你已经很清楚这些细节。

为什么不推荐源码安装，你现在应该已经很清楚了。其实我们在回答第一个问题时，也顺带回答了第二个问题：为什么不能直接从操作系统的官方仓库安装，而是需要先设置另外一个仓库地址？

这是因为，官方仓库不愿意接受第三方维护的 OpenSSL、PCRE 和 zlib 包，这会导致其他使用者的困惑，不知道选用哪一个合适。另一方面，OpenResty 又需要指定版本的 OpenSSL、PCRE 库才能正常运行，而系统默认自带的版本都比较旧。

## OpenResty CLI

安装完 OpenResty 后，默认就已经把 OpenResty 的 CLI：`resty` 安装好了。`resty`是个 1000 多行的 Perl 脚本，之前我们提到过，OpenResty 的周边工具都是 Perl 编写的，这个是由 OpenResty 作者的技术偏好决定的。

```
$ which resty
/usr/local/bin/resty
$ head -n 1 /usr/local/bin/resty
 #!/usr/bin/env perl

```

`resty` 的功能很强大，想了解完整的列表，你可以查看`resty -h`或者[[官方文档](https://github.com/openresty/resty-cli)]。下面，我挑两个有意思的功能介绍一下。

```
$ resty --shdict='dogs 1m' -e 'local dict = ngx.shared.dogs
                               dict:set(&quot;Tom&quot;, 56)
                               print(dict:get(&quot;Tom&quot;))'
56


```

先来看第一个例子。这个示例结合了 NGINX 配置和 Lua 代码，一起完成了一个共享内存字典的设置和查询。`dogs 1m` 是 NGINX 的一段配置，声明了一个共享内存空间，名字是 dogs，大小是 1m；在 Lua 代码中用字典的方式使用共享内存。另外还有`--http-include` 和 `--main-include`来设置 NGINX 配置文件。所以，上面的例子也可以写为：

```
resty --http-conf 'lua_shared_dict dogs 1m;' -e 'local dict = ngx.shared.dogs
                               dict:set(&quot;Tom&quot;, 56)
                               print(dict:get(&quot;Tom&quot;))'

```

OpenResty 世界中常用的调试工具，比如`gdb`、`valgrind`、`sysetmtap`和`Mozilla rr` ，也可以和 `resty` 一起配合使用，方便你平时的开发和测试。它们分别对应着 `resty` 不同的指令，内部的实现其实很简单，就是多套了一层命令行调用。我们以 valgrind 为例：

```
$ resty --valgrind  -e &quot;ngx.say('hello world'); &quot;
ERROR: failed to run command &quot;valgrind /usr/local/Cellar/openresty/1.13.6.2/nginx/sbin/nginx -p /tmp/resty_hTFRsFBhVl/ -c conf/nginx.conf&quot;: No such file or directory

```

在后面调试、测试和性能分析的章节，会涉及到这些工具的使用。它们不仅适用于 OpenResty 世界，也是服务端的通用工具，让我们循序渐进地来学习吧。

## 更正式的 hello world

最开始我们使用`resty`写的第一个 OpenResty 程序，没有 master 进程，也不会监听端口。下面，让我们写一个更正式的 hello world。

写出这样的 OpenResty 程序并不简单，你至少需要三步才能完成：

<li>
创建工作目录；
</li>
<li>
修改 NGINX 的配置文件，把 Lua 代码嵌入其中；
</li>
<li>
启动 OpenResty 服务。
</li>

我们先来创建工作目录。

```
mkdir geektime
cd geektime
mkdir logs/ conf/

```

下面是一个最简化的 `nginx.conf`，在根目录下新增 OpenResty 的`content_by_lua`指令，里面嵌入了`ngx.say`的代码：

```
events {
    worker_connections 1024;
}

http {
    server {
        listen 8080;
        location / {
            content_by_lua '
                ngx.say(&quot;hello, world&quot;)
            ';
        }
    }
}

```

请先确认下，是否已经把`openresty`加入到`PATH`环境中；然后，启动 OpenResty 服务就可以了：

```
openresty -p `pwd` -c conf/nginx.conf

```

没有报错的话，OpenResty 的服务就已经成功启动了。你可以打开浏览器，或者使用 curl 命令，来查看结果的返回：

```
$ curl -i 127.0.0.1:8080
HTTP/1.1 200 OK
Server: openresty/1.13.6.2
Content-Type: text/plain
Transfer-Encoding: chunked
Connection: keep-alive

hello, world

```

到这里，恭喜你，一个真正的 OpenResty 程序就完成了。

## 总结

让我们回顾下今天讲的内容。我们通过一行简单的 `hello, world` 代码，延展到OpenResty 的安装和 CLI，并在最后启动了 OpenResty 进程，运行了一个真正的后端程序。

其中， `resty` 是我们后面会频繁使用到的命令行工具，课程中的演示代码都是用它来运行的，而不是启动后台的 OpenResty 服务。

更为重要的是，OpenResty 的背后隐藏了非常多的文化和技术细节，它就像漂浮在海面上的一座冰山。我希望能够通过这门课程，给你展示更全面、更立体的 OpenResty，而不仅仅是它对外暴露出来的 API。

## 思考

最后，我给你留一个作业题。我们现在的做法，是把 Lua 代码写在 NGINX 配置文件中。不过，如果代码越来越多，那代码的可读性和可维护性就无法保证了。

你有什么方法来解决这个问题吗？欢迎留言和我分享，也欢迎你把这篇文章转发给你的同事、朋友。


