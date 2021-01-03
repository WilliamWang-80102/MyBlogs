# Spiders类

Spider类定义了我们该如何爬取网页（或者说追踪链接）以及如何从网页中提取我们需要的内容。
Spider类对象的执行过程分为以下三个阶段：

1. 产生初始请求，并指定用于处理相关响应的回调函数。
`start_requests()`方法将获得我们声明的初始url（定义在属性`start_urls`中），并根据他们产生Request对象，同时指定回调函数。

2. 回调函数被调用。
回调函数通过Scrapy内置的Selector选择器来对响应的内容进行提取。最终返回字典，Item对象或者Request对象，或者这些对象的生成器。返回的请求也将被指定相关的回调函数用来处理响应的内容。

3. 输出结果。
最终，函数返回的Item对象或者字典将被存储进数据库（使用`ItemPipeline`）或者输出成文件（使用`Feed export`）

## scrapy.Spider

**class scrapy.Spiders.Spider**
该类是所有Spider都需要继承的类，因此它本身也没有很多很复杂的功能。主要逻辑是：调用默认的`start_requests()`方法，将它的属性`start_urls`中所包含的url包装成请求发送出去，并使用默认的`parse()`方法来处理所有的响应。
Spider对象的属性：
**name**
spider的名字，用于Scrapy定位，必须唯一。
**allowed_domains**
该属性可选，值为一个字符串列表。属性定义了spider能爬取的域。比如如果要爬`https://www.example.com/1.html`，那么就可以将`example.com`加入列表。
**start_urls**
包含爬虫开始时最初的url列表，根据该类表依次生成Request对象。
**start_requests()**
爬虫开始后将自动调用该函数，该函数必须返回一个Request类型的可迭代对象，可以返回列表，也可以让该方法以迭代器的形式实现：

```python
def start_requests(self):
    urls = [
        'http://quotes.toscrape.com/page/1/',
        'http://quotes.toscrape.com/page/2/',
    ]
    for url in urls:
        yield scrapy.Request(url=url, callback=self.parse)
```

默认的，这个函数将生成Request(url,dont_filter=True)对象，其中的url来自属性`start_urls`

```python
import scrapy

class MySpider(scrapy.Spider):
    name = 'example.com'
    allowed_domains = ['example.com']
    start_urls = [
        'http://www.example.com/1.html',
        'http://www.example.com/2.html',
        'http://www.example.com/3.html',
    ]

    def parse(self, response):
        for h3 in response.xpath('//h3').getall():
            yield {"title": h3}

        for href in response.xpath('//a/@href').getall():
            yield scrapy.Request(response.urljoin(href), self.parse)
```

如果你想自定义对初始Request要进行的操作，那么就应该修改这个方法，比如你希望先登录页面，然后对跳转页面的内容进行操作：

```python
class MySpider(scrapy.Spider):
    name = 'myspider'

    def start_requests(self):
        return [scrapy.FormRequest("http://www.example.com/login",
                                   formdata={'user': 'john', 'pass': 'secret'},
                                   callback=self.logged_in)]

    def logged_in(self, response):
        # here you would extract links to follow and return Requests for
        # each of them, with another callback
        pass
```
