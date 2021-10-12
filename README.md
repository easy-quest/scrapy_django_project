sudo pacman -Sy postgresql postgresql-libs postgis
sudo passwd postgres
su - postgres -c "initdb --locale en_US.UTF-8 -D '/var/lib/postgres/data'"
systemctl start postgresql
systemctl enable postgresql
pamac install gdal geo

## Setup and test PostGIS, GeoDjango

### Test GEOS, GDAL, PyProj

```
# python manage.py shell

#GDAL

>>> from django.contrib.gis import gdal
>>> gdal.HAS_GDAL
True

#GEOS
>>> from django.contrib.gis import geos
>>> geos.geos_version()

#Proj
>>> import pyproj
>>> pyproj.__version__
>>> pyproj.test()

```
### Create DB (for PostgreSQL 9.1+)

```
# su - postgres

createdb easydb
psql easydb

> CREATE EXTENSION postgis;
> CREATE EXTENSION postgis_topology;

Configure db in settings.py and add django.contrib.gis to INSTALLED_APPS

DATABASES = {
    'default': {
         'ENGINE': 'django.contrib.gis.db.backends.postgis',
         'NAME': 'dbname',
         'USER': 'user',
         'PASSWORD': 'password',
     }
}

INSTALLED_APPS = (
    'django.contrib.admin',
    ...
    'django.contrib.gis',
    ...
)
```
```bash
dotenv set DATABASE_PASSWORD 12345678
dotenv set DATABASE_NAME easydb
dotenv set DATABASE_USER easy_quest
dotenv set DATABASE_HOST 127.0.0.1
dotenv set DATABASE_PORT 5432
```
pipenv install psycopg2
python manage.py makemigrations
python manage.py migrate

### Items

The Item is used to structure the data parsed by the spider. The Django model created earlier will be used to create the Item. DjangoItem is a class of item that gets its fields definition from a Django model, and here using the Property model defined in the main app created earlier. Update the `scraper/scraper/items.py`.

```python
# Define here the models for your scraped items
#
# See documentation in:
# https://docs.scrapy.org/en/latest/topics/items.html
import scrapy
from scrapy_djangoitem import DjangoItem
from properties.models import Property


class ScraperItem(DjangoItem):
    django_model = Property
    # define the fields for your item here like:
    # name = scrapy.Field()
    pass
```

### Pipelines.

The Item Pipeline is where the data is processed once the items have been extracted from the spiders. The pipeline performs tasks such as validation and storing items in a database. Update the `scraper/scraper/pipelines.py`.

```python
from word2number import w2n

class ScraperPipeline(object):
    """
    Saves Item to the database
    """
    def process_item(self, item, spider):
        item.save()
        return item


class PropertyStatusPipeline(object):
    """
    Replace text for item status i.e For Rent will be replaced with Rent.
    """
    def process_item(self, item, spider):
        if item.get('status'):
            item['status'] = item['status'].replace('For ', '')
            return item


class PropertyPricePipeline(object):
    """
    Removes signs from the price value. i.e replaces 10000/= with 10000
    """
    def process_item(self, item, spider):
        if item.get('price'):
            item['price'] = item['price'].replace('/=', '')
            return item


class ConvertNumPipeline(object):
    """
    Converts words to number values for bedrooms and bathrooms
    """
    def process_item(self, item, spider):
        if item.get('bathrooms'):
            item['bathrooms'] = w2n.word_to_num(item['bathrooms'])
        if item.get('bedrooms'):
            item['bedrooms'] = w2n.word_to_num(item['bedrooms'])
        return item
```

### Spiders

Define how a website will be scraped, including how to perform the crawl (i.e. follow links) and how to extract structured data from the web-pages. Create a properties spider `scraper/spiders/properties_spider.py`.

---
created: 2021-10-12T19:01:04 (UTC +00:00)
source: https://medium.com/analytics-vidhya/web-scraping-with-scrapy-and-django-94a77386ac1b
author: Ivan Yastrebov
---

# Web Scraping with Scrapy and Django. | by walugembe peter | Analytics Vidhya | Medium

> ## Excerpt
> Web scraping is the process of extracting data from websites. It simulates human interaction with a web page to retrieve wanted…

---
The Spider has features defined below.

-   **name** : a string which defines it.
-   **allowed\_domains** : an optional list of strings containing domains that this spider is allowed to crawl.
-   **start\_urls** : a list of URLs where the spider will begin to crawl from.
-   **rules** : a list of one (or more) `[**Rule**](https://docs.scrapy.org/en/latest/topics/spiders.html#scrapy.spiders.Rule)` objects. Each `[**Rule**](https://docs.scrapy.org/en/latest/topics/spiders.html#scrapy.spiders.Rule)` defines a certain behaviour for crawling the site. **link\_extractor** is a [Link Extractor](https://docs.scrapy.org/en/latest/topics/link-extractors.html#topics-link-extractors) object which defines how links will be extracted from each crawled page, and **call\_back** is a method from the spider object which will be used for each link extracted with the specified link extractor.
-   **Item\_loaders** : provide a convenient mechanism for populating scraped Items. The **add\_css** method receives an Item field in which the extracted data will be stored and a CSS selector which is used to extract data from the web page. The **load\_item** method populates the item with the data collected and return it, which is saved to database through the pipeline.

Update the `scraper/scraper/items.py` with the values below, to activate `pipelines`, add `user_agent`, update `spider_modules`and `newspider_modules`.
