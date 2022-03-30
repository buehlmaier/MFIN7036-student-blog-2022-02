---
Title: NLP_blog1 (Group N95 Limited Partner)
Date: 2022-03-29 19:30
Category: Progress Report
---

By Group "N95 Limited Partner"

# #Group objective

Our group project is 'Stock price prediction and trading strategy creation based on company disclosures'.
Our goal is to create a tool that can collect text data from company disclosure and 
analyze the impact of company disclosures on stock prices.
We try using NLP to analysis text data from company disclosure. Then we will try 
multiple deep learning algorithms to find out the potential relationship between company disclosure 
and change of stock price , and we will compare their performance on the same data.
The whole process of our project is dividied into 5 steps which are data collection, 
data cleansing and preprocessing, data labelling, model construction and comparison and investment strategy and backtesting.

# #NLP first Blog 

This is the first blog of N95 Limited Partner. 
In the first blog, we would like to share our experience in the first step data collection.
We faced many challenges when doing this part, but we overcame them.

## Data collection

The most important tool we use to collect data is web-scraping. 
In this part, we use webdriver to help us  get the infromation from and download 
the document of companies disclosure from [SEC](https://www.sec.gov/edgar/search-and-access)

First, we start with reading the company list of S&P500.

```python
all_companies = pd.read_excel("SP500Companies.xlsx")
all_companies

```

![S&P500 companies]({static}/images/NLP_SP500_companies.png)


 Then we try to pick tickers of the companies that we want to analyze.

```python
analyzed_companies = all_companies.iloc[31:33]
tickers = analyzed_companies["trade_code"]
for ticker in tickers:
    ticker = ticker.strip()
    print(ticker)
```

Set the webdriver and accesss the website[SEC](https://www.sec.gov/edgar/search-and-access)

```python
chromedriver_path = r"C:\Users\baris\chromedriver.exe"
driver = webdriver.Chrome(executable_path=chromedriver_path)
driver.get("https://www.sec.gov/edgar/search-and-access")
```

We didn't have any difficulty in the above steps. However, challenges 
arose when we tried to download company files automatically

## Problem-01:  How do we limit the range of files we can download? 

**Solution**:  The selection of range of company disclosure require adjusment of setting on the website.
In order to set the category and date of the company disclosure we want, I use the function click() and 
time.sleep(). The function 'click()' help us click the buttom we find in the website by xpath. And the function 
time.sleep() provides time for web page jump. 

```python
driver.find_element_by_xpath('//*[@id="block-secgov-content"]/article/div/div/div[2]/div/div/div/div/div/div/ul[1]/li/a').click()
time.sleep(1)
driver.find_element_by_xpath('//*[@id="show-full-search-form"]').click()
time.sleep(1)
driver.find_element_by_xpath('//*[@id="category-select"]').click()
time.sleep(2)
driver.find_element_by_xpath('//*[@id="category-type-grp"]/ul/li[3]').click()
time.sleep(1)
driver.find_element_by_xpath('//*[@id="date-range-select"]/option[1]').click()
time.sleep(1)
```

Set the save path.

```python
save_path = "data"
if not os.path.exists("data"):
    os.makedirs("data")
```

## Problem-02:  The number of sample companies is 31. 
We collected disclosure documents of various companies from different links. 
How to classify documents from different companies and store them in folders with 
different company names was a challenge

**Solution**:  We use function os.makedirs() to create a folder directory. For any company that are not 
in the directory, we create a new path for it.   
And we keep different years of company disclosures from the same company in the same folder named by 
its ticker.

```python
save_path = "data"
if not os.path.exists("data"):
    os.makedirs("data")

for ticker in tickers:
    ticker = ticker.strip()
    directory_path = save_path + f"/{ticker}"
    if not os.path.exists(directory_path):
        os.makedirs(directory_path)
```


## Problem-03: Some company disclosure documents are not available. 
When a missing file is encountered, the program is likely to report an error and stop.
How do we deal witn them?

**Solution**:   We use different discriminant statements to confirm the data state.
We also use 'try and except' to maintain the process without and interruption.


## Problem-04:  How do we get the text data and save it in our computers?  

**Solution**:  The website does not let to apply beautiful soup to extract the tags of the document.
In addition, the website did not le the manupulation of desired texts and sections before downloading.
To overcome the situtation, we used multiple functions and steps to achieve it. We first store the text data in a project body_text. 
Then we apply function from os package to write the data on our own text files. 


```python 
while (True):
        documents = driver.find_elements_by_xpath('//*[@id="hits"]/table/tbody/tr')
        for num,doc in enumerate(documents):
            if (re.match('^10-K ',doc.text)) or (re.match(' ^10-K ',doc.text)):

                filled_date = doc.find_element_by_class_name("filed").text

                driver.find_element_by_xpath(f'//*[@id="hits"]/table/tbody/tr[{num+1}]/td[1]/a').click()
                new_page = driver.find_element_by_xpath('//*[@id="open-file"]/button').click()
                driver.switch_to.window(driver.window_handles[0])
                driver.find_element_by_xpath('//*[@id="previewer"]/div/div/div[3]/button').click()

                driver.switch_to.window(driver.window_handles[-1])
                time.sleep(1)
                body_text = driver.find_element_by_xpath('/html/body').text
                time.sleep(1)
                driver.close()
                time.sleep(1)

                if (len(driver.window_handles) > 1):
                    driver.switch_to.window(driver.window_handles[-1])
                    time.sleep(1)
                    driver.close()


                driver.switch_to.window(driver.window_handles[0])
                complete_path = os.path.join(directory_path,f'{filled_date}.txt')
                with open(complete_path,"w",encoding="utf-8") as f:
                    f.write(body_text)

            else:
                continue
        try:
            driver.find_element_by_xpath('//*[@id="results-pagination"]/ul/li[12]/a').click()
            time.sleep(5)
            continue
        except:
            break
            
```

## Conclusion

In this blog, we show the process of webscarping. The outout of this part is a folder with sub-folders containg
the text data of all sample companies. 
It is the basic of our project.

![data]({static}/images/NLP_data.png)

