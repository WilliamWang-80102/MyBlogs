# scrapy入门教程

## 创建一个scrapy项目

首先输入如下指令，创建一个scrapy项目

```shell
scrapy startproject tutorial
```

该命令会在当前文件夹下，创建一个scrapy的项目的文件夹，文件结构如下：

```text
tutorial/
    scrapy.cfg            # deploy configuration file

    tutorial/             # project's Python module, you'll import your code from here
        __init__.py

        items.py          # project items definition file

        middlewares.py    # project middlewares file

        pipelines.py      # project pipelines file

        settings.py       # project settings file

        spiders/          # a directory where you'll later put your spiders
            __init__.py
```

## 编写第一个爬虫

我们在`tutorial/spiders/`下创建`quotes_spider.py`文件，这个里面包含我们的第一个爬虫类。

```python
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"

    def start_requests(self):
        urls = [
            'http://quotes.toscrape.com/page/1/',
            'http://quotes.toscrape.com/page/2/',
        ]
        for url in urls:
            yield scrapy.Request(url=url, callback=self.parse)

    def parse(self, response):
        page = response.url.split("/")[-2]
        filename = 'quotes-%s.html' % page
        with open(filename, 'wb') as f:
            f.write(response.body)
        self.log('Saved file %s' % filename)
```

我们定义的爬虫类必须要是scrapy.Spider类的子类（本例中是QuotesSpider类），它需要声明待请求的url。可选地，我们还可以添加我们感兴趣的待进一步爬取的url，以及对已请求响应内容的操作。
可以看到，`QuotesSpider`继承了scrapy.Spider，并且定义了如下的属性和方法：

- `name`：声明了我们自定义Spider类的姓名，这个姓名在整个Scrapy项目中必须是唯一的，在随后启动时，我们可以在命令行中调用它。
- `start_requests()`这个方法必须要返回可迭代的请求对象`Request`（比如列表，或者一个Request的生成器）
- `parse(response)`该方法是scrapy默认的回调函数，`response`参数接收一个`TextResponse`类的实例对象，其中包含了响应中的内容。在请求完成，并成功返回了响应后，scrapy将默认自动调用parse函数来处理响应内容。

## spider的运行

返回项目的根目录（本例为tutorial/目录下），并在命令行下用如下指令执行

```shell
scrapy crawl quotes
```

其中`quotes`是我们自定义的Spider类的`name`属性值，上述代码将会在本地保存`quotes-1.html`和`quotes-2.html`两个文件，内容分别是start_requests()方法返回的两个请求对象所请求网页的内容。

## 一种快捷的start_requests()操作

取代`start_requests()`方法，我们可以用`start_url`属性来声明待请求的url地址列表。scrapy将自动把这些地址包装成`Request`对象返回，即默认用该列表执行start_requests()方法。

```python
class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]

    def parse(self, response):
        page = response.url.split("/")[-2]
        filename = 'quotes-%s.html' % page
        with open(filename, 'wb') as f:
            f.write(response.body)
```

## 文档选择器

Scrapy自带了对HTML文档的解析工具，通过response对象的css方法，可以细致地选择到我们想选择地元素：

```python
response.css('title')
#[<Selector xpath='descendant-or-self::title' data='<title>Quotes to Scrape</title>'>]
```

`response.css('title')`方法返回的是一个选择器的列表，通过对列表中的选择器`selector`进行访问，我们可以得到更多细致的信息

```python
response.css('title::text').getall()
#['Quotes to Scrape']
```

这里注意两个地方：

1. `::text`指定“我们只选择那些我们感兴趣的标签的文本内容节点”。返回的是字符串的列表。如果不加`::text`，我们的返回结果将会包含它的标签，比如

```python
#['<title>Quotes to Scrape</title>']
```

2. `getall()`方法返回选择器列表中选择器所对应的所有信息，以列表的形式返回。因为一个选择器可能对应了多个匹配的内容。

（这里有个问题：是一个选择器能对应多个匹配结果；还是一个选择器列表中有多个选择器，`getall()`方法将返回所有选择器的内容）

如果你确定你只想要列表中选择器的第一个结果，那么你可以调用`get()`：

```python
response.css('title::text').get()
#'Quotes to Scrape'
```

