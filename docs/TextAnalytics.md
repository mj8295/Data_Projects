






```{python}
import os
import sys
import re
import pickle
import random

import numpy as np
import pandas as pd
from itertools import compress

import nltk
from nltk.tokenize import sent_tokenize
from nltk.corpus import stopwords
nltk.download('punkt')
nltk.download('stopwords')

from wordcloud import WordCloud
import matplotlib.pyplot as plt

from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
from scipy.spatial.distance import cosine
from wordcloud import WordCloud
from sklearn.decomposition import LatentDirichletAllocation as LDA
```

```{python}
def corpus(path, encoding = 'utf8', nonalphabetic = False, tolower = False):
    if os.path.isdir(path):
        files = [x for x in os.listdir(path)]
        doc8raw = []
        filenames = []
        for file in files:
            filenames.append(file)
            f = open(f'{path}/{file}', 'r', encoding=encoding)
            s = f.readlines()
            f.close()
            s = [x.strip() for x in s]
            doc8raw.append(''.join(s))
    elif os.path.isfile(path):
        filenames = path
        f = open(path, 'r', encoding = encoding)
        f.colse()
        s = [x.strip() for x in s]
        doc8raw = s
    if tolower:
        doc8raw = [x.lower() for x in doc8raw]
    if nonalphabetic:
        doc8raw = [re.sub("[^a-zA-Z]", " ", x) for x in doc8raw]
    return doc8raw, filenames
```

```{python}
def cooccurrence_next_word( sentlist, targets ): # cooccurrence defined as right after the focal word
    cooc = np.zeros((len(targets), len(targets)), np.float64)
    word2id = {w:i for (i,w) in enumerate(targets,0)}
    for sent in sentlist:
        words = sent.split()
        n = len(words)
        for i in range(n-1):
                if(words[i] in word2id.keys() and words[i+1] in word2id.keys()):
                    cooc[word2id[words[i]], word2id[words[i+1]]] +=1
#    np.fill_diagonal(cooc, 0)
    return cooc

```

```{python}
def cooccurrence_same_sentence( sentlist, targets ): # cooccurrence defined as being in the same sentence
    cooc = np.zeros((len(targets), len(targets)), np.float64)
    word2id = {w:i for (i,w) in enumerate(targets,0)}
    for sent in sentlist:
        words = sent.split()
        n = len(words)
        for i in range(n):
            for j in range(i+1,n):
                if(words[i] in word2id.keys() and words[j] in word2id.keys()):
                    cooc[word2id[words[i]], word2id[words[j]]] +=1
    np.fill_diagonal(cooc, 0)
    return cooc+cooc.T # necessary since symmetry is exploited to save computation during construction
```

```{python}
def cooccurrence_symmetric_window( sentlist, targets, weights ): # cooccurrence based on weighted moving window
    m = len(weights)
    cooc = np.zeros((len(targets), len(targets)), np.float64)
    word2id = {w:i for (i,w) in enumerate(targets,0)}
    for sent in sentlist:
        words = sent.split()
        n = len(words)
        for i in range(n):
            end = min(n-1, i+m)
            for j in range(i+1, end+1):
                if(words[i] in word2id.keys() and words[j] in word2id.keys()):
                    cooc[word2id[words[i]], word2id[words[j]]] += weights[j-i-1]
    np.fill_diagonal(cooc, 0)
    return cooc+cooc.T # necessary since symmetry is exploited to save computation during construction

```

```{python}
def ppmi_from_cooc( cooc ):
    marginal0 = cooc.sum(0) # column sum
    nzc = marginal0!=0      # filtered columns
    marginal1 = cooc.sum(1) # row sum
    nzr = marginal1!=0      # filtered rows
    lift = cooc[nzr][:,nzc]*sum(marginal0) / ( np.array([marginal1[nzr]]).T.dot( np.array([marginal0[nzc]]) ) )
    ppmi = np.log2( lift*(lift>1) + (lift<=1) ) # np.where(lift>1, np.log2(lift), 0) gives RuntimeWarning
    return ppmi,nzr, nzc
```

