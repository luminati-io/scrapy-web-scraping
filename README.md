# Web Scraping With Scrapy

[![Bright Data Promo](https://github.com/luminati-io/LinkedIn-Scraper/raw/main/Proxies%20and%20scrapers%20GitHub%20bonus%20banner.png)](https://brightdata.com/)

This guide demonstrates a practical application of web scraping to address a common parental challenge: collecting and organizing information sent from schools. We'll focus specifically on gathering homework details and lunch menus.

Here's a diagram showing the overall architecture of what we're building:

![Diagram of the final project](https://media.brightdata.com/2024/04/diagram-of-the-final-project.png)

Here is the plan:

- [Set Up the Project](#setting-up-the-project)
- [Develop the Homework Spider](#developing-the-homework-spider)
- [Develop the Meal Spider](#developing-the-meal-spider)
- [Format the Data](#data-formatting)
- [Handle Web Scraping Challenges](#handling-web-scraping-challenges)

## Requirements

Before starting this tutorial, ensure you have:

- [Python 3.10+](https://www.python.org/downloads/) installed
- An [activated Python virtual environment](https://docs.python.org/3/library/venv.html)
- [Scrapy 2.11.1+](https://docs.scrapy.org/en/latest/intro/install.html#intro-install) installed via pip
- A code editor (like [VS Code](https://code.visualstudio.com/) or [PyCharm](https://www.jetbrains.com/pycharm/))

For privacy purposes, we'll be working with this demonstration school website: [https://systemcraftsman.github.io/scrapy-demo/website/](https://systemcraftsman.github.io/scrapy-demo/website/)

## Setting Up the Project

First, create your project directory:

```sh
mkdir school-scraper
```

Navigate to this directory and initialize a new Scrapy project:

```sh
cd school-scraper & \
scrapy startproject school_scraper
```

This generates a folder structure like:

```
school-scraper
└── school_scraper
    ├── school_scraper
    │   ├── __init__.py
    │   ├── items.py
    │   ├── middlewares.py
    │   ├── pipelines.py
    │   ├── settings.py
    │   └── spiders
    │       └── __init__.py
    └── scrapy.cfg
```

The command creates two nested `school_scraper` directories. In the inner directory, Scrapy generates several key files:
- `middlewares.py` for defining Scrapy [middlewares](https://docs.scrapy.org/en/latest/topics/spider-middleware.html)
- `pipelines.py` for creating custom data processing pipelines
- `settings.py` for configuring your scraping application
- A `spiders` folder where you'll place your spider classes

Currently, the `spiders` folder is empty, but we'll populate it next.

## Developing the Homework Spider

To gather homework information, we need a spider that logs into the school system and navigates to the homework assignments page:

![diagram showing the process of creating a spider and navigating to scrape the data](https://media.brightdata.com/2024/04/diagram-showing-the-process-of-creating-a-spider-and-navigating-to-scrape-the-data.png)

Use the Scrapy CLI to generate a spider. From the `school-scraper/school_scraper` directory, run:

```sh
scrapy genspider homework_spider systemcraftsman.github.io/scrapy-demo/website/index.html
```

> **REMINDER:** Ensure you run all Python and Scrapy commands within your activated virtual environment.

You should see output like:

```
Created spider 'homework_spider' using template 'basic' in module:
  school_scraper.spiders.homework_spider
```

This creates `homework_spider.py` in the `school_scraper/spiders` directory with this content:

```python
class HomeworkSpiderSpider(scrapy.Spider):
    name = "homework_spider"
    allowed_domains = ["systemcraftsman.github.io"]
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]

    def parse(self, response):
        pass
```

Rename the class to `HomeworkSpider` to remove the redundant `Spider` in the name. The `parse` method is our entry point for scraping, which we'll use for logging in.

> **NOTE:**
> 
> The login form at `https://systemcraftsman.github.io/scrapy-demo/index.html` is simulated with JavaScript. Since it's a static HTML page, we'll use an HTTP GET request to simulate the login process.

Update the `parse` method like this:

```python
...code omitted...
    def parse(self, response):
        formdata = {'username': 'student',
                    'password': '12345'}
        return FormRequest(url=self.welcome_page_url, method="GET", formdata=formdata,
                           callback=self.after_login)
```

This submits a form request to log in, which redirects to our `welcome_page_url` with the `after_login` callback to continue scraping.

Add the `welcome_page_url` to the class variables:

```python
...code omitted...
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]
    welcome_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/welcome.html"
...code omitted...
```

Next, add the `after_login` method after the `parse` method:

```python
...code omitted...
    def after_login(self, response):
        if response.status == 200:
            return Request(url=self.homework_page_url,
                           callback=self.parse_homework_page
                           )
...code omitted...
```

This method checks for a successful response (status 200), then navigates to the homework page and calls our `parse_homework_page` method.

Add the `homework_page_url` to the class variables:

```python
...code omitted...
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]
    welcome_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/welcome.html"
    homework_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/homeworks.html"
...code omitted...
```

Now add the `parse_homework_page` method after `after_login`:

```python
...code omitted...
    def parse_homework_page(self, response):
        if response.status == 200:
            data = {}
            rows = response.xpath('//*[@class="table table-file"]//tbody//tr')
            for row in rows:
                if self._get_item(row, 4) == self.date_str:
                    if self._get_item(row, 2) not in data:
                        data[self._get_item(row, 2)] = self._get_item(row, 3)
                    else:
                        data[self._get_item(row, 2) + "-2"] = self._get_item(row, 3)
            return data
...code omitted...
```

This method verifies the response status is 200, then extracts homework data from an HTML table. It uses XPath to retrieve rows, then extracts specific data points with our helper method `_get_item`.

Add the `_get_item` helper method to your class:

```python
...code omitted...
    def _get_item(self, row, col_number):
        item_str = ""
        contents = row.xpath(f'td[{col_number}]//text()').extract()
        for content in contents:
            item_str = item_str + content
        return item_str
```

This helper method extracts cell content using XPath with row and column indices. If a cell contains multiple text nodes, it concatenates them.

The `parse_homework_page` method also needs a `date_str` variable. We'll set it to `12.03.2024` to match our static demo site data:

> **NOTE:**
> 
> In a real application, you'd likely generate this date dynamically.

Add the `date_str` to the class variables:

```python
...code omitted...
    homework_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/homeworks.html"
    date_str = "12.03.2024"
...code omitted...
```

Your completed `homework_spider.py` should now look like this:

```python
import scrapy

from scrapy import FormRequest, Request


class HomeworkSpider(scrapy.Spider):
    name = "homework_spider"
    allowed_domains = ["systemcraftsman.github.io"]
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]
    welcome_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/welcome.html"
    homework_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/homeworks.html"
    date_str = "12.03.2024"

    def parse(self, response):
        formdata = {'username': 'student',
                    'password': '12345'}
        return FormRequest(url=self.welcome_page_url, method="GET", formdata=formdata,
                           callback=self.after_login)

    def after_login(self, response):
        if response.status == 200:
            return Request(url=self.homework_page_url,
                           callback=self.parse_homework_page
                           )

    def parse_homework_page(self, response):
        if response.status == 200:
            data = {}
            rows = response.xpath('//*[@class="table table-file"]//tbody//tr')
            for row in rows:
                if self._get_item(row, 4) == self.date_str:
                    if self._get_item(row, 2) not in data:
                        data[self._get_item(row, 2)] = self._get_item(row, 3)
                    else:
                        data[self._get_item(row, 2) + "-2"] = self._get_item(row, 3)
            return data

    def _get_item(self, row, col_number):
        item_str = ""
        contents = row.xpath(f'td[{col_number}]//text()').extract()
        for content in contents:
            item_str = item_str + content
        return item_str
```

Test your spider by running:

```
scrapy crawl homework_spider
```

You should see output similar to:

```
...output omitted...
2024-03-20 01:36:05 [scrapy.core.scraper] DEBUG: Scraped from <200 https://systemcraftsman.github.io/scrapy-demo/website/homeworks.html>
{'MATHS': "Matematik Konu Anlatımlı Çalışma Defteri-6 sayfa 13'ü yapınız.\n", 'ENGLISH': 'Read the story "Manny and His Monster Manners" on pages 100-107 in your Reading Log and complete the activities on pages 108 and 109 according to the story.\n\nReading Log kitabınızın 100-107 sayfalarındaki "Manny and His Monster Manners" isimli hikayeyi okuyunuz ve 108 ve 109\'uncu sayfalarındaki aktiviteleri hikayeye göre tamamlayınız.\n'}
2024-03-20 01:36:05 [scrapy.core.engine] INFO: Closing spider (finished)
...output omitted...
```

## Developing the Meal Spider

Create another spider for the meal list page:

```sh
scrapy genspider meal_spider systemcraftsman.github.io/scrapy-demo/website/index.html
```

This generates a new file with:

```python
class MealSpiderSpider(scrapy.Spider):
    name = "meal_spider"
    allowed_domains = ["systemcraftsman.github.io"]
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]

    def parse(self, response):
        pass
```

The meal spider follows a similar pattern to the homework spider, with the main difference being the HTML parsing logic.

Replace the contents of `meal_spider.py` with:

```python
import scrapy

from datetime import datetime

from scrapy import FormRequest, Request


class MealSpider(scrapy.Spider):
    name = "meal_spider"
    allowed_domains = ["systemcraftsman.github.io"]
    start_urls = ["https://systemcraftsman.github.io/scrapy-demo/website/index.html"]
    welcome_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/welcome.html"
    meal_page_url = "https://systemcraftsman.github.io/scrapy-demo/website/meal-list.html"
    date_str = "13.03.2024"

    def parse(self, response):
        formdata = {'username': 'student',
                    'password': '12345'}
        return FormRequest(url=self.welcome_page_url, method="GET", formdata=formdata,
                           callback=self.after_login)

    def after_login(self, response):
        if response.status == 200:
            return Request(url=self.meal_page_url,
                           callback=self.parse_meal_page
                           )

    def parse_meal_page(self, response):
        if response.status == 200:
            data = {"BREAKFAST": "", "LUNCH": "", "SALAD/DESSERT": "", "FRUIT TIME": ""}
            week_no = datetime.strptime(self.date_str, '%d.%m.%Y').isoweekday()
            rows = response.xpath('//*[@class="table table-condensed table-yemek-listesi"]//tr')
            key = ""
            try:
                for row in rows[1:]:
                    if self._get_item(row, week_no) in data.keys():
                        key = self._get_item(row, week_no)
                    else:
                        data[key] = self._get_item(row, week_no, "\n")
            finally:
                return data

    def _get_item(self, row, col_number, seperator=""):
        item_str = ""
        contents = row.xpath(f'td[{col_number}]//text()').extract()
        for i, content in enumerate(contents):
            item_str = item_str + content + seperator
        return item_str
```

Note that the `parse` and `after_login` methods are nearly identical to those in the homework spider. The main difference is in the `parse_meal_page` method, which extracts meal information using different XPath expressions and data handling logic.

Test your meal spider with:

```sh
scrapy crawl meal_spider
```

You should see:

```
...output omitted...
2024-03-20 02:44:42 [scrapy.core.scraper] DEBUG: Scraped from <200 https://systemcraftsman.github.io/scrapy-demo/website/meal-list.html>
{'BREAKFAST': 'PANCAKE \n KREM PEYNİR \n SÜZME PEYNİR \n\nKAKAOLU FINDIK KREMASI \n SÜT\n', 'LUNCH': 'TARHANA ÇORBA\nEKŞİLİ KÖFTE\nERİŞTE\n', 'SALAD/DESSERT': 'AYRAN\nKIRMIZILAHANA SALATA\nROKALI GÖBEK SALATA\n', 'FRUIT TIME': 'FINDIK& KURU ÜZÜM\n'}
2024-03-20 02:44:42 [scrapy.core.engine] INFO: Closing spider (finished)
...output omitted...
```

## Data Formatting

Now that your spiders can extract data in JSON format, you may want to format it more nicely by programmatically triggering the spiders.

While Scrapy projects typically don't require a `main.py` entry point (since the Scrapy CLI provides the framework for running spiders), we'll create one to format our output data.

Create a file named `main.py` in the root directory of your `school-scraper` project with:

```python
import sys

from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

from school_scraper.school_scraper.spiders.homework_spider import HomeworkSpider
from school_scraper.school_scraper.spiders.meal_spider import MealSpider

results = []


class ResultsPipeline(object):
    def process_item(self, item, spider):
        results.append(item)


def _prepare_message(title, data_dict):
    if len(data_dict.items()) == 0:
        return None

    message = f"===={title}====\n----------------\n"
    for key, value in data_dict.items():
        message = message + f"==={key}===\n{value}\n----------------\n"

    return message


def main(args=None):
    if args is None:
        args = sys.argv
    settings = get_project_settings()
    settings.set("ITEM_PIPELINES", {'__main__.ResultsPipeline': 1})
    process = CrawlerProcess(settings)

    if args[1] == "homework":
        process.crawl(HomeworkSpider)
        process.start()
        print(_prepare_message("HOMEWORK ASSIGNMENTS", results[0]))
    elif args[1] == "meal":
        process.crawl(MealSpider)
        process.start()
        print(_prepare_message("MEAL LIST", results[0]))


if __name__ == "__main__":
    main()
```

This script provides an entry point for your application. The `main` function processes command line arguments to determine which spider to run.

It configures a custom `ResultsPipeline` that collects the spider's output into the `results` array. The `_prepare_message` helper function formats this data for display.

Based on the argument provided (either "homework" or "meal"), the script runs the appropriate spider and formats its output.

Run it to get homework assignments:

```sh
python main.py homework
```

You should see:

```
...output omitted...
====HOMEWORK ASSIGNMENTS====
----------------
===MATHS===
Matematik Konu Anlatımlı Çalışma Defteri-6 sayfa 13'ü yapınız.

----------------
===ENGLISH===
Read the story "Manny and His Monster Manners" on pages 100-107 in your Reading Log and complete the activities on pages 108 and 109 according to the story.

Reading Log kitabınızın 100-107 sayfalarındaki "Manny and His Monster Manners" isimli hikayeyi okuyunuz ve 108 ve 109'uncu sayfalarındaki aktiviteleri hikayeye göre tamamlayınız.

----------------
...output omitted...
```

For the meal list:

```sh
python main.py meal
```

Output:

```
...output omitted...
====MEAL LIST====
----------------
===BREAKFAST===
PANCAKE 
 KREM PEYNİR 
 SÜZME PEYNİR 

KAKAOLU FINDIK KREMASI 
 SÜT

----------------
===LUNCH===
TARHANA ÇORBA
EKŞİLİ KÖFTE
ERİŞTE

----------------
===SALAD/DESSERT===
AYRAN
KIRMIZILAHANA SALATA
ROKALI GÖBEK SALATA

----------------
===FRUIT TIME===
FINDIK& KURU ÜZÜM

----------------

...output omitted...
```

## Handling Web Scraping Challenges

While Scrapy makes web scraping relatively straightforward, you might encounter various challenges in real-world applications. Here are some tips for handling common obstacles:

### Dynamic Websites

Dynamic websites deliver different content to users based on factors like location, device, or preferences. While Scrapy can handle dynamic websites, it's not specifically designed for this purpose.

For dynamic content, consider scheduling your Scrapy spiders to run regularly and implementing a system to track changes over time. In some cases, you may be able to treat infrequently updated dynamic content as essentially static.

### CAPTCHAs

CAPTCHAs are security mechanisms that display distorted text or images that humans must identify to proceed. They're specifically designed to block automated scraping tools.

While our example doesn't use CAPTCHAs, you might encounter them in real-world applications. One approach is to create a Scrapy middleware that downloads the CAPTCHA image and uses OCR (Optical Character Recognition) libraries to convert it to text.

### Session and Cookie Management

Web applications use sessions to maintain state information about users as they navigate through the site. Cookies store information on the user's device that the website can access during subsequent visits.

You may need to maintain session state or manipulate cookies during scraping. Scrapy has built-in support for handling cookies and session management, and additional capabilities are available through third-party libraries.

### IP Blocking

Many websites implement IP blocking to prevent excessive requests from a single source. If a site detects unusual patterns of requests (like those from a scraping bot), it may temporarily or permanently block your IP address.

If you encounter IP blocking, consider using rotating IP addresses or a proxy service to distribute your requests across multiple IPs.

## Summary

For those looking to enhance their web scraping capabilities, Bright Data offers solutions specifically designed for public web data collection. Their [Scrapy integration](https://brightdata.com//integration/scrapy) extends Scrapy's capabilities by using proxy services to help avoid IP bans.

Sign up today to speak with their data experts about your scraping needs.