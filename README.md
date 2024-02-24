# Elasticsearch Vectorstore

## Steps
* `sudo docker compose up`
* Run `index_with_elasticsearch_dsl.ipynb` for indexing `sample-trec` data
* Go to kibanna dev console: http://localhost:5601/app/dev_tools#/console

## Get all indices
```
GET _cat/indices
```

## verify that it has been index
```
GET sample-trec/_search
```

## Check our hugging face model
```
POST /_ml/trained_models/sentence-transformers__all-minilm-l6-v2/_infer
{
  "docs": {
    "text_field": "how is the weather in jamaica"
  }
}
```


## Creating ingestion pipeline
```
PUT _ingest/pipeline/text-embeddings
{
  "description": "Text embedding pipeline",
  "processors": [
    {
      "inference": {
        "model_id": "sentence-transformers__all-minilm-l6-v2",
        "target_field": "text_embedding",
        "field_map": {
          "content": "text_field"
        }
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "description": "Index document to 'failed-<index>'",
        "field": "_index",
        "value": "failed-{{{_index}}}"
      }
    },
    {
      "set": {
        "description": "Set error message",
        "field": "ingest.failure",
        "value": "{{_ingest.on_failure_message}}"
      }
    }
  ]
}
```

## Create embeddings index
```
PUT sample-trec-with-embeddings
{
  "mappings": {
    "properties": {
      "text_embedding.predicted_value": {
        "type": "dense_vector",
        "dims": 384,
        "index": true,
        "similarity": "cosine"
      },
      "content": {
        "type": "text"
      }
    }
  }
}
```

## Reindex data
```
POST _reindex?wait_for_completion=false
{
  "source": {
    "index": "sample-trec"
  },
  "dest": {
    "index": "sample-trec-with-embeddings",
    "pipeline": "text-embeddings"
  }
}
```

## Track progress of the task
```
GET _tasks/<task_id>
```

## Verify that the pipeline has been succesful and embeddings are created

```
GET _cat/indices
```

```
GET sample-trec-with-embeddings/_search
```

# Get an embedding of a sentence
```
POST /_ml/trained_models/sentence-transformers__all-minilm-l6-v2/_infer
{
  "docs": {
    "text_field": "This is the definition of RNA along with examples of types of RNA molecules."
  }
}
```

# Neareast embedding search
```
GET sample-trec-with-embeddings/_knn_search
{
  "knn": {
    "field": "text_embedding.predicted_value",
    "query_vector": [
        0.058760009706020355,
        -0.021021468564867973,
        -0.030731286853551865,
        0.03542604297399521,
        0.11785433441400528,
        0.006186880171298981,
        0.004657561890780926,
        ...................
        ...................
        # paste the embedding obtained in the previous step here
        -0.058936621993780136,
        0.06186993047595024,
        -0.08111287653446198,
        -0.04787856340408325,
        0.02004200965166092,
        0.003357150126248598,
        0.01042801234871149,
        0.011861362494528294,
        0.14425651729106903,
        -0.03517891466617584
      ],
    "k": 10,
    "num_candidates": 100
  },
  "_source": [
    "id",
    "content"
  ]
}
```