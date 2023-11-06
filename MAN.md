# Teaching myself webscraping
Application of webscraping with scrapy library

## 1. Basics

Create a new project:

```python
scrapy startproject <project_name>
```

Create a new spider from a generic spider

```python
scrapy genspider chocolatespider chocolate.co.uk
```

With the scrapy shell we can find our css selectors, the scrapy shell is very useful in finding out how to extract info from css html file

```shell
scrapy shell
```

Edit setuo.cfg after install ipython like this

```python
## scrapy.cfg
[settings]
default = chocolatescraper.settings
shell = ipython
```

First, we fetch the main products page, saving the HTML response into the `response`variable.

```python
fetch('https://www.chocolate.co.uk/collections/all')
response
```

Next we find product selectors

```python
response.css('product-item') # all product information
response.css('product-item').get() # get first product
products = response.css('product-item') # get all products

product.css('a.product-item-meta__title::text').get() # get the name of product

product.css('span.price').get() # gets the price of the product, some replacement needs to be done on top

product.css('div.product-item-meta a').attrib['href'] # get the href of the product
```

We update the spider as follows:

```python
import scrapy 

class ChocolatespiderSpider(scrapy.Spider)

    #the name of the spider
    name = 'chocolatespider'

    #the url of the first page that we will start scraping
    start_urls = ['https://www.chocolate.co.uk/collections/all']

    def parse(self, response):

        #here we are looping through the products and extracting the name, price & url
        products = response.css('product-item')
        for product in products:
            #here we put the data returned into the format we want to output for our csv or json file
            yield{
                'name' : product.css('a.product-item-meta__title::text').get(),
                'price' : product.css('span.price').get().replace('<span class="price">\n              <span class="visually-hidden">Sale price</span>','').replace('</span>',''),
                'url' : product.css('div.product-item-meta a').attrib['href'],
            }
```

The spider is then run with

```shell
scrapy crawl chocolate spider # 24 outputs of same page
scrapy crawl chocolatespider -O myscrapeddata.csv # save output to .csv or .json
scrapy crawl chocolatespider -o myscrapeddata.csv # APPEND output to .csv or .json
```

It is also important to navigate to the next page to get all products. We find the necessary href in the scrapy shell:

```python
fetch('https://www.chocolate.co.uk/collections/all')
response.css('[rel="next"] ::attr(href)').get()
```

Leading to a spider update

```python
import scrapy
 
class ChocolateSpider(scrapy.Spider):

   #the name of the spider
   name = 'chocolatespider'

   #these are the urls that we will start scraping
   start_urls = ['https://www.chocolate.co.uk/collections/all']

   def parse(self, response):

       products = response.css('product-item')
       for product in products:
           #here we put the data returned into the format we want to output for our csv or json file
           yield{
               'name' : product.css('a.product-item-meta__title::text').get(),
               'price' : product.css('span.price').get().replace('<span class="price">\n              <span class="visually-hidden">Sale price</span>','').replace('</span>',''),
               'url' : product.css('div.product-item-meta a').attrib['href'],
           }

       next_page = response.css('[rel="next"] ::attr(href)').get()

       if next_page is not None:
           next_page_url = 'https://www.chocolate.co.uk' + next_page
           yield response.follow(next_page_url, callback=self.parse)
```

## 2. Edge Cases

### Organizing Data with Item Loaders

Creating an Item is very easy. Simply create a Item schema in our `items.py` file.

```python
import scrapy

class ChocolateProduct(scrapy.Item):
   name = scrapy.Field()
   price = scrapy.Field()
   url = scrapy.Field()
```

And and then we import it our `chocolatespider.py` file.

```python

import scrapy
from chocolatescraper.items import ChocolateProduct 
```

Item Loaders provide an easier way to populate items from a scraping process. Item Loader prove a mcuh more convenient mechansim for populating Items. It also makes code more readable by moving all xpath/css refernce to the item loader

To create our Item Loader, we will create a file called itemsloaders.py and define the following Item Loader:

```python

from itemloaders.processors import TakeFirst, MapCompose
from scrapy.loader import ItemLoader

class ChocolateProductLoader(ItemLoader):

    default_output_processor = TakeFirst()
    price_in = MapCompose(lambda x: x.split("£")[-1])
    url_in = MapCompose(lambda x: 'https://www.chocolate.co.uk' + x )
```

We can then clean up our chocolatepspider code:


```python
import scrapy
from chocolatescraper.itemloaders import ChocolateProductLoader
from chocolatescraper.items import ChocolateProduct  

 
class ChocolateSpider(scrapy.Spider):

   # The name of the spider
   name = 'chocolatespider'

   # These are the urls that we will start scraping
   start_urls = ['https://www.chocolate.co.uk/collections/all']

   def parse(self, response):
       products = response.css('product-item')

       for product in products:
            chocolate = ChocolateProductLoader(item=ChocolateProduct(), selector=product)
            chocolate.add_css('name', "a.product-item-meta__title::text")
            chocolate.add_css('price', 'span.price', re='<span class="price">\n              <span class="visually-hidden">Sale price</span>(.*)</span>')
            chocolate.add_css('url', 'div.product-item-meta a::attr(href)')
            yield chocolate.load_item()

       next_page = response.css('[rel="next"] ::attr(href)').get()

       if next_page is not None:
           next_page_url = 'https://www.chocolate.co.uk' + next_page
           yield response.follow(next_page_url, callback=self.parse)
```

### Processsing our Data with scrapy Item Pipelines

Once an Item has been scraped by the spider, it is sent to the `Item Pipeline` for validation and processing.

Each Item Pipeline is a Python class that implements a simple method called ``process_item``.

The process_item method takes in an Item, performs an action on it and decides if the item should continue through the pipeline or be dropped.

```python
from itemadapter import ItemAdapter
from scrapy.exceptions import DropItem


class PriceToUSDPipeline:

    gbpToUsdRate = 1.3

    def process_item(self, item, spider):
        adapter = ItemAdapter(item)

        ## check is price present
        if adapter.get('price'):

            #converting the price to a float
            floatPrice = float(adapter['price'])

            #converting the price from gbp to usd using our hard coded exchange rate
            adapter['price'] = floatPrice * self.gbpToUsdRate

            return item

        else:
            # drop item if no price
            raise DropItem(f"Missing price in {item}")

class DuplicatesPipeline:


    def __init__(self):
        self.names_seen = set()

    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        if adapter['name'] in self.names_seen:
            raise DropItem(f"Duplicate item found: {item!r}")
        else:
            self.names_seen.add(adapter['name'])
            return item
```

To enable the PriceToUSDPipeline and DuplicatesPipelinewe just created, we need to add it to the ITEM_PIPELINES settings in our settings.py file, like in the following example:

```python

ITEM_PIPELINES = {
    'chocolatescraper.pipelines.PriceToUSDPipeline': 100,
    'chocolatescraper.pipelines.DuplicatesPipeline': 200,
}

```


The integer values you assign to classes in this setting determine the order in which they run. Items go through from lower valued to higher valued classes. It’s customary to define these numbers in the 0-1000 range.






