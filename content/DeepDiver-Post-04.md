---
Title: Multiprocessing and Texblog (Group DeepDiver)
Date: 2022-03-29 16:52
Category: Progress Report
---

By Group "DeepDiver"

While trying to scrap all the comments of a popular product on Amazon, we usually find it time-consuming doing such with only a single-line processing. The reason is that there could be hundreds of pages of reviews and python need to switch pages before extracting the data we need every time. Therefore, we may adopt multiprocessing to make this task faster and easier. 

The multiprocessing tool I used for my web scraping task is the ***parallel*** function from **joblib**:

> *class* `joblib.``Parallel`(*n_jobs=None*, *backend=None*, *verbose=0*, *timeout=None*, *pre_dispatch='2 * n_jobs'*, *batch_size='auto'*, *temp_folder=None*, *max_nbytes='1M'*, *mmap_mode='r'*, *prefer=None*, *require=None*)

This is a very powerful and simple tool which enable us to do multiprocessing with one single line of codes. we just need to predefine out comments extraction process as a function and use comment page as the parameter. Then we can easily loop through all comment pages with the parallel function and a desired number of concurrent scraping process with ***n_jobs*** argument. I will give an example of what I did here:

```python
def storeData(i):

    option = webdriver.ChromeOptions()
    driver_path = r'C:\Users\Bill\Desktop\HKU\7033\chrome\chromedriver.exe'
    service = ChromeService(executable_path=driver_path)
    browser = webdriver.Chrome(service=service,options = option)
    #below is the url of comment page from verified users
    browser.get('https://www.amazon.com/Oral-B-Black-Pro-1000-Rechargeable/product-reviews/B01AKGRTUM/ref=cm_cr_arp_d_viewopt_rvwer?ie=UTF8&reviewerType=avp_only_reviews&pageNumber={}'.format(i))
    time.sleep(0.01)
    result = getRdata(browser)
    return result

sampleData = Parallel(n_jobs = 6)(delayed(storeData)(i) for i in range(500))
```

We can ignore the getRdata function for it is just the function to scrap specific elements on a webpage and it is irrelevant to multiprocessing. I set ***n_jobs*** here as 6 since my laptop has 6 **CPU**s. However, we do need to pay attention to something about the function we put into the ***parallel*** function. **<u>we need to initialize the webdriver inside this function</u>** since the multiprocessing is a process different **CPU**s doing separate tasks so that every time the webdriver must be initialized for the purpose of future data extraction in every line of work. Also, we need to store the parallel result since the what the parallel does is essentially returning a list of results of the input functions with different parameters. 

I also thought about using polarity score from textblob as one of the metrics to support the comment analysis. However, after a bit tryout I decided it is not so reliable upon analyzing the sentiment of reviews. I will show an example here:

![picture group DeepDiver]({static}/images/DeepDiver_post-03_comment.png)

This is apparently a negative comment but the textblob gives it a 0.02 positive polarity score. However, splitting the whole comment into sentences gives us some clues why textblob is somehow misleading:

```python
print(TextBlob("The overall function of the toothbrush is good").polarity)
print(TextBlob("and I like the sensitive teeth bristles (reason for 2 stars)").polarity)
print(TextBlob("but the brush heads do not tightly fit on my toothbrush which results in mold in the brush heads.").polarity)
print(TextBlob("I’ve tried changing the brush heads but it doesn’t help.").polarity)
print(TextBlob("My picture shows how much mold is inside after skipped a single day of cleaning the inside (which shouldn’t be required).").polarity)
print(TextBlob("Very frustrating").polarity)
print(TextBlob("I will say, I got my husband one as well and this isn’t an issue.").polarity)
print(TextBlob("I got unlucky with a design flaw it seems.").polarity)
```

The results are below:

> 0.35
> 0.1
> -0.2
> 0.0
> 0.0642857142857143
> -0.52
> 0.0
> 0.0

However, picking up a few key words from each sentence gives us the same result:

```python
print(TextBlob("overall good").polarity)
print(TextBlob("sensitive").polarity)
print(TextBlob("do not fit").polarity)
print(TextBlob("it doesn’t help").polarity)
print(TextBlob("how much mold a single day").polarity)
print(TextBlob("very frustrating").polarity)
print(TextBlob("this isn’t an issue.").polarity)
print(TextBlob("unlucky with a design flaw").polarity)
```

Besides these words, textblob will simply consider the other input as totally neutral while we know they are not. It really shows that textblob as a sentiment analysis tool can be really arbitrary (relying on a few key words) and non-sensitive. 
