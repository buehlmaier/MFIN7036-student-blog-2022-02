---
Title: Obtaining Data through Webscraping (Group Processors)
Date: 2022-03-21 12:25
Category: Progress Report
---

By Group "Processors"

# Text Sources

The first step of our project is to obtain bitcoin and cryptocurrency related text data. We mainly focus on two types of text data - social media and news articles. For social media, we make use of the following sources:

1. [Conversation under BTC-USD at Yahoo Finance](https://finance.yahoo.com/quote/BTC-USD/community/)
2. [Reddit](https://www.reddit.com/)
3. [Bitcointalk](https://bitcointalk.org/)
4. [Twitter](https://twitter.com/?lang=en)

While for news article, we obtain from the following sources:

1. [Crypto News](https://cryptonews.com/)
2. [Cryptocurrency News at Yahoo Finance](https://finance.yahoo.com/topic/crypto/)

# Price data Sources

The price data related bitcoin price we obtain from the below website:

1. [CoinMarketcap](https://coinmarketcap.com/currencies/bitcoin/historical-data/)

# Tool

To extra data from the above mentioned website, we utilize web scaping with the tool of `WebDriver` from the `Selenium` library along with a little bit of `BeautifulSoup`.

# Web Scraping

### Conversation under BTC-USD at Yahoo Finance

The default way to display the comments on this website is by popularity, but because we want to obtain all comments within a certain timeframe, we have to first change the display method to by publish time.

```python
# display discussion by reaction time from the most recent to the least
browser.find_element_by_xpath('//*[@id="canvass-0-CanvassApplet"]/div/div[3]/button').click()
browser.find_element_by_xpath('//*[@id="canvass-0-CanvassApplet"]/div/div[3]/ul/li[2]/button').click()
```

The initial page only shows 20 comments. To access more comments, we need to click the 'Show more' button to load 20 extra each time. We decide to load all the comments within the desired timeframe on the website before scrapping them. We are not able to come up with a smart way to have the computer to stop itself automatically when it reaches a specific date so we just go with the stupid one instead. The stupid way is to write a loop to click the button 300 times, then we manually check the date of the last comment. If it has not gotten to the desired date, we would have the button clicked for another 300 times. This process would be repeated until we have all the data we wanted. 

```python
# click the "Load more" button for 300 times
for i in range(300): 
    browser.find_element_by_xpath('//*[@id="canvass-0-CanvassApplet"]/div/button').click()
    i = i + 1
    time.sleep(uniform(1,5)) # add random time sleep to avoid being detected by the website
    
# after the certain clicks, the xpath of the button changes, we then have to use the updated xpath
for i in range(50): 
    browser.find_element_by_xpath('//*[@id="canvass-0-CanvassApplet"]/div/button[2]').click()
    i = i + 1
    time.sleep(uniform(1,5))  
```

After finish loading all the comments, we scrap the content, date, and author of these comments and save them into a dataframe.

```python
Author = browser.find_elements_by_xpath('//*[@id="canvass-0-CanvassApplet"]/div/ul/li[position()>=1]/div/div[1]/button')                                                                         
Author = list(map(lambda x: x.text, Author))
    
Con = browser.find_elements_by_xpath('//*[@id="canvass-0-CanvassApplet"]/div/ul/li[position()>=1]/div/div[2]')
Con = list(map(lambda x: x.text, Con))
      
Time = browser.find_elements_by_xpath('//*[@id="canvass-0-CanvassApplet"]/div/ul/li[position()>=1]/div/div[1]/span/span')                                                                                               
Time = list(map(lambda x: x.text, Time))

df = pd.DataFrame({'author': Author, 'date': Time, 'content': Con})
df = df[(df['content'] != '') & (df['date'] != 'last month')] # we only want the comments from the past 30 days and we don't want comments only contain picture without any text.
df = df.reset_index(drop = True)
```

Only the comments posted in less than 24 hours would have an exact time and all others only have the date as 'x day ago', but since we would like to keep the format of the date organized, we change all the date into the 'x day ago' format.

```python
for i in range(0,len(df['date'])):
    if 'hour' in df['date'][i]:
        df['date'][i] = '0 day ago'
    elif 'minutes' in df['date'][i]:
        df['date'][i] = '0 day ago'
    elif 'yesterday' in df['date'][i]:
        df['date'][i] = '1 day ago'
```

After changed all the date into the 'x days ago' format, I changed all the date into '%Y-%m-%d' format in order to match the date format in to price data.
```python
for i in range(0,len(df['date'])):
    days_age = int(df['date'][i][:2])
    df['date'][i] = (datetime.now() - timedelta(days_age)).strftime('%Y-%m-%d')

```


### Reddit

Redidt is an American social news aggregation, but unlike normal news sites he will stream a comment section under each news item for people to discuss freely. Thus, we scarpe comments to learn people's attitude towards latest news.

At the beginning of the data crawl, we chose to search for relevant posts using bitcoin as a keyword. Then, recording all the headlines and comments of the searched result. However, while browsing through the text results, we noticed that some news headlines and comments were not protected by the keywords in all the results. This made us suspicious of the crawl results. 

Later, after searching, we found that most of the keywords searched were contained in the content of the news content, not the headline. Besides, many news contents are more important than the headlines. 

However, because the news content is too long, adding them to the original comments storage file always reports an error indicating that the length is exceeded. Therefore, we rewrote the code and decided to generate two txt files. We wrote two functions that generate text documents, and one of them holds the news contents and the other holds the title and each comment.

```python
def save_comments_data(keyword, post, comment_list):
    with open("data/" + keyword + ".txt", "a", encoding='utf-8') as file:
        for comment in comment_list:
            file.write(post + "\t" + comment + "\n")

def save_contents_data(keyword, txt_list, contents_list, time_list):
    with open("data/" + keyword + "_contents.txt", "a", encoding='utf-8') as file:
        for i in range(len(txt_list)):
            file.write(txt_list[i] + "\t" + contents_list[i] + "\t" + time_list[i] + "\n")

```

Since the site could not load all relevant news on the same page, we needed to keep clicking the go to next page button to make sure the program could navigate to all relevant news.

```python
res_urls = soup.findAll('span', class_=re.compile("nextprev"))

```

After that, we also need to add new parameters in order to get the time of each news release and find them all the time by looping through the implementation. We get the web address, the exact time, and the related content of each news item through a self-defined function.

Among other things, we initially had some difficulty in getting a specific time. This is because, when browsing the web, we find that the web page only shows how many days ago the news was published, and there is no way to get the exact time of publication of each article. When querying its html code, the exact time when this news was published is known after the datetime indicator.

```python
def getPageData(browser, search_url):
    txt_list = []
    url_list = []
    time_list = []
    js = "window.open(\"" + search_url + "\")"
    
    # open link in a new browser window
    browser.execute_script(js)  
    time.sleep(3)

    # move the handle to operate on the newly opened page
    browser.switch_to.window(browser.window_handles[-1])  
    soup = BeautifulSoup(browser.page_source, "lxml")
    res_contents = soup.findAll('div', class_=re.compile("listing search-result-listing"))
    
    if len(res_contents) > 0:
        res_content = res_contents[-1]
        res_times = res_content.findAll('time')
        for res_time in res_times:
            time_list.append(res_time['datetime'])

        res_posts = res_content.findAll('a', class_=re.compile("search-title may-blank"))
        
        for post in res_posts:
            text = "".join(list(post.strings))
            url_list.append(post['href'])
            txt_list.append(text)
        print(len(txt_list), txt_list)
        print(len(url_list), url_list)
        print(len(time_list), time_list)
        res_urls = soup.findAll('span', class_=re.compile("nextprev"))
        
        if len(res_urls) > 0:
            res_url = res_urls[-1].findAll('a')[-1]
            
            if res_url.string == "下一頁 ›":
                
                return txt_list, url_list, time_list, res_url['href']
        
        # adjust the scroll bar to the bottom of the web page
        scroll_to_bottom(browser)  
        return txt_list, url_list, time_list, None

```

After that, we decided to crawl mainly two forums on reddit related to bitcoin. And with the above custom function, a loop was formed. We decided to crawl mainly two forums on reddit related to bitcoin. And with the above custom function, a loop was formed. Eventually aggregated into a function that crawls all messages.

```python
def findAllData(keywords):
    browser = webdriver.Chrome("chromedriver.exe")
    browser.implicitly_wait(10)
    # url = "https://www.quora.com/"
    # browser.get(url)
    # time.sleep(3)
    for keyword in keywords:
        if os.path.exists("data/" + keyword + ".txt"):
            os.remove("data/" + keyword + ".txt")
        search_url = "https://old.reddit.com/r/CryptoCurrency/search/?q=" + keyword + "&sort=top&t=year"
        txt_list, url_list, time_list, res_url = getPageData(browser, search_url)
        getPageComments(browser, keyword, txt_list, url_list, time_list)
        while res_url is not None:
            txt_list, url_list, time_list, res_url = getPageData(browser, res_url)
            browser.close()
            browser.switch_to.window(browser.window_handles[0])
            getPageComments(browser, keyword, txt_list, url_list, time_list)
            browser.close()
            browser.switch_to.window(browser.window_handles[0])
            # time.sleep(3)
            # break
    browser.close()

```


### Bitcointalk

Bitcointalk.org is a forum where bitcoin followers are actively posting their discussions 
around various topics like bitcoin trading and technical development. To align with the timeline
of our project, we scrape the threads under the 'Bitcoin Discussion' section of the website for the last 30 days.

However, the threads all contain quotes from before. We thus decide to filter out all the threads we do not need,
and see if the term frequency might give a more accurate and appropriate result. 

The web scraping includes both selenium and beautifulsoup, since this makes it easier to locate the posts.
But one limit is that during the craping, the new discussions will not be able to be captured at the time and they
might also change the order of the discussion topics, which could result in omission or repetition. We hope that this
would be resolved during data cleaning.

```python
# Use number of replies to calculate the total pages of posts (easier to go to next page).
    # Rather than going through all the pages, displaying all discussions is also an explicit way. 
    # But the website is not responding well for some discussion topics
        num_of_replies = browser.find_element_by_xpath('/html/body/div[2]/div[2]/table/tbody/tr[{}]/td[5]'.format(order))
        pages = math.ceil(int(num_of_replies.text) / 20) 
    
    # Choose a Discussion Topic
        open_discussion = browser.find_element_by_xpath('/html/body/div[2]/div[2]/table/tbody/tr[{}]/td[3]/span/a'.format(order))
        if 'Bitcoin puzzle transaction ~32 BTC prize to who solves it' in open_discussion.text: # the discussion under this topic contains too many threads and is irrelevant to our project.
            continue
        open_discussion.click()
        time.sleep(3)

    # Extract Posts (easier to locate using beautifulsoup)
        searchweb = rq.get(str(browser.current_url)).text 
        soup = bs(searchweb)
        text = soup.find_all('td','td_headerandpost') 
        for i in text:
            postlist.append(i.get_text())
    
    # Go Through All the Pages
        if pages > 1:
            for page in range(2,pages+1):
                All_link = browser.find_element_by_link_text(str(page))
                All_link.click()
                time.sleep(1)
                searchweb = rq.get(str(browser.current_url)).text 
                soup = bs(searchweb)
                text = soup.find_all('td','td_headerandpost')
                for i in text:
                    postlist.append(i.get_text())
            
    # Return to the discussion list page.
        browser.get(url)
```


### Twitter

We are using twitter APi to access text data on twitter. We decide to obtain tweets created since 2022-02-16, with the keyword 'bitcoin', simple retweets not included.
Also, as required, we only choose the tweets in English.

```python

client = tweepy.Client(bearer_token = bearer_token)
query = 'bitcoin -is:retweet lang:en'
tweets = client.search_all_tweets(query = query,max_results = 500, tweet_fields =['created_at'], start_time = '2022-02-16T00:00:01Z')

```

### Crypto News

Since the news at Yahoo Finance covers only the latest news dating back to only 7 days ago, I think this time range is too short and some other earlier news need to be included as well. Under Cryptonews - News - Bitcoin News (https://cryptonews.com/news/bitcoin-news/), a section called 'All News' collected Bitcoin news initially shows 16 news. A button below called 'Load more news...' need to be clicked to show 8 more news. In this case, considering that sometimes the click fails, I looped the try-except click function for 60 times to get enough news dating back to February 9, 2022. 

```python

cryptonews_path = 'https://cryptonews.com/news/bitcoin-news/'
browser.get(cryptonews_path)
for i in range(0, 60):
    try:  
        time.sleep(6)
        browser.find_element_by_id('load_more').click()
    except:
        i = i - 1

```

After getting enough news we want, I made a loop to scrape the headline, content, and the time of the news. All these information of one single news are stored in a dictionary. This dictionary is then appended to a list of dictionaries. 

```python
news_list = []
xpath_list = browser.find_elements_by_xpath('//*[@id="load_more_target"]/div[position()>=1]/article/div/div[2]/a')
for i in range(0, len(xpath_list)):
    link = xpath_list[i].get_attribute('href')
    browser_news.get(link)
    news = {'headline': browser_news.find_element_by_css_selector('h1.mb-40').text, 
           'content': browser_news.find_element_by_xpath('/html/body/main/div[2]/div[2]/div[2]/div[2]').text, 
           'time': browser_news.find_element_by_class_name('fs-12').text.split('·')[0]}
    news_list.append(news)
```
A copy of the scrapped data is then made and then formatted into a .csv file, source of these news is labelled as "Cryptonews" to enable future identification.  

###Cryptocurrency News at Yahoo Finance

Under the News section on Yahoo Finance, we specifically focused on those under tab cryptocurrencies news. The initial website shows only 20 latest cryptocurrencies and bitcoin related news. To get a full record of all those news, we need to scroll down to the bottom of the web page and let the page to auto-load. Since a total of 150 news is expected, I made a scroll-to-bottom loop for 15 times to get the full page. 

```python
yahoo_path = 'https://finance.yahoo.com/topic/crypto/'
browser.get(yahoo_path)
for i in range (0, 15):
    # Scroll down to bottom
    browser.execute_script("window.scrollTo(0, 1000000);")
    time.sleep(3)
```

After loading all the data, I developed a loop to click into each headline to extract the headline, content and time of the news. Each news with these three attributes are stored in a dictionary and then appended to a list containing all news dictionaries. Something worth noticing is that some news have button 'Story Continues', which need to be clicked for all content to be shown. In this case, a try-except command is included in the loop to ensure full content of a news is shown. 

```python
news_list = []
xpath_list = browser.find_elements_by_xpath('//*[@id="Fin-Stream"]/ul/li[position()>=1]/div/div/div[2]/h3/a')
for i in range(0, len(xpath_list)):
    link = xpath_list[i].get_attribute('href')
    browser_news.get(link) 
    try:
        browser_news.find_element_by_xpath("//button[contains(., 'Story continues')]").click()
    except:
        pass
    news = {'headline': browser_news.find_element_by_class_name('caas-title-wrapper').text, 
           'content': browser_news.find_element_by_class_name('caas-body').text, 
           'time': (browser_news.find_element_by_class_name('caas-attr-time-style').text).split('·')[0]}
    news_list.append(news)
```

Then I made a copy of the scraped data and output it into a .csv form. I labeled the source as "Yahoo Finance" to distinguish news from different sources if we further merge all data together. 

```python
yahoo_news_150 = news_list
headline = []
content = []
time_list = []
source = []
for news in yahoo_news_150:
    headline.append(news['headline'])
    content.append(news['content'])
    time_list.append(news['time'])
    source.append('Yahoo Finance')
news_df = pd.DataFrame({'Headline': headline, 'Content': content, 'Time': time_list, 'Source': source} )
news_df.to_csv(current_path + '/Yahoo Finance cryptonews.csv')
```

###Bitcoin price data from coinmarketcap

The initial page only shows 60 days price. To access more comments, we need to click the \textbf{Show more} button to load more each time. We decide to load all the comments within the desired timeframe on the website before scrapping them. We are not able to come up with a smart way to have the computer to stop itself automatically when it reaches a specific date so we just go with the stupid one instead. The stupid way is to write a loop to click the button 11 times, then we manually check the date of the last price data. This process would be repeated until we have all the data we wanted.

```python
df = pd.DataFrame()
path = 'https://coinmarketcap.com/currencies/bitcoin/historical-data/'
browser.get(path)
time.sleep(20)
    
for i in range(11):
    element = browser.find_element_by_xpath( '//*[@id="__next"]/div[1]/div[1]/div[2]/div/div[3]/div/div/p[1]/button')                                          
    element.click()
    i = i + 1
    time.sleep(10)
```

After finish loading all the price data, we scrap the date, open price, close price, highest price, lowest data, volume and Market Cap of the price date and save them into a dataframe.

```python
recorded_date = browser.find_elements_by_xpath('//*[@id="__next"]/div[1]/div[1]/div[2]/div/div[3]/div/div/div[2]/table/tbody/tr[position()>=1]/td[1]')
rec_date = list(map(lambda x: x.text, recorded_date))
df['Date'] = rec_date

open_price = browser.find_elements_by_xpath('//*[@id="__next"]/div[1]/div[1]/div[2]/div/div[3]/div/div/div[2]/table/tbody/tr[position()>=1]/td[2]')
open_price = list(map(lambda x: x.text,open_price))
df['Open'] = open_price
    
high = browser.find_elements_by_xpath('//*[@id="__next"]/div[1]/div[1]/div[2]/div/div[3]/div/div/div[2]/table/tbody/tr[position()>=1]/td[3]')
high = list(map(lambda x: x.text,high))
df['High'] = high

low = browser.find_elements_by_xpath('//*[@id="__next"]/div[1]/div[1]/div[2]/div/div[3]/div/div/div[2]/table/tbody/tr[position()>=1]/td[4]')
low = list(map(lambda x: x.text,low))
df['Low'] = low

close = browser.find_elements_by_xpath('//*[@id="__next"]/div[1]/div[1]/div[2]/div/div[3]/div/div/div[2]/table/tbody/tr[position()>=1]/td[5]')
close = list(map(lambda x: x.text,close))
df['Close'] = close

volume = browser.find_elements_by_xpath('//*[@id="__next"]/div[1]/div[1]/div[2]/div/div[3]/div/div/div[2]/table/tbody/tr[position()>=1]/td[6]')
volume = list(map(lambda x: x.text,volume))
df['Volume'] = volume


Market_Cap= browser.find_elements_by_xpath('//*[@id="__next"]/div[1]/div[1]/div[2]/div/div[3]/div/div/div[2]/table/tbody/tr[position()>=1]/td[7]')
Market_Cap = list(map(lambda x: x.text,Market_Cap))
df['Market Cap'] = Market_Cap
return df
```



