---
layout: post
tags: 
  - hadoop
title: TF-IDF with Hadoop Streaming
image: /assets/img/pipeline.jpg
---

<img 
    src="/assets/img/pipeline.jpg" 
    alt="pipeline" 
   class="mb-0">
<sub><sup>
Photo by <a href="https://unsplash.com/@mbenna" rel="nofollow">Mike Benna</a> on Unsplash 
</sup></sub>

Today I want to talk about how we can calculate tf-idf with hadoop streaming.

First of all, for those who don't know what TF-IDF is, I can explain. It is statistical metrics of words, which reflects
the importance of each word to a document. The bigger TF-IDF value of a particular word, and a particular document the
more frequently this word appears in a document and the rarely in other documents. You can gather more information from
the <a href="https://en.wikipedia.org/wiki/Tf–idf" rel="nofollow">Wikipedia article</a>. It is widely used in data science tasks, especially
related to text processing problems. In this article, I will use a term and a word interchangeably.

Before starting, I give some information about Hadoop streaming. It's a member of big Hadoop family tools. The component
lets process data in a map-reduce way. As you know, the native language for Hadoop is java, but it isn't necessary to
know this language to work with it. In this article, we will use python. Unfortunately, debugging python scripts can't
be a pain in your neck. I highly recommend testing your scripts locally before running them on a Hadoop cluster. More
information about Hadoop streaming you can find
on <a href="https://hadoop.apache.org/docs/r1.2.1/streaming.html" rel="nofollow">the official website</a>

Let's get to the point, in order to calculate TF-IDF metric, we need to know how we can do it. Look at the formula below

$$ TFIDF(term,document) = TF(term, document)*IDF(term) $$

**TF** is called **Term Frequency**. So you can look at it as if it is a function
**TF(term, document)** which has several parameters: _term_ and _document_.

$$ TF = \frac{\text{term count in a document}}{\text{total count of words in the same document}} $$

**IDF** is called **Inverse document frequency**. It depends on а term and shows how many documents contain a particular
term. In other words, It is a function:

$$ IDF(term) = \frac{1}{\log_e(1+ \text{count of documents which contain a particular term})} $$

That's all that I wanted to say about **TF-IDF**. The next step is the implementation of this formula.

We will compute the metric for each word and an article in 3 steps.

## The first stage.

Originally, input data will be a dump of Wikipedia articles. It has a pretty straightforward format:

```text
12	Anarchism is often defined as a political philosophy which holds the state to be undesirable, unnecessary, or harmful.  ...
```

i.e.

```text
<article_id>\tab<text>
```

The input data should be split into separated words. Also, we have a list of stop words we should take it into account
during parsing text. After the mapping phase, output data will have the next format:

```text
<article_id>\tab<word>\tab<number>
```

However, map output data contains 3 different types of information.

The first of them contains a count of words in the article, i.e. we will write for each word a row like that:

```text
<article_id>\tab<word>\tab1
```

The second type contains information about the total count of words. For each word, we will write a row like that:

```text
<article_id>\tab\ \tab1
```

As you can see, a word is missed.

Finally, the third type contains information about the count of articles contains a particular word. During parsing
text, we will write a row for each word which appear for the first time in a text. Its format bellow:

```text
 \tab<word>\tab1
```

Аn article id would be missed here.

There is a part of **mapper-1.py** which do all this stuff:

```python
import sys
import re

reload(sys)
sys.setdefaultencoding('utf-8')  # required to convert to unicode

path = 'stop_words_en.txt'
stop_words = set()

# Your code for reading stop words here
for l in open(path):
    line = l.strip()
    stop_words.add(line.strip())

for line in sys.stdin:
    try:
        article_id, text = unicode(line.strip()).split('\t', 1)
    except ValueError as e:
        continue

    text = re.sub("^\W+|\W+$", "", text, flags=re.UNICODE)
    words = re.split("\W*\s+\W*", text, flags=re.UNICODE)

    total_count = 0
    unique_words = set()
    for w in words:
        word = w.lower().strip()
        if word not in stop_words:
            print("%s\t%s\t%s" % (article_id, word, 1))
            print("%s\t%s\t%s" % (article_id, ' ', 1))

            if word not in unique_words:
                print("%s\t%s\t%s" % (" ", word, 1))
                unique_words.add(word)
```

In a reduce phase, we will group all values by keys: article id and word and sum all of them.

```python
import sys
import re

reload(sys)
sys.setdefaultencoding('utf-8')  # required to convert to unicode

current_article_id = None
current_word = None
current_count = 0

for line in sys.stdin:
    try:
        article_id, word, count = unicode(line).split('\t', 2)
    except ValueError as e:
        raise e

    if article_id == current_article_id and word == current_word:
        current_count = int(count) + current_count
    else:
        if current_article_id:
            print("%s\t%s\t%s" % (current_article_id, current_word, current_count))

        current_article_id = article_id
        current_word = word
        current_count = int(count)

if current_count > 0:
    print("%s\t%s\t%s" % (current_article_id, current_word, current_count))
```

In order to run hadoop job, we should execute a shell command:

```shell
OUT_DIR="intermediate-result-1"
hdfs dfs -rm -r ${OUT_DIR} > /dev/null

yarn jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-streaming.jar \
    -D mapred.output.key.comparator.class=org.apache.hadoop.mapred.lib.KeyFieldBasedComparator \
    -D mapred.jab.name="tf-idf-job" \
    -D mapreduce.job.reduces=5 \
    -D stream.num.map.output.key.fields=2 \
    -D mapred.text.key.partitioner.options="-k1,1" \
    -D mapred.text.key.comparator.options="-k1,1 -k2,2" \
    -files mapper-1.py,reducer-1.py,/datasets/stop_words_en.txt \
    -mapper "python2 mapper-1.py" \
    -reducer "python2 reducer-1.py" \
    -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
    -input /data/wiki/en_articles_part \
    -output ${OUT_DIR} > /dev/null
```

