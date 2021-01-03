# Item Loaders

`Item`对象相当于存储我们提取的信息的容器，他类似于`dict`的特性能帮助我们格式化的存储。而`ItemLoader`对象则相当于容器的填充工具，用来将信息装载到`Item`对象中。

## 使用ItemLoader来装载数据

首先，初始化`ItemLoader`对象时，我们可以通过`Item`对象、`dict`来初始化它的`item参数`，当然也可以不设置，此时该参数被赋予默认值`ItemLoader.default_item_class`如：

```python
l = ItemLoader(item=Product(), response=response)
```

然后我们就可以通过`Selector`选择器对象来为`Item`的字段进行数据分析。同一个字段可以有多个赋值，这样所有的赋值会拼接起来。
下面来看一个`ItemLoader`的使用实例，`Item`类选用`Product`：

```python
from scrapy.loader import ItemLoader
from myproject.items import Product

def parse(self, response):
    l = ItemLoader(item=Product(), response=response)
    l.add_xpath('name', '//div[@class="product_name"]')
    l.add_xpath('name', '//div[@class="product_title"]')
    l.add_xpath('price', '//p[@id="price"]')
    l.add_css('stock', 'p#stock]')
    l.add_value('last_updated', 'today') # you can also use literal values
    return l.load_item()
```

ItemLoader对象的`add_xpath()`方法接收两个参数，第一个要赋值的`Item字段`，另一个是`xpath语句`，来提取文档中的数据。
`add_css()`方法同理于上，但是使用的是CSS选择器。
`add_value()`方法不经过选择器，直接将文本常量赋值给Item的字段。
在所有的数据都定位并提取完毕后，我们通过调用`ItemLoader.load_item()`方法来用获得的数据对Item对象各个字段进行填充。

## 输入处理器与输出处理器

一个ItemLoader对象为每一个Item的字段都设置了输入处理器与输出处理器：

- 输入处理器一旦接受到被提取的数据后，就立刻对其进行处理（即数据经由`add_xpath()`方法，`add_css()`方法，`add_value()`方法提取），并将数据搜集的结果存放在ItemLoader对象内。
- 输出处理器在`ItemLoader.load_item()`方法被调用之后被调用（load_item方法对Item对象进行填充并返回Item对象），存储在ItemLoader中的结果经过输出处理器处理之后被赋值给了Item对象。

值得一提的是，这些处理器其实都是些被调用的对象，给予待解析的数据然后返回解析的结果，所以我们可以用任意其它的函数来作为我们的输入输出处理器。但这些的函数有且仅有一个形参，而且它必须是一个迭代器。
另一个值得注意的事是输入处理器返回的结果将是一个序列，它被保存在ItemLoader对象中，并在之后作为输出处理器的输入来填充Item对象的Field。
Scrapy内置有很多常用的处理器，方便我们使用。

如果你想用一个简单函数（不是某一个类的方法）来作为ItemLoader类的输入输出处理器（这也是一个确定输入输出处理器的方法），那么它的必须将第一个参数必须是`self`，这是因为它成为一个方法后，`self`将默认的被传入该对象的实例。如下：

```python
def lowercase_processor(self, values):
    for v in values:
        yield v.lower()

class MyItemLoader(ItemLoader):
    name_in = lowercase_processor
```

## 声明一个自定义的ItemLoader类

像类一样，ItemLoader使用常规的类定义语法进行定义：

```python
from scrapy.loader import ItemLoader
from scrapy.loader.processors import TakeFirst, MapCompose, Join

class ProductLoader(ItemLoader):

    default_output_processor = TakeFirst()

    name_in = MapCompose(unicode.title)
    name_out = Join()

    price_in = MapCompose(unicode.strip)

    # ...
```

- 可以设置`ItemLoader.default_input_processor`属性和`ItemLoader.default_input_processor`属性来设置默认的处理器。
- 设置字段+后缀（`_in`或者`_out`）来设置字段的输入、输出处理器。

## 声明输入输出处理器

除了如上所述的在`ItemLoader`的对象中声明输入输出处理器以外，我们还可以在`Item`对象的`Field()`函数中声明输入与输出处理器函数：

```python
class Product(scrapy.Item):
    name = scrapy.Field(
        input_processor=MapCompose(remove_tags),
        output_processor=Join(),
    )
    price = scrapy.Field(
        input_processor=MapCompose(remove_tags, filter_price),
        output_processor=TakeFirst(),
    )
```

这里考虑输入输出处理器有效的优先级：

1. `field_in`和`field_out`（最高优先级）
2. `Field()`方法中的元数据（`input_processor`和`output_processor`关键字）
3. ItemLoader的默认处理器属性（`ItemLoader.default_input_processor`和`ItemLoader.default_output_processor`）

## ItemLoader的上下文

通过设置`ItemLoader`对象的`context`属性，我们可以设置一个能被输入输出处理器共同访问的上下文环境变量。通过在回调函数上加上`loader_context`参数，我们就可以显式的告知ItemLoader对象，这个处理器函数需要接受上下文参数，ItemLoader对象就会把当前有效的上下文参数赋值给它。
设置`context`上下文属性的方法有下面几种：

1. 设置当前有效的ItemLoader对象的context属性

```python
loader = ItemLoader(product)
loader.context['unit'] = 'cm'
```

2. 在实例化ItemLoader对象时

```python
loader = ItemLoader(product, unit='cm')
```

此时任何关键词参数(**kwargs)都会以键值对的形式存储在上下文环境中。

3. 在ItemLoader类中，定义某个Item的输入输出函数时

```python
class ProductLoader(ItemLoader):
    length_out = MapCompose(parse_length, unit='cm')
```
