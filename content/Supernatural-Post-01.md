---
Title: Scraping package selection and company selection (Group Supernatural)
Date: 2022-03-29 16:42
Category: Progress Report
---

By Group "Supernatural"

Our group plans to explore the relationship between sentiments of tweets regarding a specific company and the price, volatility or trading volume of its stock. We have currently finished the data collection and cleaning and is diving into analyzing stage. This blog will describe the progress in our project and intended analyzing methods that may be used.

## Scraping tweets from twitter

#### Tweepy

The first package comes into our mind for doing such task is Tweepy. We applied for an [API](https://en.wikipedia.org/wiki/API) successfully and tested the code. 

However, we soon found out that Tweepy is a live stream that always start from the most recent tweets relating to a certain topic and it takes around half an hour to scrape the first 10000 tweets about Tesla. We were also informed that there’s limits when streaming with Tweepy, so it is really hard to scrape tweets in a relatively long time period. However, our project requires at least two months data on twitter comments, so we then turn to other choices.

#### Twint

We then noticed that Twint is another package for scraping tweets, and one can set the time for scarping and the number of tweets to scrape. 

This should be an ideal package but when implement into practice, we encountered several problems. Twint needs a lot of other packages to function together but some of these packages apply only for a specific version of python. This leads to error when running twint code. We also tried to implement package nest_asyncio.apply() as instructed by StackOverflow, it still can’t function well.

#### snscrape

We finally found a website describing package called ‘snscrape’. This package doesn’t require the API from twitter and have several attributes controlling the criterion for scraping. The speed of scraping is also satisfactory. The time to scrape 1000 tweets on Costco is 10 seconds and although the rate would decrease a bit when the number scraped increase, we decided to use this package to scrape twitter.

The code here is to scrape three target companies’ tweets:
```python
tickers = ['Rio Tinto', 'Tesla', 'costco']

for ticker in tickers:
    # Creating list to append tweet data to
    tweets_list = []

    # Using TwitterSearchScraper to scrape data and append tweets to list
    for i,tweet in enumerate(sntwitter.TwitterSearchScraper('{} since:2021-03-01 \
                                                            until:2022-03-01 lang:en'.format(ticker)).get_items()):
        tweets_list.append([tweet.date, tweet.content.encode('utf-8'), tweet.user.username])
    
    # Creating a dataframe from the tweets list above
    tweets_df = pd.DataFrame(tweets_list, columns=['Datetime', 'Text', 'Username'])

    tweets_df.to_csv(path + os.sep + 'Rawdata_{}.csv'.format(ticker))
```

## Company selection

The companies selected are also important for our project, it would decide the quality of tweets we acquire from twitter and thus affect later analysis. 

We tried several companies at first, including Costco, Tesla, Apple, Microsoft etc., but most tweets regarding these companies are advertisements on their products, which will be extremely positive if doing sentiment analysis. It is also interesting to notice that tweets on Elon Mask will always add hashtag Tesla but have nothing to do with Tesla itself. 

After several round of trial, we concluded that companies monopolizing within an industry or companies mainly doing research would be a good choice, since tweets regarding these types of companies are news on some achievements or actions taken. 

We then select three companies: Aramco Ltd, Pfizer and Rio Tinto for further selection. Considering the large number of tweets on Pfizer, it is possible for us to take 2 days to download the texts, which is pretty time-consuming. And the discussion of Aramco is not enough. Finally, we choose Rio Tinto as the target company.

Figures below are tweets on Costco and Pfizer.

###### Tweets on Costco:

![Picture showing Powell]({static}/images/Supernatural-Costco.png)

###### Tweets on Pfizer:


![Picture showing Powell]({static}/images/Supernatural-Pfizer.png)

#### Scraping financial data from website

For collecting financial data about these companies, we select Yahoo Finance as our target website. Package selenium is the tools used. 

First, we download a ChromeDriver into our folder to define the browser and by applying website URL to ‘get’ attribute, the targeted website containing historical data will be opened. 

We then use find_elements_by_xpath to navigate the data we intend to scrape and then successfully download required financial data into our database. Since the factors are in the form of list when downloaded from the website, a database needs to be created for further analysis. 

However, when concatenating these lists, we encountered a problem. As shown below, Yahoo Finance would also record the dividend payment into historical data, but we can not avoid these lines when doing web scraping, so the length of the date list is bigger than other lists of factors. 

###### Historical data on Yahoo Finance:

![Picture showing Powell]({static}/images/Supernatural-Yahoo.png)

After discussion and testing, we found that factors other than opening price are not affected and we can use drop duplicates command in python to delete duplicated dates in the date list, making the date list the same length as factor lists. We finally obtained data frames on financial factors of companies.
