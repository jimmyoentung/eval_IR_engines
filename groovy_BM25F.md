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