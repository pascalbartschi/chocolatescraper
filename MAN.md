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






