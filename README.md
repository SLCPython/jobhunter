We will be building and deploying a python script to scrape sites which will end up looking like this:

run:

`python3.12 -m venv .venv && . .venv/bin/activate`

then 

`pip install -r requirements.txt`

then

`playwright install`

finally:


`python spiders/google_job_hunt.py`


```python
import re
import scrapy
from scrapy.crawler import CrawlerProcess
from scrapy.selector import Selector


class GoogleSpider(scrapy.Spider):
    name = 'google_spider'
    allowed_domains = ['www.google.com']
    custom_settings = {
        "TWISTED_REACTOR": "twisted.internet.asyncioreactor.AsyncioSelectorReactor",
        "DOWNLOAD_HANDLERS": {
            "https": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
            "http": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
        }
    }

    def __init__(self, domain, stop, user_agent, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.domain = domain
        self.stop = int(stop)
        self.custom_settings['USER_AGENT'] = user_agent
        self.start_urls = [
            f'https://www.google.com/search?q=intitle%3A%28%22Data+Scientist%22+OR+%22Data+Engineer%22+OR+%22Machine+Learning%22+OR+%22Data+Analyst%22+OR+%22Software+Engineer%22%29+Remote+-%22Director%22+-%22Principal%22+-%22Staff%22+-%22Frontend%22+-%22Front+End%22+-%22Full+Stack%22+site%3A{self.domain}%2F%2A+after%3A2023-03-27']
        self.urls_collected = []

    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        return super().from_crawler(crawler, *args, **kwargs)

    def start_requests(self):
        yield scrapy.Request(self.start_urls[0], meta={"playwright": True,
                                                       "playwright_include_page": True})

    async def get_page_info(self, page):
        for i in range(10):
            val = page.viewport_size["height"]
            await page.mouse.wheel(0, val)
            await page.wait_for_timeout(1000)
        text = await page.content()
        selector = Selector(text=text)
        urls = []
        for row in selector.xpath("//div[contains(@class, 'kCrYT')]"):
            text = row.xpath(".//h3//text()").get()
            url = row.xpath(".//a/@href").get()
            if url:
                urls.append({text: url})
                print(urls)
        self.urls_collected += urls
        return urls

    async def parse(self, response):
        page = response.meta['playwright_page']
        urls = await self.get_page_info(page)
        found = True
        while found:
            try:
                element = page.get_by_text("Next")
                print(element, "parsing next page")
                await element.click()
                more_urls = await self.get_page_info(page)
                urls += more_urls
            except:
                found = False
        return urls


def main(domain, stop, user_agent):
    process = CrawlerProcess()
    process.crawl(GoogleSpider, domain=domain, stop=stop, user_agent=user_agent)
    process.start()


if __name__ == '__main__':
    domain = 'jobs.lever.co'
    stop = 25
    user_agent = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.3'
    user_agent2 = "Opera/9.80 (Windows NT 5.1; U; MRA 5.5 (build 02842); ru) Presto/2.7.62 Version/11.00"
    user_agent3 = "Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.2; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0)"
    main(domain=domain, stop=stop, user_agent=user_agent3)
```

