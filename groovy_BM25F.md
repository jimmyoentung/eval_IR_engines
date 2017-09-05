# Implement BM25F using function script (Groovy)

Based on "The Probabilistic Relevance Framework - BM25 and Beyond" by Robertson and Zaragoza, 2009.


## IDF
Page 17
IDF = log((N-ni+0.5)/(ni+0.5))

Where:
* N = number of documents in collections
* ni = number of documents containing terms i (in any field)

```kibana
GET /eval_es/eval/2/_explain
{
  "query": {
    "function_score": {
      "functions": [
        {
          "script_score": {
            "script": {
              "lang": "groovy",
              "params":{
                "k_1":1.2,
                "b_t":0.75,
                "b_b":0.75,
                "w_t":7,
                "w_b":3
              },
              "inline": "N=_index.numDocs(); n_i=_index['_all']['earth'].df(); IDF=Math.log(1+((N-n_i+0.5)/(n_i+0.5))); return IDF"
            }

            }
        }
      ]
    }
  }
}
```

# Get Average field length
```Kibana
POST /eval_es/_search?size=0
{
    "aggs" : {
        "avg_title" : { "avg" : { "field" : "title.length" } },
        "avg_body" : { "avg" : { "field" : "body.length" } }
    }
}

```


# Final BM25F
```kibana
GET /eval_es/eval/2/_explain
{
  "query": {
    "function_score": {
      "functions": [
        {
          "script_score": {
            "script": {
              "lang": "groovy",
              "params":{
                "k_1":1.2,
                "b_title":0.75,
                "b_body":0.75,
                "w_title":7,
                "w_body":3
              },
              "inline": "N=_index.numDocs(); n_i=_index['_all']['earth'].df(); IDF=Math.log(1+((N-n_i+0.5)/(n_i+0.5))); tf_title=_index['title']['earth'].tf(); tf_body=_index['body']['earth'].tf(); Bs_title=((1-b_title)+b_title*(doc['title.length'].value/10)); Bs_body=((1-b_body)+b_body*(doc['body.length'].value/20)); tfi= w_title*(tf_title/Bs_title) + w_body*(tf_body/Bs_body); bm25f=(tfi/(k_1+tfi))*IDF; return bm25f"
            }

            }
        }
      ]
    }
  }
}
```