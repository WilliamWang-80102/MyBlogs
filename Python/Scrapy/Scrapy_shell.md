# Scrapy shell

Scrapy shell允许我们以交互式的方式来尝试用xpath或者css来提取网站信息，而不必每次运行爬虫程序来进行调试。

## shell配置

Linux下安装IPython以辅助我们在命令行中编写python代码（支持语法高亮和自动补齐）
安装了IPython后，Scrapy shell将自动用启动IPython。
你也可以在配置文件`scrapy.cfg`中设置使用`ipyhon`或者标准的`python`命令行：

```text
[settings]
shell = ipython
```

## 运行shell程序

运行shell的指令是：

```shell
scrapy shell <url>
```

其中url是你想要爬取的网址，注意这个地址也可以是当前文件系统的（相对|绝对）地址，但是在使用相对地址时，请在前面加上前缀`./`或者`../`。因为scrapy优先认为Url是基于http协议。

## 使用shell

**常用快捷方法**
`shelp` - 打印帮助信息，包括可用的对象和快捷方法
`fetch(url,[redirect=True])` - 根据url获得一个新的当前响应对象，可以通过设置`redirect=False`来阻止`HTTP 3xx`（重定向）。
`fetch(request)` - 根据提供的请求获得一个新的响应对象
`view(response)` - 从当前浏览器打开响应页面

## shell中可用的对象

Scrapy shell会自动地根据当前访问的URL生成一些内置对象，比如Response对象和Selector对象（针对HTML文档和XML文档）

- crawler
- spider
- request - 该标志引用一个Request类型的对象，对应于上一次获得的页面所发送的请求。使用`replace()`方法或者`fetch()`方法将修改这个变量的内容。
- response - 该标志引用一个Response类型的对象，对应于上一次请求的响应内容。
- settings - 对应于当前Scrapy Setting的内容。

## 使用实例

详见[官方文档](https://docs.scrapy.org/en/latest/topics/shell.html#example-of-shell-session)
