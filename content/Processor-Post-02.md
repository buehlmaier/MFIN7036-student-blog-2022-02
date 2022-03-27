---
Title: The Appetizers - Text Preprocessing (Group Processors)
Date: 2022-03-26 10:16
Category: Progress Report
---

By Group "Processors"

# Introduction

Before showing you the main dishes of our project, let me first introduce you to this can't miss appetizer - text preprocessing. To be fair, preprocessing is extremely important in text analysis since text data always comes with numerous formats and structures. We will need to remove unnecessary contents and uniform the form of the text data to further process the analysis more efficiently and effectively. 

The required steps of preprocessing really depend on the text data. We just have to be going back and forth until we get the satisfied "cleaned" data so it is basically like tailor-made. For our text data, the following preprocessing steps are involved:

1. Convert to lower case
2. Remove line break
3. Deal with named entity
4. Remove emoji
5. Remove Twitter ID
6. Remove web address
7. Remove stop word
8. Remove punctuation
9. Remove number
10. Lemmatize
11. Remove customized list of words

I would like to note that the steps are performed in this specific order because we want to minimize the interruptions between the steps. For example, I may have a problem with identifying Twitter ID and web address if our first step is the removal of punctuation.

# Text Preprocessing

### Convert to Lower Case

The first step is lower casing. We want all our text data to be in the same case format so that the same word will not be differentiated when one is in lower case while the other is in upper case.

```python
df['text_clean'] = df['Content'].str.lower()
```

Before:
> This Is An Example

After:
> this is an example

### Remove Line Break

When we scrap relatively long text data from a website, like news articles, very often there will be line breaks in the text. 

```python
df['text_clean'] = df['text_clean'].str.replace('\n', ' ')
```

Before:

> this is a\nline break

After:

> this is a line break

### Deal with Named Entity

Some word combinations may not be meaningful when you look at them separately, especially for named entities and  people. Therefore, we have to group them together to make sense for later analysis. However, there are just way too many word combinations and it is impossible to obtain a complete list of them. Our approach is to use the built-in methods for Named Entity Recognition in `spacy` to extract all named entities in our text data, then remove the space between the words to turn them into one "word". 

```python
# get the list of named entities 
NER = spacy.load('en_core_web_sm')
ent_list = []
for i in range(0,len(df)):
    article = NER(df['text_clean'][i])
    for word in article.ents:
        if word.label_ in ['NORP', 'GPE', 'ORG', 'PERSON']:
            # NORP = Nationalities or religious or political groups
            # GPE = Countries, cities, states
            # ORG = organization, PERSON = person
            ent_list.append(word.text)
ent_list = list(set(ent_list))

# keep only the items with more than one word
ent_list = [word for word in ent_list if ' ' in word]

# remove space in named entity
def group_named_entity(article):
    list1 = ent_list
    list2 = [item.replace(' ','') for item in list1]
    for i in range(len(list1)):
          if list1[i] in article:
                article = article.replace(list1[i],list2[i])
    return article
df['text_clean'] = df['text_clean'].apply(lambda x: group_named_entity(x))
```

Before:

>Bank of China

After:

>BankofChina

### Remove Emoji

Emojis are frequently used in social media as a way to express emotion, but it may not be too helpful for text analysis.

