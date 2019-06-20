---
layout: post
title: Setting up Pyspark for live Twitter sentiment analysis
categories: [Miscellaneous]
---

# Twitter sentiment analysis
Essentially following this tutorial:
https://towardsdatascience.com/another-twitter-sentiment-analysis-bb5b01ebad90

Downloaded the Sentiment140 training data (Stanford):
`curl https://cs.stanford.edu/people/alecmgo/trainingandtestdata.zip -o data/trainingandtestdata.zip`

create a dataframe for the training data
drop unnecessary ids 
`df.drop(['id', 'date', 'query_string', 'user'], axis=1, inplace=True)`

make a dictionary to describe the data





# Using VirtualEnv with PySpark
I created a git project and a virtualenv
`git init sentiment-analysis`
`cd sentiment-analysis` and `virtualenv venv` and `. venv/bin/activate`

Then I pip installed findspark and pyspark.
I had to set environment variables for PySpark, which I did with direnv:
~~~~
# /.envrc
source venv/bin/activate

export SPARK_HOME="/home/g/Desktop/sentiment_analysis/venv/lib/python3.7/site-packages/pyspark"
export PYSPARK_PYTHON="/home/g/Desktop/sentiment_analysis/venv/bin/python"
# sets ipython as the pyspark command prompt
export PYSPARK_DRIVER_PYTHON="/usr/bin/ipython"
~~~~

I followed this excellent tutorial for just about everything:
https://towardsdatascience.com/sentiment-analysis-with-pyspark-bc8e83f80c35

All notes below are paraphrases of that tutorial, unless otherwise indicated.

Create a SparkContext:
