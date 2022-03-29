---
Title: Tricks and Techniques in Improving Web-scraping Efficiency (Group Tower of Babel)
Date: 2022-03-29 13:22
Category: Progress Report
---

By Group "Tower of Babel"

Our group project aims at analyzing the customer reviews on e-commerce platform. In order to extract review data from the e-commerce website, we use Selenium to conduct the web-scraping process.
Since web-scraping is time-consuming, especially when a large scale dataset is required by some research purposes, here we want to share several useful tricks that can improve the efficiency of scraping information.

The target website is [amazon](http://https://www.amazon.com/product-reviews/B08SP5GYJP) . Let's take one self-empty product as an example.
We use Chrome as our browser.
The pakage and modules we need:
```python
from selenium import webdriver
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.by import By
```

## 1 Browser Setting arguments

As beginners, normally we just import webdriver.exe and start "get(url)" to start a scraping process.
However, we'd like to introduce some automatic initial changes to the browser which can substitude manual settings.
Before "get(url)", use the following code to add options:

```python 
options = webdriver.ChromeOptions()
#setting the language to English in case you were in non-english country
options.add_argument("--lang=en-US")
#close the tips
options.add_experimental_option('excludeSwitches', ['enable-automation'])
#forbid showing image and css format 
prefs = {"profile.managed_default_content_settings.images": 2,'permissions.default.stylesheet': 2}
options.add_experimental_option("prefs", prefs)
```

As shown in the figure, the browser won't loading the figure, thus saving much time for us. 

![Picture of group tower of babel]({static}/images/Tower-of-Babel-Post-01-image-01.jpg)


## 2 WebDriverWait

WebDriverWait is one of the explicit wait methods.([What is explicit wait and implicit wait?](https://selenium-python.readthedocs.io/waits.html))
As the instrution mentioned,

>An explicit wait is a code you define to wait for a certain condition to occur before proceeding further in the code.

So how WebDriverWait can help in scraping?
We want to scrape all the reviews showing in one page, but sometimes the scraping code was executed just before the website was fully loaded.
In such case, python will stop running and report "NoSuchElementException" Error.
To address such issue, we write a while+try clause with WebDriverWait:

```python
# use a loop to locate review elements

while True:
    try:
        WebDriverWait(driver, 30).until(EC.presence_of_element_located((By.ID,'cm_cr-review_list')))
        break
    except Exception as e:
        print(e)
        driver.refresh()
us = driver.find_element_by_id("cm_cr-review_list")
```

Although the clause for WebDriverWait is a bit complex, it is more useful than "time.sleep()" for it can be combined with other selenium modules to achieve specific goals.
As you see, we combine it with the module "expected_conditions.presence_of_element_located()" (i.e. EC in the code)
The same logit applies to locating and clicking the button.

```python
#let the webdriver wait for specific elements, the maximum waiting time is 100s
#the element we need is a 'next page' button 

#once find the button, just click it
try:
    WebDriverWait(driver, 5).until(EC.element_to_be_clickable((By.XPATH, '//li[@class = "a-last"]/a')))
    next_page = us.find_element_by_xpath('.//li[@class = "a-last"]/a').get_attribute("href")

#if there is not such button, just print "no next page" and end it 
except NoSuchElementException:
    driver.find_elements_by_xpath('//li[@class = "a-disabled a-last"]')
    next_page = None
    print("no next page")
    
#if timeout, just refresh
except TimeoutException:
    next_page = None
    print("timeout")
    driver.refresh()
print("this page is finished")

```


## 3 Methods to automatically scroll down web-page
The method in #2 applies to webistes that contain limitted reviews. 
As for some other websites, there maybe a "read more" option, or they need manual scroll down operatin to show more information.
So the following content shows some other ways to scroll down the page.
Once you srcoll down for enough times, you can start scraping information.

(1) One simple way is to window.scrollBy() argument.

```python
numb = 5    # define how many times it needs to srcoll

for i in range(number):
    browser.execute_script("window.scrollBy(0, 1000)")
    time.sleep(2)   # leave enough time between two scroll operations
```

(2) Use send_keys to send tap to imitate manual tapping.

```python
numb = 5  # define how many times it needs to srcoll

for i in range(numb):
    browser.find_element_by_tag_name('body').send_keys(Keys.END)   #send tap to the webpage
    time.sleep(1)
    
```

(3) click buttons like "read more" for several times and tap.

```python
numb =5

for i in range(numb):
    read_more = driver.find_element_by_xpath('//a[text()="read_more"]')   #find read more button
    browser.find_element_by_tag_name('body').send_keys(Keys.END)

```