This method to remove emoji is found in [this website](https://poopcode.com/how-to-remove-emoji-from-text-in-python/).

```python
def remove_emoji(string): # citation: https://poopcode.com/how-to-remove-emoji-from-text-in-python/
    emoji_pattern = re.compile("["
                               u"\U0001F600-\U0001F64F"  # emoticons
                               u"\U0001F300-\U0001F5FF"  # symbols & pictographs
                               u"\U0001F680-\U0001F6FF"  # transport & map symbols
                               u"\U0001F1E0-\U0001F1FF"  # flags (iOS)
                               u"\U00002500-\U00002BEF"  # chinese char
                               u"\U00002702-\U000027B0"
                               u"\U00002702-\U000027B0"
                               u"\U000024C2-\U0001F251"
                               u"\U0001f926-\U0001f937"
                               u"\U00010000-\U0010ffff"
                               u"\u2640-\u2642"
                               u"\u2600-\u2B55"
                               u"\u200d"
                               u"\u23cf"
                               u"\u23e9"
                               u"\u231a"
                               u"\ufe0f"  # dingbats
                               u"\u3030"
                               "]+", flags=re.UNICODE)
    return emoji_pattern.sub(r'', string)
df['text_clean'] = df['text_clean'].apply(lambda x: remove_emoji(x))
```
### Remove Twitter ID

Twitter IDs are also not very useful in text analysis so we would remove them in our case. The regular expression is used in this part through the `re` module. The main idea is to identity the '@' sign before each user name.

```python
df['text_clean'] = df['text_clean'].apply(lambda x: re.sub(r'@\S+', ' ', x))
```

### Remove Web Address

Similarly, web addresses do not have much meaning either so there is no point of keeping them. In consideration of the need in our text data, it only has the simplest kind of address which is the one with '.com', so our code only covers this type of web address. 

```python
df['text_clean'] = df['text_clean'].apply(lambda x: re.sub(r'\S+\.com\S*', ' ', x))
```
### Remove Stop Word

Stop words are  words that frequently show occurred in text but do not provide much useful information. We normally would not want these words to take up space in the dataset and they might actually affect the later analysis process of counting term frequency in our case. Thus, we decide to remove them for our own convenience. 

We try to combine the stop word list from a few libraries to get a more comprehensive list of stop words. We first use the built-in method `remove_stopwords` from  `gensim.parsing.preprocessing` to have the first round of stop words removal. Then we define `remove_stopwords_again` which we remove stop words from `stopwords` in `nltk.corpus	`, `ENGLISH_STOP_WORDS` from `sklearn.feature_extraction.stop_words`, and `STOP_WORDS` from `spacy.lang.en.stop_words`.

```python
df['text_clean'] = df['text_clean'].apply(lambda x: remove_stopwords(x)) # remove stropwords in gensim package
stopword_list = set(stopwords.words('english') + list(ENGLISH_STOP_WORDS) +list(STOP_WORDS)) # stopwords in nltk, spacy, sklearn
def remove_stopwords_again(string):
    text = ' '.join([word for word in string.split() if word not in stopword_list])
    return text
df['text_clean'] = df['text_clean'].apply(lambda x: remove_stopwords_again(x))
```
Before:

>this is an example so hi how are you doing today

After:

>example hi today

### Remove Punctuation

The existence of punctuation may as well affect us counting term frequency in later process. It may attach to the word right next to it. As a result, that word will be treated as a different word for the same exact word without the punctuation attached. 

We obtain a list of punctuations from the `string` library and replace each of them with a space. However, we notice some characters are included in the list when we screen the result after removal, so we have the code to manually remove those as well.

```python
df['text_clean'] = df['text_clean'].apply(lambda x: x.translate(str.maketrans(string.punctuation, ' '*len(string.punctuation))))
df['text_clean'] = df['text_clean'].str.replace('’', ' ') 
df['text_clean'] = df['text_clean'].str.replace('‘', ' ') 
df['text_clean'] = df['text_clean'].str.replace('”', ' ')
df['text_clean'] = df['text_clean'].str.replace('“', ' ') 
df['text_clean'] = df['text_clean'].str.replace('—', ' ')
df['text_clean'] = df['text_clean'].str.replace('–', ' ')
```

Before:

>@may: this is an example!

After:

>may this is an example

## Remove Number

Numbers do not carry much meaning in text analysis so we may just remove them for simplification. 

```python
df['text_clean'] = df['text_clean'].apply(lambda x: re.sub(r'\d+', ' ', x))
```

### Lemmatize

Both stemming and lemmatization can help us to restore the original form of a word. The main difference between this two methods is that stemming will reduce to only the root form of the word which sometimes may lead to incorrect spelling and meaning, while for lemmatization, it will convert the word to the simplest but meaningful form. Of course, the process of lemmatization will also be more time consuming, but we do want our result to be interpretable words so we will go with lemmatization here.

We use `WordNetLemmatizer` from `nltk.stem` in our process. 

```python
lemmatizer = WordNetLemmatizer() 
wordnet_map = {"N":wordnet.NOUN, "V":wordnet.VERB, "J":wordnet.ADJ, "R":wordnet.ADV}
def lemmatize_words(text):
    pos_tagged_text = nltk.pos_tag(text.split())
    return " ".join([lemmatizer.lemmatize(word, wordnet_map.get(pos[0], wordnet.NOUN)) for word, pos in pos_tagged_text])
df['text_clean'] = df['text_clean'].apply(lambda x: lemmatize_words(x))
```

Before:

>this is an example. how are you doing?

After:

>this be an example. how be you doing?

Side note, I will normally perform an additional round of stop words removal after lemmatization to catch the left out stop words that are not in their standard form.

### Remove Customized List of Words

We would also customize a list of words `common_list` that we frequently see in the text but believe to be insignificant to our analysis. This list will be updated along the way of analysis process.

```python
common_list = ['usd', 'cad', 'day', 'week', 'month', 'past', 'today', 'time', 'period', 'near', 'overall']
def remove_common_words(string):
    text = ' '.join([word for word in string.split() if word not in common_list])
    return text
df['text_clean'] = df['text_clean'].apply(lambda x: remove_common_words(x))
```

# Conclusion

The above steps are what we have for text preprocessing, but as I said before, it will be possible for us to add new steps later on to get a more satisfied result if needed. I would like to end this blog with an example of one news article that we preprocessed. 

Before:

>Vitalik Buterin, the 28-year-old billionaire co-creator of Ethereum, isn’t a fan of the famous Bored Ape Yacht Club NFT collection.\n“The peril is you have these $3 million monkeys and it becomes a different kind of gambling,” Buterin, 28, told TIME Magazine. The publication noted that he was “referring to the Bored Ape Yacht Club.”\nBAYC is one of the top NFT collections living on Ethereum, with $1.41 billion in total volume traded, according to data site DappRadar. Many of the top all-time BAYC sales were for over $1 million, with celebrity owners like Jimmy Fallon, Serena Williams and Paris Hilton.\nBut Buterin has more wholesome hopes for future Ethereum use cases, including becoming “the launchpad for all sorts of sociopolitical experimentation: fairer voting systems, urban planning, universal basic income, public-works projects,” TIME reported.\nInstead, “There definitely are lots of people that are just buying yachts and Lambos,” he said.\nHe also mentioned the millions of dollars in cryptocurrency donated to aid Ukraine as Russian President Vladimir Putin began an invasion, and seemingly compared it to money spent on Bored Apes.\n“One silver lining of the situation in the last three weeks is that it has reminded a lot of people in the crypto space that ultimately the goal of crypto is not to play games with million-dollar pictures of monkeys, it’s to do things that accomplish meaningful effects in the real world,” Buterin told TIME on March 14.\nIn 2022, Buterin hopes to be “more risk-taking and less neutral,” he told TIME. “I would rather Ethereum offend some people than turn into something that stands for nothing.”\nThis story was originally featured on Fortune.com

After:

>vitalikbuterin year old billionaire creator ethereum fan famous bore ape yacht club nft collection peril million monkey different kind gamble buterin tell magazine publication note refer bored ape yacht club bayc nft collection live ethereum billion total volume trade accord data site dappradar bayc sale million celebrity owner like jimmyfallon serenawilliams parishilton buterin wholesome hope future ethereum use case include launchpad sort sociopolitical experimentation fairer vote urban plan universal basic income public work project report instead definitely lots people buy yacht lambos mention million dollar cryptocurrency donate aid ukraine russian president vladimirputin begin invasion seemingly compare money spend bore ape silver line situation remind lot people crypto space ultimately goal crypto play game million dollar picture monkey thing accomplish meaningful effect real world buterin tell march buterin hop risk neutral told ethereum offend people turn stand story originally feature

Thanks for reading and please look forward to our later blogs on text analysis. Meanwhile, have a good day and stay safe.