此时只返回一个内容，你也可以通过下标访问列表中的某一个选择器，如：

```python
response.css('title::text')[0].get()
```

但这种方法不如`getall()`好，一方面它会报`IndexError`的错误（如果你不能事先知道列表元素的个数），而`getall()`在没有匹配元素的时候返回`None`。

你还可以通过re()方法实现对文本内容节点的正则表达式匹配。

```python
response.css('title::text').re(r'Quotes.*')
#['Quotes to Scrape']
response.css('title::text').re(r'Q\w+')
#['Quotes']
```

为了获得元素的其它属性值（如`<a href="..."></a>`），可以通过`::attr(...)`来实现：

```python
response.css('li.next a::attr(href)').get()
#'/page/2/'
```

我们也可以通过访问`attrib`属性来获得html文档元素的属性值：

```python
response.css('li.next a').attrib['href']
#'/page/2/'
```

## 在我们的spider中提取数据

Scrapy的Spider对象可以将提取出的数据保存成数据字典的形式。我们通过`yield`关键字来达到这一目的：

```python
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.css('small.author::text').get(),
                'tags': quote.css('div.tags a.tag::text').getall(),
            }
```

## 储存我们提取的数据

通过使用`Feed exports`（传输导出文件），我们可以简单的将数据保存成JSON形式：

```shell
scrapy crawl quotes -o quotes.json
```

这将会产生一个`quotes.json`文件在本地，这个文件的一个问题在于，如果你重复用这个命令运行爬虫项目（此时文件已包含一次爬取信息），那么Scrapy会将新爬去的信息附加在文件末尾，而不是重写文件。
我们也可以保存成另一种格式，`JSON Lines`:

```shell
scrapy crawl quotes -o quotes.jl
```

`JSON Lines`格式文件不会有重复运行而带来的问题，而且每条记录都是单独的一行。

## 伴随链接

首先我们找到待抓取的链接所在html文档中的位置

```html
<ul class="pager">
    <li class="next">
        <a href="/page/2/">Next <span aria-hidden="true">&rarr;</span></a>
    </li>
</ul>
```

爬取这些伴随链接的代码如下

```python
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.css('small.author::text').get(),
                'tags': quote.css('div.tags a.tag::text').getall(),
            }

        next_page = response.css('li.next a::attr(href)').get()
        if next_page is not None:
            next_page = response.urljoin(next_page)
            yield scrapy.Request(next_page, callback=self.parse)
```

获得了`href`属性值后，我们判断其是否为空（为空则说明已到达文档末尾）由于这个值可能只是一个相对路径，我们使用`urljoin()`方法将这个相对路径自动补全。
接下来我们通过`yield`关键字来处理对新链接生成的请求。Scrapy对这些伴随链接将有这么一个处理机制：yield一个请求，Scrapy将调度这些请求的发送，并且对每个请求制定回调函数，等到Scrapy得到响应以后，再用回调函数处理结果。
所以，上述例子就构成了一个循环：发送请求，分析响应内容以得到关键信息，分析伴随链接：若有，则进入下一轮分析；若无，则停止。

## 产生请求的快捷方式

通过`response.follow`你可以快速产生有效的请求，不用将`href`中的地址补齐

```python
import scrapy


class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = [
        'http://quotes.toscrape.com/page/1/',
    ]

    def parse(self, response):
        for quote in response.css('div.quote'):
            yield {
                'text': quote.css('span.text::text').get(),
                'author': quote.css('span small::text').get(),
                'tags': quote.css('div.tags a.tag::text').getall(),
            }

        next_page = response.css('li.next a::attr(href)').get()
        if next_page is not None:
            yield response.follow(next_page, callback=self.parse)

```

`response.follow`方法允许接收一个相对地址作为参数，指定其回调函数后，方法返回一个Request对象，我们再将其yield处理。

注意，这个方法甚至能够令一些包含了请求URL的选择器作为它的参数，比如

```python
for href in response.css('li.next a::attr(href)'):
    yield response.follow(href, callback=self.parse)
```

对于`<a>`标签，我们还有更快捷的办法，`response.follow()`方法可以直接获取他们的href属性

```python
for a in response.css('li.next a')
 yield response.follow(a,callbakc=self.parse)
```
