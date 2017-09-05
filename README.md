# eval_IR_engines
This repo is to evaluate BM25F on ES and Terrier.

First, we fixed a single word as the query: "earth"

Second: we created a test collection which consist of 5 documents.
Each document has two fields (title and body).

Each document was fabricated to be the top document for the query using one out of total 5 weighting schemes.

The 5 weighting schemes are as follows:
* Title only (Wt = 10, Wb = 0)
* Title is boosted (Wt = 7, Wb = 3)
* Title = Body (Wt = 5, Wb = 5)
* Body is boosted (Wt = 3, Wb = 7)
* Body only (Wt = 0, Wb = 10)

Normalisation parameter B values (for both title and body) are set to default 0.75

The fabricated documents are as follows

id|Title|	Body
---|----|----
1|earth earth  earth earth earth earth earth earth earth|	mars mars  mars  mars mars mars  mars  mars mars mars  mars  mars mars mars  mars  mars mars mars  mars  mars mars mars  mars  mars
2|earth earth earth earth earth earth earth earth earth earth earth mars	|earth earth earth earth earth earth earth earth earth mars mars mars mars mars mars mars mars mars mars mars mars mars
3|earth earth earth earth earth earth earth earth earth earth mars mars mars mars	|earth earth earth earth earth earth earth earth earth earth earth mars mars mars mars mars mars mars mars mars
4|earth earth earth mars mars	|earth earth earth earth earth earth earth earth earth earth earth earth mars mars mars mars mars mars
5|mars mars mars mars mars mars mars mars mars mars	|earth earth earth earth earth earth earth earth earth earth earth earth mars mars mars mars



Field statistics the above documents are as follows

id |titleLen|AvgTitleLen|BodyLen|AvgBodyLen|x_title|x_body|nx_dft_title|nx_dft_body|
---|--------|---------- |-------|--------  |------ |----- |----------- |---------  |
1|9	|10	|24	|20	|9	|0	|9.72972973	    |0	            |
2|12	|10	|22	|20	|11	|9	|9.565217391	|8.372093023	|
3|14	|10	|20	|20	|10	|11	|7.692307692	|11	            |
4|5	|10	|18	|20	|3	|12	|4.8	        |12.97297297	|
5|10	|10	|16	|20	|0	|12	|0	            |14.11764706	|


expected BM25F scores for each weighting schemes:

id|Title Only Score|Boost Title Score|Title = Body Score|Body Boost Score|Body Only Score
---|----------------|-----------------|------------------|----------------|---------------
1|0.9878	|0.9827	|0.9759	|0.9605	|0.0000
2|0.9876	|0.9871	|0.9868	|0.9864	|0.9859
3|0.9846	|0.9864	|0.9873	|0.9882	|0.9892
4|0.9756	|0.9837	|0.9867	|0.9887	|0.9908
5|0.0000	|0.9724	|0.9833	|0.9880	|0.9916



# Index and Retrieval in Elastisearch

## Create Index

```Kibana
PUT eval_es
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "similarity": {
        "sim_title": {
            "type": "BM25",
            "b": 0.75,
            "k1": 1.2
        },
        "sim_body": {
            "type": "BM25",
            "b": 0.75,
            "k1": 1.2
        }
      }
  },
  "mappings":
  {
    "eval":
    {
      "properties":
      {
        "title":
        {
          "type": "text",
          "term_vector": "yes",
          "similarity": "sim_title"
        },
        "summary":
        {
          "type": "text",
          "term_vector": "yes",
          "similarity": "sim_title"
        }
      }
    }
  }
}
```


## Index the 5 documents
```Kibana
PUT /eval_es/eval/1
{
  "title":"earth earth  earth earth earth earth earth earth earth",
  "body": "mars mars  mars  mars mars mars  mars  mars mars mars  mars  mars mars mars  mars  mars mars mars  mars  mars mars mars  mars  mars"
}

PUT /eval_es/eval/2
{
  "title":"earth earth earth earth earth earth earth earth earth earth earth mars",
  "body": "earth earth earth earth earth earth earth earth earth mars mars mars mars mars mars mars mars mars mars mars mars mars"
}

PUT /eval_es/eval/3
{
  "title":"earth earth earth earth earth earth earth earth earth earth mars mars mars mars",
  "body": "earth earth earth earth earth earth earth earth earth earth earth mars mars mars mars mars mars mars mars mars"
}

PUT /eval_es/eval/4
{
  "title":"earth earth earth mars mars",
  "body": "earth earth earth earth earth earth earth earth earth earth earth earth mars mars mars mars mars mars"
}

PUT /eval_es/eval/5
{
  "title":"mars mars mars mars mars mars mars mars mars mars",
  "body": "earth earth earth earth earth earth earth earth earth earth earth earth mars mars mars mars"
}
```

## Check the documents' term vectors
```Kibana
GET /eval_es/eval/1/_termvectors
GET /eval_es/eval/2/_termvectors
GET /eval_es/eval/3/_termvectors
GET /eval_es/eval/4/_termvectors
GET /eval_es/eval/5/_termvectors
```

The term vectors shows that all field statistics are expected

## Retrieval using the 5 weighting schemes
```Kibana
GET /eval_es/eval/_search
{
    "query": {
        "query_string": {
            "query": "earth",
            "fields": ["title^10","body^0"]
        }
    }
}

GET /eval_es/eval/_search
{
    "query": {
        "query_string": {
            "query": "earth",
            "fields": ["title^7","body^3"]
        }
    }
}


GET /eval_es/eval/_search
{
    "query": {
        "query_string": {
            "query": "earth",
            "fields": ["title^5","body^5"]
        }
    }
}


GET /eval_es/eval/_search
{
    "query": {
        "multi_match": {
            "query": "earth",
            "fields": ["title^3","body^7"]
        }
    }
}

GET /eval_es/eval/_search
{
    "query": {
        "query_string": {
            "query": "earth",
            "fields": ["title^0","body^10"]
        }
    }
}

```

Results shows that ES BM25F Scores that are different that the expected BM25 Scores.
The difference made the retrieval ranking not as expected.

Next, we need to investigate how BM25F is computed in ES 5.1.1.

## BM25 in ES 5.1.1.
ES 5.1.1. uses lucene-core 6.3.0. the bm25 scoring function is described in
https://lucene.apache.org/core/6_3_0/core/org/apache/lucene/search/similarities/BM25Similarity.html

BM25 implementation code is available at:
https://github.com/apache/lucene-solr/blob/8218a5b2c6ca78c05a6060af4770c504e28a6f99/lucene/core/src/java/org/apache/lucene/search/similarities/BM25Similarity.java

Now, let use examine how Doc Id is scored using weighting scheme: Title only
```Kibana
GET /eval_es/eval/1/_explain
```

Next, we will try to reproduce every figures as produced by elastic search BM25

### IDF = 0.28768 (docFreq: 4, docCount: 5)
```java
protected float idf(long docFreq, long docCount) {
    return (float) Math.log(1 + (docCount - docFreq + 0.5D)/(docFreq + 0.5D));
  }
```
docFreq = returns the number of documents this term occurs in this