---
Title: Using Selenium to Scrape Customer Reviews (Group DeepDiver)
Date: 2022-03-16 20:29
Category: Progress Report
---

By Group "DeepDiver"

The DeepDiver group tried to analyze customer reviews from different
shopping websites, like [Amazon](https://www.amazon.com/),
[BestBuy](https://www.bestbuy.com/?intl=nosplash),
[Target](https://www.target.com/). We encountered some problems in the
web-scrapting process, and after trial and error, we succeeded in
gathering the data we needed. This blog is about the bugs/problems we
met and how we found solutions, mainly about locating elements and
dealing with dynamic contents.

The whole process was like this: first we initiated Google Chrome via
`ChromeDriver`, loaded the comment page, located the comment title,
content and other information we need, then clicked the next page
button, each time the button was clicked, 8 or 10 more comments were
automatically loaded. We then concated together the comments for each
page and exported to a `.csv` file for use next time.

### Problem 1: The review list was not loaded until the page has scrolled to the bottom

**Solution**: Every time a new page was opened, the browser
automatically scrolled to review list area in the bottom of the page,
which was about 1500px above from the footer of the page. Then the
function slept for a while and waited for the review contents to be
loaded. Otherwise, we still can not get the comment content.

```python
# Scroll down to bottom
driver.execute_script('window.scrollTo(0, document.body.scrollHeight-1500);')
# Wait to load page
RANDOM_SLEEP_TIME = uniform(5, 9)
time.sleep(RANDOM_SLEEP_TIME)
```

### Problem 2: Some reviews only had review title, without content

We saved the review titles and contents into two lists, and found that
the lengths of the two lists were different. The review title list
contained more items than review content list, since some comments
only had titles and no content.

```python
# Gather info on each page
review_dict = {'title': [], 'content': []}
while True:
    # get the reviews' title and content
    review_title = [title.text for title in driver.find_elements(By.CSS_SELECTOR, '[data-hook="review-title"]')]
    review_content = [content.text for content in driver.find_elements(By.CSS_SELECTOR, '[data-hook="review-body"]')]
    # join the original list with '+'
    review_dict['title'] = review_dict['title'] + review_title
    review_dict['content'] = review_dict['content'] + review_content
```

```console
len(review_dict['title'])
Out[32]: 1306
len(review_dict['content'])
Out[33]: 1084
```

**Solution:** Change the data structure, save a single comment into a
dictionary, then insert all the dictionaries of that page into an
array.

```python
reviews_current_page = []
for review in review_list:
    current_review = {} # single review
    current_review['username'] = review.find_element(By.CSS_SELECTOR, '[data-test="review-card--username"]').text
    current_review['title'] = review.find_element(By.CSS_SELECTOR, '[data-test="review-card--title"]').text
    current_review['content'] = review.find_element(By.CSS_SELECTOR, '[data-test="review-card--text"]').text
    current_review['date'] = review.find_element(By.CSS_SELECTOR, '[data-test="review-card--reviewTime"]').text
    current_review['stars'] = review.find_element(By.CLASS_NAME, 'ilnowk').text
    reviews_current_page.append(current_review) # Append reviews to review list
```

### Problem 3: The Class Name of the element contained a dynamic hash, and cannot be obtained by `CLASS_NAME`

```html
<h3 class="Heading__StyledHeading-sc-1mp23s9-0 jCmZWX" data-test="review-card--title" tabindex="-1">Love this toothbrush</h3>
```

**Solution:** The `data-` attribute of the element was fixed, we could
use `CSS_SELECTOR` instead of `CLASS_NAME` to get the element.

```python
current_review['title'] = review.find_element(By.CSS_SELECTOR, '[data-test="review-card--title"]').text
```

### Problem 4: When jumping to the next page, the 'Load 8 more' button was not found

```shell
StaleElementReferenceException: stale element reference: element is not attached to the page document
```

**Solution:** There were two possibilities: one was that the element
had not been loaded, and the other was that the element might have
been removed. We replaced `presence_of_element_located` with
`element_to_be_clickable`, still invalid

```python
from selenium.webdriver.support import expected_conditions as EC

next_page_path = WebDriverWait(driver, RANDOM_SLEEP_TIME).until(EC.element_to_be_clickable((By.XPATH, '//*[@id="pageBodyContainer"]/div[11]/div[8]/button')))
```

Then we found that the button's XPath changed after turning the page,
from `//*[@id="pageBodyContainer"]/div[11]/div[8]/button` to
`//*[@id="pageBodyContainer"]/div[10]/div[8]/button` . So XPath cannot
be used, but `<button>` itself did not have an obvious `class name`,
only its parent element has the `data-test="load-more-btn"` attribute,
so the `CSS SELECTOR` was used, first located the parent element, and
then used `>` to get child element button.

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
next_page_path = WebDriverWait(self.driver, RANDOM_SLEEP_TIME).until(EC.presence_of_element_located((By.CSS_SELECTOR, '[data-test="load-more-btn"]>button'))) 
```

### Problem 5: Empty elements when crawling

```shell
NoSuchElementException: Message: no such element: Unable to locate element: {"method":"css selector","selector":"[data-hook="helpful-vote-statement"]"}
```

**Solution:** If the comment was "0 guest found this helpful", then
this text would not be displayed, so the program reported an
`NoSuchElementException error`. We needed to add a `try except` to
deal with the empty element. If this element was not found, fill this
item manually.

```python
try:
    current_review['vote'] = review.find_element(By.CSS_SELECTOR, '[data-hook="helpful-vote-statement"]').text
except NoSuchElementException:
    current_review['vote'] = '0 people found this helpful'
```
