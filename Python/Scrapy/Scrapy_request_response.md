# Request对象

向服务器发送的对象，得到响应后，将响应对象传递给回调函数`callback`
构造器参数：

- **url**(string) - 描述请求的目标网址
- **callback**(callable) - 接收到服务器发送回的响应后响应对象被传递到的方法
- **method**(string) - HTTP请求执行的方法，默认`'GET'`方法
- **meta**(dict) - 请求对象的元数据,传递字典时执行浅拷贝
- **body**(str or unicode) - 请求的体
- **headers**(dict) - 请求的头。字典的值可以是字符串，也可以是列表。若是`None`那么该HTTP头不会发送
- **coookies**(dict or list)

1. 使用字典：

```python
request_with_cookies = Request(url="http://www.example.com",
            cookies={'currency': 'USD', 'country': 'UY'})
```

2. 使用列表：

```python
request_with_cookies = Request(url="http://www.example.com",
                              cookies=[{'name': 'currency',
                                        'value': 'USD',
                                        'domain': 'example.com',
                                        'path': '/currency'}])
```

后者允许我们自定义cookie的`domain`和`path`属性，这在为请求保存cookie时会起到作用。

- **encoding**(string) - 请求的编码，默认utf-8
- **priority**(int) - 请求优先级（默认0）
- **dont_filter**(boolean) - 指明调度器不应该过滤掉请求（默认False）。如果需要重复执行请求，则设置为True。谨慎设置该参数，防止出现死循环。
- **errback**(callable) - 请求执行出现异常时调用该函数。异常包括404 HTTP错误等。
- **flags**(list) - 请求的一些标志，用来记录。

可访问属性：

- **url**
- **method**
- **body**
- **meta**
一个请求的描述数据的字典，该属性默认为空，只会被Scrapy的组件（如中间件，扩展等）所填充内容。元数据中包含一些保留关键字，以供Scrapy组件读取。
- **copy()**
返回一个请求的拷贝
- **replace()**

## 通过设置meta来让各函数访问附加数据

例如，有时候一个Item对象不能在一个函数中初始化完毕并返回，此时我们可以将item对象绑定到请求中：

```python
def parse_page1(self, response):
    item = MyItem()
    item['main_url'] = response.url
    request = scrapy.Request("http://www.example.com/some_page.html",
                             callback=self.parse_page2)
    request.meta['item'] = item
    yield request

# parse_page2中通过response访问到request绑定到meta属性中的item对象
def parse_page2(self, response):
    item = response.meta['item']
    item['other_url'] = response.url
    yield item
```

## 请求对象的子类 -- FormRequest对象

该类继承Request类，主要用于处理HTML表单
`class scrapy.http.FromRequest(url[ ,formdata,...])`
这个类的构造器相较于Request类的构造器只增加了一个参数：
**formdata**(dict or iterable tuples) - 是包含了表单所需信息的字典或者可迭代的键值对，这些数据将进行Url编码并附在请求体中

FormRequest类还包含下面这个类方法：
`classmethod from_response(response[, formname=None, formid=None, formnumber=0, formdata=None, formxpath=None, formcss=None, clickdata=None, dont_click=False, ...])`
该方法将返回一个FormRequest类对象，该对象填充了response中所包含的表单中的字段值。
方法参数：
**response**(Response object)
**formname**(string)
**formid**(string)
**formxpath**(string)
**formcss**(string)
**formnumber**(integer)
**formdata**(dict)
**clickdata**(dict)
**dont_click**(boolean)
**callback**(callable)
