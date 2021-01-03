# 选择器

现有的流行的网页解析工具：

- `BeautifulSoup`对残缺文档的处理能力强，但是处理得很慢
- `lxml`基于文档树对文档进行处理

Scrapy在处理文档上有自己的机制，可以通过`XPath`和`CSS`表达式来获得指定元素的选择器

## 如何使用选择器？

### 创建选择器

`response`对象中包含了一个`Selector`的实例，我们可以通过它的`.selector`属性来访问

```python
response.selector.css('title::text').getall()
```

由于对响应内容的查询操作相当频繁，我们简化了这行代码

```python
response.css('span::text').get()
```

### 使用选择器

我们对这样一段HTML代码进行处理

```html
<html>
 <head>
  <base href='http://example.com/' />
  <title>Example website</title>
 </head>
 <body>
  <div id='images'>
   <a href='image1.html'>Name: My image 1 <br /><img src='image1_thumb.jpg' /></a>
   <a href='image2.html'>Name: My image 2 <br /><img src='image2_thumb.jpg' /></a>
   <a href='image3.html'>Name: My image 3 <br /><img src='image3_thumb.jpg' /></a>
   <a href='image4.html'>Name: My image 4 <br /><img src='image4_thumb.jpg' /></a>
   <a href='image5.html'>Name: My image 5 <br /><img src='image5_thumb.jpg' /></a>
  </div>
 </body>
</html>
```

为了访问HTML中title元素的文本内容，我们可以：

```python
response.css('title::text')
#[<Selector xpath='//title/text()' data='Example website'>]
```

`css()`为我们返回了一个选择器的列表，如果我们要获得列表中所有选择器的文本数据`data`，我们需要使用`getall()`方法，如下：

```python
response.css('title::text').getall()
#['Example website']
response.css('title::text').get()
#'Example website'
```

`getall()`方法将返回数据的列表，而`get()`方法返回匹配的选择器列表中第一个匹配选择器的数据，如果没有匹配的选择器，则返回`None`。
值得注意的是，`css()`方法和`xpath()`方法返回的选择器列表可以产生新的选择器列表，这种特性帮助我们很方便地处理嵌套的数据，比如：

```python
response.css('img').xpath('@src').getall()
#['image1_thumb.jpg',
 'image2_thumb.jpg',
 'image3_thumb.jpg',
 'image4_thumb.jpg',
 'image5_thumb.jpg']
```

对于`get()`我们也可以向其传递参数，用以确定如果没有任何匹配项时，我们能返回参数指定的信息，如：

```python
response.xpath('//div[@id="not-exists"]/text()').get(default='not-found')
#'not-found'
```

除了`xpath`下的`@src`语法，我们可以用选择器的`.attrib`属性来指定我们想访问的元素的属性，也可以使用`::attr()`方法，他们都会返回选择器对应元素的该属性的值：

```python
#[img.get() for img in response.css('img::attr(src)')]

[img.attrib['src'] for img in response.css('img')]

#['image1_thumb.jpg',
 'image2_thumb.jpg',
 'image3_thumb.jpg',
 'image4_thumb.jpg',
 'image5_thumb.jpg']
```

### 再看看CSS的这两个伪元素

W3C标准下的CSS其实并没有声明访问元素的文本内容和元素的属性，但在处理文档中对他们的访问是如此重要，因此我们引入两个非标准的伪元素`::text`以及`::attr(name)`
**::text**

- `css('tname::text')`返回包含tname元素的所有文本节点内容的选择器对象（每个选择器的data字段是文本内容）。例如：

```python
response.css('title::text').get()
#'Example website'
```

- `*::text`返回的是当前节点下所有的文本节点（这也包括当前节点的子孙节点中的所有文本内容）

```python
response.css('#images *::text').getall()
['\n   ',
 'Name: My image 1 ',
 '\n   ',
 'Name: My image 2 ',
 '\n   ',
 'Name: My image 3 ',
 '\n   ',
 'Name: My image 4 ',
 '\n   ',
 'Name: My image 5 ',
 '\n  ']
```

- 如果节点存在，但没有文本内容，`getall()`将返回空的列表，

```python
response.css('img::text').getall()
[]
```

而`get()`将返回`None`，此时如果你希望返回一些字符串信息，需要调用`get(default='blabla...')`

**::attr(name)**
注意`css(tname::attr(aname))`和`css(tname).attrib[aname]`的区别：

- `css(tname::attr(aname))`返回的仍然是SelectorList，需要通过调用`get()`或者`getall()`来获取具体的数据值。
- `css(tname).attrib[aname]`获取的是选择器列表中第一个选择器的属性值。
