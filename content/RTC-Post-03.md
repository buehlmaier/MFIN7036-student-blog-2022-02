---
Title: Part 3: Quantifying Performance: From 10K to Keyword Score (Group Raw Text Connoisseurs)
Date: 2022-03-27 22:36
Category: Progress Report
---

By Group "Raw Text Connoisseurs"

Although it was anticipated that the earlier HTML Extractor function will be sufficient to extract information from the 10-k reports, multiple problems had emerged from the way the 10-K reports were uploaded to EDGAR and the lack of consistent html formatting over the years. 

# Problem 1: The varying HTML format in 10-K report
Throughout the 2007-2021 time frame of 10-K reports we had extracted, the main portion of the text had been located in three different elements. One can find it in <p\>. <font\> and even in <span\>. This changing structure in 10-K report had broke the extractHTML function and required modification to successfully turn 10-K reports into String for processing.

### **Solutions:**
The first version of the solution is to check if the extractHTML function yield text at the end of reading the element. Ranked by the frequency and the times these element were used, the snipet is as follow:

```python
def extractHTML(file):
    text = ''
    soup = BeautifulSoup(file, 'html.parser')
    paragraph = soup.findAll('p')
    
    for p in paragraph:
        if not p.findChildren('a') and not p.has_attr('class'):
            text += p.get_text(strip = True)
    
    if text == '':
        paragraph = soup.findAll('font')
        for p in paragraph:
            text += p.get_text(strip = True)
        
    if text == '':
        paragraph = soup.findAll('span')
        for p in paragraph:
            text += p.get_text(strip = True)

    return text
```

**An even better solution:** <br>
Although this approach solve the problem, as we scale the project, the keyword scoring will be repeatedly used and optimisation in speed at this part will be very helpful in the future. For this, we arrive to a soluton where we only check for the text once and check for extraction in the <font\> element instead. This allow the program to only run the get_text function twice in the worst case scenario with the same output. 


```python
def extractHTML(file):
    text = ''
    soup = BeautifulSoup(file, 'html.parser')
    paragraph = soup.findAll('p')
    
    for p in paragraph:
        if not p.findChildren('a') and not p.has_attr('class'):
            text += p.get_text(strip = True)
    
    if text == '':
        paragraph = soup.findAll('font')
        if paragraph == []:
            paragraph = soup.findAll('span')
        
        for p in paragraph:
            text += p.get_text(strip = True)     
    return text
```

# Problem 2: The varying 10-K length introducing noise to keyword score
If the keyword score are used directly from the amount tallied, a problem that this introduce will be that companies that write more generally in 10-K would be rewarded with more favourable scores on average. 

### **Solutions:**
To solve this problem, the project is adjusted to retain the bag of words from the 10-K reports directly and tally up the words to be passed into the data frame before being tallied with the keyword lists. This is a simple addition but provided a lot of value for further analysis. 

```python
def scoreFactor(text, keywordLists):
    scoreArr = []
    
    lemmatized = bagOfWords(text)
    count = Counter(lemmatized)
    
    # This line of code
    scoreArr.append(sum(count.values()))
    # Addition to count the total words in 10-K report
    
    for keywordList in keywordLists:
        result = countKey(count, keywordList)
        scoreArr.append(sum(result.values()))
    
    return scoreArr
```


# Problem 3: 10-K Reports with both <p\> and <div\> element used
When the program is ran, there were a few entries with supiciously low scores. Upon more investigation with the 10-K files, it is found that the 10-K problems did not stop from the previous solution. Another issue for 10-K is that some files store parts of the content in <p\> but moved the main contents in the <div\> element.

### **Solutions:**
The approach from the previous problem will not work in this part as <p\> actually have some value. For this, we arrive in the solution to catch keyword results that have length lower than the threshold of 4000 words. The 4000 word threshold is extracted from the current lowest 10-K document length. After implementing this olution, the problem was resolved.

``` python
# catching the error in the flow
if scores[0] <= 4647:
    try:
        f = codecs.open(masterPath + company + '/' + str(year) + '.html','r', encoding = 'unicode_escape')
    except: 
        f = codecs.open(masterPath + company + '/' + str(year) + '.htm','r', encoding = 'unicode_escape')
    patched = handleError(f.read())
    print('   caught')
    scores = scoreFactor(patched, keywordList)

```

```python
# The Function to handle error
def handleError(file):
    text = ''
    soup = BeautifulSoup(file, 'html.parser')
    paragraph = soup.findAll('div')
    
    for p in paragraph:
        if not p.findChildren('a') and not p.has_attr('class'):
            text += p.get_text(strip = True)
    return text
```

