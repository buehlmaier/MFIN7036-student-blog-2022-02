---
Title: Web Scraping on Best Buy Website (Group DeepDiver)
Date: 2022-03-29 13:46
Category: Progress Report
---

By Group "DeepDiver"

## Simulation Request

We need to find the pattern of parameters in the request, construct and initiate the request ourselves.

```python
def fetchurl(page, sku):
    #url
    url = "https://www.bestbuy.com/ugc/v2/reviews"    #all review in this url, different sku
    #request header
    headers = {
        "user-agent" : "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/98.0.4758.109 Safari/537.36"
        }
    #parameters
    params = {
        "page": page,
        "pageSize": 20,
        "sku": sku,
        "sort": "MOST_RECENT",
        }

    r = requests.get(url, headers=headers, params=params)
    return r.json()
```

There are some request parameters in request head, we can manually create each parameter to obtain the information we want. "page" means the current review page, "pageSize" means the total review numbers in one page, "sku" means the unique number for each products, therefore, if you want to obtain the reviews of different products, you just need to modify the sku number. "sort" means the presentation order of reviews.

## Parse data

After we successfully simulate the request, we need to parse the data we obtained. 

```python
def parseJson(jsonObj):
    data = jsonObj["topics"]

    reviewData = [] 
    for item in data:
        #review author
        review_author = item["author"]
        #review rating
        rating = item["rating"]
        #review title
        title = item["title"]
        #review content
        content = BeautifulSoup(item["text"], "html.parser").text
        #review time
        create_time = item["submissionTime"]

        dataItem = [create_time, review_author, title, content, rating]
        reviewData.append(dataItem)

    return reviewData
```

The returned data is in json format, we can use python's own json library to parse it and get the data we want in this order: author, rating, title, review content, create time

## Save data

After parse data, we need to save it into csv file for further processing

```python
def save_data(data, path, filename):

    if not os.path.exists(path):
        os.makedirs(path)

    dataframe = pd.DataFrame(data)
    dataframe.to_csv(path + filename, encoding='utf_8_sig', mode='a', index=False, sep=',', header=False )
```

## Main Function

Successfully defined all the function we need, we could run the web scrapping program to obtain the product reviews. Here, we use time.sleep to avoid the block from the website

```python
if __name__ == "__main__":
    skuList = [1493375, 6485362]
    #Oral-B Pro 1000, Philips Sonicare 4100, 
    for sku in skuList:
        startPage = 1
        endPage = 63
        path = "/Users/wangqingran/Desktop/Python course/7036 NLP/Group project/scraping_review_data/"
        filename = "reviews-" + str(sku) + ".csv"

        csvHeader = [["create_time", "review_author", "title", "content", "rating"]]
        save_data(csvHeader, path, filename)

        for p in range(startPage, endPage+1):
            print("\rNow scraping page ", str(p), " ...", end="")
            html = fetchurl(p, sku)
            review = parseJson(html)
            save_data(review, path, filename)
            time.sleep(uniform(0.2, 0.6))
```