The most interesting part here is sorting parameters. They are required to sort rows by 2 keys. In our case, 2 keys are
article id and words. A reducer consumes sorted data, and it lets make reducer code simpler. Another interesting thing
is to pass a stop_words_en.txt file that contains stop words. This file would be present in the current working
directory of the task and would be available to read.

As a result, output data of the first stage would look like that:

```text
993	varying	3
993	version	1
993	video	4
993	voltage	5
993	vs	2
993	waveform	1
```

## The second stage.

In this stage, we will calculate the **TF** value for each word. The output of the previous stage contains all required
information. All work will be done in a reducer. A mapper does nothing. Its output data will be the same as input data.

Below there is a code of **mapper-2.py**:

```python
import sys

reload(sys)
sys.setdefaultencoding('utf-8')  # required to convert to unicode

for line in sys.stdin:
    print(unicode(line))
```

In this case, you can try to use in-build mapper `org.apache.hadoop.mapred.lib.IdentityMapper`
and `org.apache.hadoop.mapred.lib.IdentityReducer` reducer. Unfortunately, I have caught some exceptions related to
wrong typecasting, so I had to write own mapper with the same logic. 😕

As I said earlier, a reducer will calculate **TF** values and skip rows with blank article id values. We will use
skipped rows to calculate **IDF** values the next time.

```python
import sys
import re

reload(sys)
sys.setdefaultencoding('utf-8')  # required to convert to unicode

current_article_id = None
count_terms_in_article = 0
current_word = None

for line in sys.stdin:
    try:
        article_id, word, count = unicode(line).split('\t', 2)
    except ValueError as e:
        print(line)
        continue

    if article_id == " ":
        print("%s\t%s\t%s" % (word, article_id, int(count)))
    elif current_article_id == article_id:
        tf = float(count) / count_terms_in_article
        print("%s\t%s\t%s" % (word, article_id, tf))
    else:
        current_article_id = article_id
        count_terms_in_article = float(count)
        current_word = word
```

The shell command to run a Hadoop job is the same as the shell command of the previous stage. We sort mapper output data
by 2 keys as well.

```shell
IN_DIR="intermediate-result-1"
OUT_DIR="intermediate-result-2"
hdfs dfs -rm -r ${OUT_DIR} > /dev/null

yarn jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-streaming.jar \
    -D mapred.output.key.comparator.class=org.apache.hadoop.mapred.lib.KeyFieldBasedComparator \
    -D mapred.jab.name="tf-idf-job-2" \
    -D mapreduce.job.reduces=5 \
    -D stream.num.map.output.key.fields=2 \
    -D mapred.text.key.partitioner.options="-k1,1" \
    -D mapred.text.key.comparator.options="-k1,1 -k2,2" \
    -files reducer-2.py,identity-mapper.py \
    -mapper "python2 identity-mapper.py" \
    -reducer "python2 reducer-2.py" \
    -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
    -input ${IN_DIR} \
    -output ${OUT_DIR} > /dev/null
```

As the result, we would have the output data like that:

```text
valued	988	0.00263157894737
various	988	0.00526315789474
view	988	0.00263157894737
```

## The third stage.

There is nothing to do for a mapper, so we would use identity-mapper like in the previous stage. We need to sort mapper
output data by 2 keys - word and article id. Input data contains rows with blank article id. We need them to compute **
IDF** then we will multiply
**IDF** values with all values of other rows grouped by word. By this way, we will have **TF-IDF** metric values
calculated for each word and document.

Reducer code is below.

```python
import sys
import re
from math import log

reload(sys)
sys.setdefaultencoding('utf-8')  # required to convert to unicode

count_of_articles_with_term = 0
current_word = None

for line in sys.stdin:
    try:
        word, article_id, num = unicode(line).split('\t', 2)
    except ValueError as e:
        continue

    if current_word == word:
        idf = 1.0 / log(1 + count_of_articles_with_term)
        tf = float(num)
        result = str(tf * idf)
        print("%s\t%s\t%s" % (word, article_id, result))
    else:
        count_of_articles_with_term = int(num)
        current_word = word
```

A shell command to run a hadoop job is the same.

```shell
IN_DIR="intermediate-result-2"
OUT_DIR="result"
hdfs dfs -rm -r ${OUT_DIR} > /dev/null

yarn jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-streaming.jar \
    -D mapred.output.key.comparator.class=org.apache.hadoop.mapred.lib.KeyFieldBasedComparator \
    -D mapred.jab.name="tf-idf-job-3" \
    -D mapreduce.job.reduces=5 \
    -D stream.num.map.output.key.fields=2 \
    -D mapred.text.key.partitioner.options="-k1,1" \
    -D mapred.text.key.comparator.options="-k1,1 -k2,2" \
    -files reducer-3.py,identity-mapper.py \
    -mapper "python2 identity-mapper.py" \
    -reducer "python2 reducer-3.py" \
    -partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
    -input ${IN_DIR} \
    -output ${OUT_DIR} > /dev/null
```

Hooray 🎉 Everything is done, and we can have a look at the result:

```shell
hdfs dfs -cat result/* | grep -P 'labor\t12' | head -1 | awk '{print $3}'
```

For labor and article id 12 **TF-IDF** equals **0.00035046896211**

There is a <a href="https://hub.docker.com/r/bigdatateam/yarn-notebook/" rel="nofollow">docker playground</a> 
to reproduce all step on your own. Moreover, you can get a Jupyterhub notebook that encompasses all stages on the Github repo.
