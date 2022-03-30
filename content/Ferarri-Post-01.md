---
Title: Scraping and preprocessing earning calls from seekingalpha.com (Group Ferrari)
Date: 2022-03-30 12:00
Category: Progress Report
---

By group "Ferrari"

## Scraping earning calls
Our group did some analysis in Earning calls of Top 50 companies on SP500. We collected the data from [seekingalpha.com](https://seekingalpha.com/).

When we scraped data from the website, the anti-scrape on the website caused some trouble. We used selenium to scrape data and the website will block our bot after visiting some pages. The website will show a window as below, which requires the visitor to prove it’s not a bot. This was not easy for the program to handle. Even after proving by hand as the website required, this window will show up again on the next visit.

![Picture showing Powell]({static}/images/Ferarri_pop_ups.png)

We searched online and try to avoid this window showing up. One of the measures helped us a lot. The related code:
```
option = webdriver.ChromeOptions()
option.add_experimental_option("excludeSwitches",['enable-automation'])
option.add_argument("disable-blink-features=AutomationControlled")

browser = webdriver.Chrome(options=option)
```

The measure is adding some options before creating the chrome webdriver, which made the program more like a human and not easy to be detected as a bot. After that, we rarely see that windows. And there are more measures to hide the bot on [How to make Selenium undetectable and stealth](https://piprogramming.org/articles/How-to-make-Selenium-undetectable-and-stealth--7-Ways-to-hide-your-Bot-Automation-from-Detection-0000000017.html)

## Preprocessing earning calls 
The data we scraped look like this, there are more than 2000 earning calls from about 50 companies, earning calls of one company looks like this:
```
AAPL
├── 2011-01-18T23:05:21-05:00_Apple Management Discusses Q1 2011 Results - Earnings Call Transcript.html
├── 2011-04-20T22:00:22-04:00_Apple Management Discusses Q2 2011 Results - Earnings Call Transcript.html
├── 2011-07-19T21:40:10-04:00_Apple Management Discusses Q3 2011 Results - Earnings Call Transcript.html
├── 2011-10-18T22:10:11-04:00_Apple's CEO Discusses Q4 2011 Results - Earnings Call Transcript.html
├── 2012-01-24T20:30:07-05:00_Apple's CEO Discusses Q1 2012 Results - Earnings Call Transcript.html
├── 2012-04-24T21:10:04-04:00_Apple's CEO Discusses Q2 2012 Results - Earnings Call Transcript.html
├── ...
├── ...
├── ...
├── 2021-10-28T21:25:05-04:00_Apple Inc. (AAPL) CEO Tim Cook on Q4 2021 Results - Earnings Call Transcript.html
└── 2022-01-27T20:19:05-05:00_Apple Inc. (AAPL) CEO Tim Cook on Q1 2022 Results - Earnings Call Transcript.html
```

There are still some problems when we need directly use the data. The two basic problems are: 

1. There are some missing data for the corresponding date and firm. We need to identify them.
2. There are multiple earning calls for a corresponding firm and date. The reason that first came to our mind is that some companies have their subsidiaries and they also have their earning calls.

To handle these problems, I first read all these files from their path and record the corresponding path. To handle the difference in the separator of Windows and Mac, I use os.sep to replace “/”. I use regular expression to read the date information contained in the file name. Then I save the dataframe of all the data and their path and date information.
```
for company in companies:
    if not company.startswith("."):
        comPath = path + os.sep + company
        os.chdir(comPath)
        fileLs = os.listdir()
        for file in fileLs:
            if not file.startswith("."):
                date = re.search(r'(\d+\-\d+\-\d+)',file).group()
                filePath = comPath + os.sep + file
                filePath = filePath.replace(originalPath,"")
                df = df.append({"Company":company,"Date":date,"Path":filePath},ignore_index=True)
```

#### Missing data
Then is to handle the missing data and filter the redundant data. 
The strategy I take for missing data is that I supplement all missing data in the form of nan so that when we do further work, we can notice the data is missing. I take track of all the time quarters and all the companies shown in these earning calls and go over the for loop to supplement the missing data.
```
for date in dates:
    for company in companies:
        #         Missingg data
        if len(df[(df["Date"]==date) & (df["Company"]==company)]) == 0:
            df = df.append({"Company":company,"Date":date,"Children":False},ignore_index=True)
```

#### Redundant data
To handle the redundant data, the strategy is to set another attribute called “Children” to recognize all the earning calls belonging to the subsidiaries. (If it is a subsidiary, Children will be True.”
I use regular expression to identify whether a specific Ticker’s abbreviation appears in the name of the earning call file. It works for some of the files’ names but there are still many earning call files that do not contain the abbreviation. So, I ask my partner to give me a dictionary to relate the abbreviation with the full name of the company and search for the full name. However, it works even worse since the full name of the company is complex and may not fully match the corresponding files. So, I use regular expression to do some changes to the full name and combine the full name and abbreviation to search for the corresponding data. And it seems that it works. 
The last step is to go through the whole DataFrame manually and see whether something goes wrong.
```
names = pd.read_csv("Top 50 company in S&P500.csv")
names["Company name "] = names["Company name "].apply(lambda x:re.sub(r"(,*\s*Inc.)","",x))
names["Company name "] = names["Company name "].apply(lambda x:re.match(r"\w+",x).group())
names["Ticker"] = names["Ticker"].apply(lambda x:re.sub(r"(\w*:)","",x))
names["Ticker"] = names["Ticker"].apply(lambda x:re.sub(r"\.","",x))
names = names.set_index("Ticker")
nameDic = names["Company name "].to_dict()
```
One problem is that there are still some files that miss after we ignore the subsidiary since we do the data supplement and category together and if the data only contains the subsidiary earning call, it will miss. The solution is simple. I just first identify the subsidiary, then supplement all the data.
```
for date in dates:
    for company in companies: 
        if len(df[(df["Date"]==date) & (df["Company"]==company)]) > 1:
            temp = df[(df["Date"]==date) & (df["Company"]==company)] 
            condition1 = temp["Path"].str.contains(nameDic[company])==False
            condition2 = path.str.contains(company)==False
            index = temp[(condition1) & (condition2)&(condition3)].index
            df.loc[index,"Children"]= True
```

#### Progamely and Manually Change some text inaccuracy
The original data frame is finely organized. However, our goal is to check the relationship between the text of earning call and the stock return in the following days. Then we need to use the specific date. 
After I append the specific date using regular expression, new problems show up:

1. Some companies still do not show up even if there is an earning call for it. 
2. There are sometimes multiple earning calls for a specific period for a given company.
   
For the first problem, I check the company that we miss in the data and find out that the reason is that some company change their name such as Google to Alphabet, Facebook to Meta. For this problem, I check their names based on time independently. 

And the second problem, there are many reasons for the redundant data, and I solve them one by one:

* Some of the earning calls have wrong time information. For example, the 2014Q2 Earning call of CRM is corresponding to 2015Q2. For these stocks, I change the name of files manually.  
* Some separate the earning call from the Q&A session. For these stocks, I merge the two sessions manually.
* Some have both original version and upload version. For these stocks, I delete the upload version manually.
* Also, I mistakenly identify some subsidiary companies as the original company because of the use of abbreviation. The abbreviation of some of the companies is just one letter, such as T and we likely match something else. So, I choose to match the abbreviation only if the abbreviation is more than 3 letters.

After debugging the previous problem, the data is clean except AVGO data has still some problems. The reason is that it merges the subsidiary in 2015Q4, and uses the name of its subsidiary as its new name. So, I also check the name corresponding to the given period and manually do some changes to it.
Finally, this part is done and we can start with text processing.
```
for date in dates:
    for company in companies:
        # Missingg data
        if len(df[(df["Date"]==date) & (df["Company"]==company)]) == 0:
            df = df.append({"Company":company,"Date":date,"Children":False},ignore_index=True)
        # Child Company
        elif len(df[(df["Date"]==date) & (df["Company"]==company)]) > 1:
            temp = df[(df["Date"]==date) & (df["Company"]==company)] 
            condition1 = temp["Path"].str.contains(nameDic[company])==False
            # Campanies that have changed name
            if company == "GOOG" :
                condition1 = condition1 & (temp["Path"].str.contains("Google")==False)
            elif company == "AVGO":
                if date <= "2015Q4":
                    condition1 = temp["Path"].str.contains("Avago")==False           
            path = temp["Path"].str.replace("/"+company,"")
            condition2 = True
            if len(company)>=3:
                condition2 = path.str.contains(company)==False
            condition3 = temp["Path"].str.contains((nameDic[company].capitalize()))==False
            if company == "AVGO":
                if date <= "2015Q4":
                    condition3 = True
            index = temp[(condition1) & (condition2)&(condition3)].index
            df.loc[index,"Children"]= True
```