```{python}
def plotrend1(vectors, t, color):
    ts = pd.Series()
    for index in vectors:
        ts[index] = vectors[index][t]
    ts.plot(label = t)
    plt.legend()
    
```

```{python}
def plotrend2(vocab, matrices, t1, t2, order, color):
    word2id = {w:i for (i,w) in enumerate(vocab, 0)}
    ts = pd.Series()
    if order==1:
        for index in matrices:
            ts[index] = matrices[index][word2id[t1], word2id[t2]]
    elif order==2:
        for index in matrices:
            a = matrices[index][word2id[t1], :]
            b = matrices[index][word2id[t2], :]
            ts[index] = a.dot(b) / (np.linalg.norm(a) * np.linalg.norm(b))
    ts.plot(label = f'{t1}, {t2}')
    plt.legend()
```

```{python}
def txtseries(targets, year, month, window = -1):
    coocs = {}
    freqs = {}
    sentcnts = {}
    wordcnts = {}
    nz = np.ones(len(targets), dtype = bool)
    for y in year:
        for m in month:
            index = pd.datetime(y, m, 1)
            filename = f'fpost-{y}-{m}.csv'
            f = open(filename, 'r')
            symbols = f.read().lower()
            sentencelist = sent_tokenize(symbols)
            print(f'Parsing {filename}')
            coocs[index] = cooccurrence_symmetric_window(sentencelist, targets, 1/np.arange(1,4))
            nz = np.logical_and(nz, (coocs[index].sum(0)>0))
            
            wordlist = ' '.join(sentencelist).split()
            sentcnts[index] = len(sentencelist)
            wordcnts[index] = len(wordlist)
            
            freq = {i:0 for i in targets}
            for w in wordlist:
                if w in targets:
                    freq[w] +=1
            freqs[index] = freq
    return sentcnts, wordcnts, freqs, coocs, nz
```

```{python}
os.chdir('C:/Users/Max/Documents/MSBA/MSBA Spring B/Text Analytics/Assignment 1/spritzer15/spritzer/15')
doc8raw, filenames = corpus('C:/Users/Max/Documents/MSBA/MSBA Spring B/Text Analytics/Assignment 1/spritzer15/spritzer/15', encoding = 'latin1')
```

```{python}
random.seed(10)
stop = {'http', 'https', 'co', 'amp', 'via', 'don', 'dont'}|set(stopwords.words("english"))|set(stopwords.words("spanish"))|set(stopwords.words("portuguese"))
vectorizer = CountVectorizer(stop_words = stop)
tf = vectorizer.fit_transform(doc8raw).toarray()
nonsparse = (tf>0).sum(0) > .001*tf.shape[0]
A = tf[:,nonsparse]
```
```{python}
terms = np.array(vectorizer.get_feature_names())
terms = terms[nonsparse]

T = 10

lda = LDA(n_components = T)
lda.fit(A)
```
```{python}
docTopics = lda.transform(A)
topicTerms = lda.components_
```
```{python}
k = 1
for k in range(0, T):
    topic = dict(zip(terms, topicTerms[k,:]))
    topic = {k: v for k, v in sorted(topic.items(), key = lambda item: item[1], reverse = True)}
    topicwords = list(topic.keys())
    print("/ntopic", k, ":/t", topicwords[0:10])
```
```{python}
k = 1
topic = dict(zip(terms, topicTerms[k,:]))
topic = {k: v for k, v in sorted(topic.items(), key = lambda item: item[1], reverse = True)}
wcloud = WordCloud().generate_from_frequencies(topic)
plt.imshow(wcloud, interpolation = 'bilinear')

```
```{python}

```
```{python}

```
```{python}

```
```{python}

```
```{python}

```
```{python}

```
