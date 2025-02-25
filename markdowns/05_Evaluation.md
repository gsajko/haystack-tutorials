---
layout: tutorial
colab: https://colab.research.google.com/github/deepset-ai/haystack-tutorials/blob/main/tutorials/05_Evaluation.ipynb
toc: True
title: "Evaluation of a QA System"
last_updated: 2022-10-12
level: "advanced"
weight: 100
description: Learn how to evaluate the performance of individual nodes as well as entire pipelines.
category: "QA"
aliases: ['/tutorials/evaluation']
---
    

# Evaluation of a Pipeline and its Components

To be able to make a statement about the quality of results a question-answering pipeline or any other pipeline in haystack produces, it is important to evaluate it. Furthermore, evaluation allows determining which components of the pipeline can be improved.
The results of the evaluation can be saved as CSV files, which contain all the information to calculate additional metrics later on or inspect individual predictions.

### Prepare environment

#### Colab: Enable the GPU runtime
Make sure you enable the GPU runtime to experience decent speed in this tutorial.
**Runtime -> Change Runtime type -> Hardware accelerator -> GPU**

<img src="https://raw.githubusercontent.com/deepset-ai/haystack/main/docs/img/colab_gpu_runtime.jpg">

You can double check whether the GPU runtime is enabled with the following command:


```bash
%%bash

nvidia-smi
```

To start, install the latest release of Haystack with `pip`:


```bash
%%bash

pip install --upgrade pip
pip install git+https://github.com/deepset-ai/haystack.git#egg=farm-haystack[colab]
```

## Logging

We configure how logging messages should be displayed and which log level should be used before importing Haystack.
Example log message:
INFO - haystack.utils.preprocessing -  Converting data/tutorial1/218_Olenna_Tyrell.txt
Default log level in basicConfig is WARNING so the explicit parameter is not necessary but can be changed easily:


```python
import logging

logging.basicConfig(format="%(levelname)s - %(name)s -  %(message)s", level=logging.WARNING)
logging.getLogger("haystack").setLevel(logging.INFO)
```

## Start an Elasticsearch server

You can start Elasticsearch on your local machine instance using Docker:


```python
# Recommended: Start Elasticsearch using Docker via the Haystack utility function
from haystack.utils import launch_es

launch_es()
```

If Docker is not readily available in your environment (eg., in Colab notebooks), then you can manually download and execute Elasticsearch from source:


```bash
%%bash

wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.9.2-linux-x86_64.tar.gz -q
tar -xzf elasticsearch-7.9.2-linux-x86_64.tar.gz
chown -R daemon:daemon elasticsearch-7.9.2
```


```bash
%%bash --bg

sudo -u daemon -- elasticsearch-7.9.2/bin/elasticsearch
```

Wait 30 seconds only to be sure Elasticsearch is ready before continuing:


```python
import time

time.sleep(30)
```

## Fetch, Store And Preprocess the Evaluation Dataset


```python
from haystack.utils import fetch_archive_from_http


# Download evaluation data, which is a subset of Natural Questions development set containing 50 documents with one question per document and multiple annotated answers
doc_dir = "data/tutorial5"
s3_url = "https://s3.eu-central-1.amazonaws.com/deepset.ai-farm-qa/datasets/nq_dev_subset_v2.json.zip"
fetch_archive_from_http(url=s3_url, output_dir=doc_dir)
```


```python
import os

from haystack.document_stores import ElasticsearchDocumentStore


# make sure these indices do not collide with existing ones, the indices will be wiped clean before data is inserted
doc_index = "tutorial5_docs"
label_index = "tutorial5_labels"

# Get the host where Elasticsearch is running, default to localhost
host = os.environ.get("ELASTICSEARCH_HOST", "localhost")

# Connect to Elasticsearch
document_store = ElasticsearchDocumentStore(
    host=host,
    username="",
    password="",
    index=doc_index,
    label_index=label_index,
    embedding_field="emb",
    embedding_dim=768,
    excluded_meta_data=["emb"],
)
```


```python
from haystack.nodes import PreProcessor

# Add evaluation data to Elasticsearch Document Store
# We first delete the custom tutorial indices to not have duplicate elements
# and also split our documents into shorter passages using the PreProcessor
preprocessor = PreProcessor(
    split_by="word",
    split_length=200,
    split_overlap=0,
    split_respect_sentence_boundary=False,
    clean_empty_lines=False,
    clean_whitespace=False,
)
document_store.delete_documents(index=doc_index)
document_store.delete_documents(index=label_index)

# The add_eval_data() method converts the given dataset in json format into Haystack document and label objects. Those objects are then indexed in their respective document and label index in the document store. The method can be used with any dataset in SQuAD format.
document_store.add_eval_data(
    filename="data/tutorial5/nq_dev_subset_v2.json",
    doc_index=doc_index,
    label_index=label_index,
    preprocessor=preprocessor,
)
```

## Initialize the Two Components of an ExtractiveQAPipeline: Retriever and Reader


```python
# Initialize Retriever
from haystack.nodes import BM25Retriever

retriever = BM25Retriever(document_store=document_store)

# Alternative: Evaluate dense retrievers (EmbeddingRetriever or DensePassageRetriever)
# The EmbeddingRetriever uses a single transformer based encoder model for query and document.
# In contrast, DensePassageRetriever uses two separate encoders for both.

# Please make sure the "embedding_dim" parameter in the DocumentStore above matches the output dimension of your models!
# Please also take care that the PreProcessor splits your files into chunks that can be completely converted with
#        the max_seq_len limitations of Transformers
# The SentenceTransformer model "sentence-transformers/multi-qa-mpnet-base-dot-v1" generally works well with the EmbeddingRetriever on any kind of English text.
# For more information and suggestions on different models check out the documentation at: https://www.sbert.net/docs/pretrained_models.html

# from haystack.retriever import EmbeddingRetriever, DensePassageRetriever
# retriever = EmbeddingRetriever(document_store=document_store,
#                                embedding_model="sentence-transformers/multi-qa-mpnet-base-dot-v1")
# retriever = DensePassageRetriever(document_store=document_store,
#                                   query_embedding_model="facebook/dpr-question_encoder-single-nq-base",
#                                   passage_embedding_model="facebook/dpr-ctx_encoder-single-nq-base",
#                                   use_gpu=True,
#                                   max_seq_len_passage=256,
#                                   embed_title=True)
# document_store.update_embeddings(retriever, index=doc_index)
```


```python
# Initialize Reader
from haystack.nodes import FARMReader

reader = FARMReader("deepset/roberta-base-squad2", top_k=4, return_no_answer=True)

# Define a pipeline consisting of the initialized retriever and reader
from haystack.pipelines import ExtractiveQAPipeline

pipeline = ExtractiveQAPipeline(reader=reader, retriever=retriever)

# The evaluation also works with any other pipeline.
# For example you could use a DocumentSearchPipeline as an alternative:

# from haystack.pipelines import DocumentSearchPipeline
# pipeline = DocumentSearchPipeline(retriever=retriever)
```

## Evaluation of an ExtractiveQAPipeline
Here we evaluate retriever and reader in open domain fashion on the full corpus of documents i.e. a document is considered
correctly retrieved if it contains the gold answer string within it. The reader is evaluated based purely on the
predicted answer string, regardless of which document this came from and the position of the extracted span.

The generation of predictions is separated from the calculation of metrics. This allows you to run the computation-heavy model predictions only once and then iterate flexibly on the metrics or reports you want to generate.



```python
from haystack.schema import EvaluationResult, MultiLabel

# We can load evaluation labels from the document store
# We are also opting to filter out no_answer samples
eval_labels = document_store.get_all_labels_aggregated(drop_negative_labels=True, drop_no_answers=True)

## Alternative: Define queries and labels directly

# eval_labels = [
#    MultiLabel(
#        labels=[
#            Label(
#                query="who is written in the book of life",
#                answer=Answer(
#                    answer="every person who is destined for Heaven or the World to Come",
#                    offsets_in_context=[Span(374, 434)]
#                ),
#                document=Document(
#                    id='1b090aec7dbd1af6739c4c80f8995877-0',
#                    content_type="text",
#                    content='Book of Life - wikipedia Book of Life Jump to: navigation, search This article is
#                       about the book mentioned in Christian and Jewish religious teachings...'
#                ),
#                is_correct_answer=True,
#                is_correct_document=True,
#                origin="gold-label"
#            )
#        ]
#    )
# ]

# Similar to pipeline.run() we can execute pipeline.eval()
eval_result = pipeline.eval(labels=eval_labels, params={"Retriever": {"top_k": 5}})
```


```python
# The EvaluationResult contains a pandas dataframe for each pipeline node.
# That's why there are two dataframes in the EvaluationResult of an ExtractiveQAPipeline.

retriever_result = eval_result["Retriever"]
retriever_result.head()
```


```python
reader_result = eval_result["Reader"]
reader_result.head()
```


```python
# We can filter for all documents retrieved for a given query
query = "who is written in the book of life"
retriever_book_of_life = retriever_result[retriever_result["query"] == query]
```


```python
# We can also filter for all answers predicted for a given query
reader_book_of_life = reader_result[reader_result["query"] == query]
```


```python
# Save the evaluation result so that we can reload it later and calculate evaluation metrics without running the pipeline again.
eval_result.save("../")
```

## Calculating Evaluation Metrics
Load an EvaluationResult to quickly calculate standard evaluation metrics for all predictions,
such as F1-score of each individual prediction of the Reader node or recall of the retriever.
To learn more about the metrics, see [Evaluation Metrics](https://haystack.deepset.ai/guides/evaluation#metrics-retrieval)


```python
saved_eval_result = EvaluationResult.load("../")
metrics = saved_eval_result.calculate_metrics()
print(f'Retriever - Recall (single relevant document): {metrics["Retriever"]["recall_single_hit"]}')
print(f'Retriever - Recall (multiple relevant documents): {metrics["Retriever"]["recall_multi_hit"]}')
print(f'Retriever - Mean Reciprocal Rank: {metrics["Retriever"]["mrr"]}')
print(f'Retriever - Precision: {metrics["Retriever"]["precision"]}')
print(f'Retriever - Mean Average Precision: {metrics["Retriever"]["map"]}')

print(f'Reader - F1-Score: {metrics["Reader"]["f1"]}')
print(f'Reader - Exact Match: {metrics["Reader"]["exact_match"]}')
```

## Generating an Evaluation Report
A summary of the evaluation results can be printed to get a quick overview. It includes some aggregated metrics and also shows a few wrongly predicted examples.


```python
pipeline.print_eval_report(saved_eval_result)
```

## Advanced Evaluation Metrics
As an advanced evaluation metric, semantic answer similarity (SAS) can be calculated. This metric takes into account whether the meaning of a predicted answer is similar to the annotated gold answer rather than just doing string comparison.
To this end SAS relies on pre-trained models. For English, we recommend "cross-encoder/stsb-roberta-large", whereas for German we recommend "deepset/gbert-large-sts". A good multilingual model is "sentence-transformers/paraphrase-multilingual-mpnet-base-v2".
More info on this metric can be found in our [paper](https://arxiv.org/abs/2108.06130) or in our [blog post](https://www.deepset.ai/blog/semantic-answer-similarity-to-evaluate-qa).


```python
advanced_eval_result = pipeline.eval(
    labels=eval_labels, params={"Retriever": {"top_k": 5}}, sas_model_name_or_path="cross-encoder/stsb-roberta-large"
)

metrics = advanced_eval_result.calculate_metrics()
print(metrics["Reader"]["sas"])
```

## Isolated Evaluation Mode
The isolated node evaluation uses labels as input to the Reader node instead of the output of the preceeding Retriever node.
Thereby, we can additionally calculate the upper bounds of the evaluation metrics of the Reader. Note that even with isolated evaluation enabled, integrated evaluation will still be running.



```python
eval_result_with_upper_bounds = pipeline.eval(
    labels=eval_labels, params={"Retriever": {"top_k": 5}, "Reader": {"top_k": 5}}, add_isolated_node_eval=True
)
```


```python
pipeline.print_eval_report(eval_result_with_upper_bounds)
```

## Advanced Label Scopes
Answers are considered correct if the predicted answer matches the gold answer in the labels. Documents are considered correct if the predicted document ID matches the gold document ID in the labels. Sometimes, these simple definitions of "correctness" are not sufficient. There are cases where you want to further specify the "scope" within which an answer or a document is considered correct. For this reason, `EvaluationResult.calculate_metrics()` offers the parameters `answer_scope` and `document_scope`.

Say you want to ensure that an answer is only considered correct if it stems from a specific context of surrounding words. This is especially useful if your answer is very short, like a date (for example, "2011") or a place ("Berlin"). Such short answer might easily appear in multiple completely different contexts. Some of those contexts might perfectly fit the actual question and answer it. Some others might not: they don't relate to the question at all but still contain the answer string. In that case, you might want to ensure that only answers that stem from the correct context are considered correct. To do that, specify `answer_scope="context"` in `calculate_metrics()`. 

`answer_scope` takes the following values:
- `any` (default): Any matching answer is considered correct.
- `context`: The answer is only considered correct if its context matches as well. It uses fuzzy matching (see `context_matching` parameters of `pipeline.eval()`).
- `document_id`: The answer is only considered correct if its document ID matches as well. You can specify a custom document ID through the `custom_document_id_field` parameter of `pipeline.eval()`.
- `document_id_and_context`: The answer is only considered correct if its document ID and its context match as well.

In Question Answering, to enforce that the retrieved document is considered correct whenever the answer is correct, set `document_scope` to `answer` or `document_id_or_answer`.

`document_scope` takes the following values:
- `document_id`: Specifies that the document ID must match. You can specify a custom document ID through the `custom_document_id_field` parameter of `pipeline.eval()`.
- `context`: Specifies that the content of the document must match. It uses fuzzy matching (see the `context_matching` parameters of `pipeline.eval()`).
- `document_id_and_context`: A Boolean operation specifying that both `'document_id' AND 'context'` must match.
- `document_id_or_context`: A Boolean operation specifying that either `'document_id' OR 'context'` must match.
- `answer`: Specifies that the document contents must include the answer. The selected `answer_scope` is enforced.
- `document_id_or_answer` (default): A Boolean operation specifying that either `'document_id' OR 'answer'` must match.


```python
metrics = saved_eval_result.calculate_metrics(answer_scope="context")
print(f'Retriever - Recall (single relevant document): {metrics["Retriever"]["recall_single_hit"]}')
print(f'Retriever - Recall (multiple relevant documents): {metrics["Retriever"]["recall_multi_hit"]}')
print(f'Retriever - Mean Reciprocal Rank: {metrics["Retriever"]["mrr"]}')
print(f'Retriever - Precision: {metrics["Retriever"]["precision"]}')
print(f'Retriever - Mean Average Precision: {metrics["Retriever"]["map"]}')

print(f'Reader - F1-Score: {metrics["Reader"]["f1"]}')
print(f'Reader - Exact Match: {metrics["Reader"]["exact_match"]}')
```


```python
document_store.get_all_documents()[0]
```


```python
# Let's try Document Retrieval on a file level (it's sufficient if the correct file identified by its name (for example, 'Book of Life') was retrieved).
eval_result_custom_doc_id = pipeline.eval(
    labels=eval_labels, params={"Retriever": {"top_k": 5}}, custom_document_id_field="name"
)
metrics = eval_result_custom_doc_id.calculate_metrics(document_scope="document_id")
print(f'Retriever - Recall (single relevant document): {metrics["Retriever"]["recall_single_hit"]}')
print(f'Retriever - Recall (multiple relevant documents): {metrics["Retriever"]["recall_multi_hit"]}')
print(f'Retriever - Mean Reciprocal Rank: {metrics["Retriever"]["mrr"]}')
print(f'Retriever - Precision: {metrics["Retriever"]["precision"]}')
print(f'Retriever - Mean Average Precision: {metrics["Retriever"]["map"]}')
```


```python
# Let's enforce the context again:
metrics = eval_result_custom_doc_id.calculate_metrics(document_scope="document_id_and_context")
print(f'Retriever - Recall (single relevant document): {metrics["Retriever"]["recall_single_hit"]}')
print(f'Retriever - Recall (multiple relevant documents): {metrics["Retriever"]["recall_multi_hit"]}')
print(f'Retriever - Mean Reciprocal Rank: {metrics["Retriever"]["mrr"]}')
print(f'Retriever - Precision: {metrics["Retriever"]["precision"]}')
print(f'Retriever - Mean Average Precision: {metrics["Retriever"]["map"]}')
```

## Storing results in MLflow
Storing evaluation results in CSVs is fine but not enough if you want to compare and track multiple evaluation runs. MLflow is a handy tool when it comes to tracking experiments. So we decided to use it to track all of `Pipeline.eval()` with reproducability of your experiments in mind.

### Host your own MLflow or use deepset's public MLflow

If you don't want to use deepset's public MLflow instance under https://public-mlflow.deepset.ai, you can easily host it yourself.


```python
# !pip install mlflow
# !mlflow server --serve-artifacts
```

### Preprocessing the dataset
Preprocessing the dataset works a bit differently than before. Instead of directly generating documents (and labels) out of a SQuAD file, we first save them to disk. This is necessary to experiment with different indexing pipelines. 


```python
import tempfile
from pathlib import Path
from haystack.nodes import PreProcessor
from haystack.document_stores import InMemoryDocumentStore

document_store = InMemoryDocumentStore()

label_preprocessor = PreProcessor(
    split_length=200,
    split_overlap=0,
    split_respect_sentence_boundary=False,
    clean_empty_lines=False,
    clean_whitespace=False,
)

# The add_eval_data() method converts the given dataset in json format into Haystack document and label objects.
# Those objects are then indexed in their respective document and label index in the document store.
# The method can be used with any dataset in SQuAD format.
# We only use it to get the evaluation set labels and the corpus files.
document_store.add_eval_data(
    filename="data/tutorial5/nq_dev_subset_v2.json",
    doc_index=document_store.index,
    label_index=document_store.label_index,
    preprocessor=label_preprocessor,
)

# the evaluation set to evaluate the pipelines on
evaluation_set_labels = document_store.get_all_labels_aggregated(drop_negative_labels=True, drop_no_answers=True)

# Pipelines need files as input to be able to test different preprocessors.
# Even though this looks a bit cumbersome to write the documents back to files we gain a lot of evaluation potential and reproducibility.
docs = document_store.get_all_documents()
temp_dir = tempfile.TemporaryDirectory()
file_paths = []
for doc in docs:
    file_name = doc.id + ".txt"
    file_path = Path(temp_dir.name) / file_name
    file_paths.append(file_path)
    with open(file_path, "w") as f:
        f.write(doc.content)
file_metas = [d.meta for d in docs]
```

### Run experiments
In this experiment we evaluate extractive QA pipelines with two different retrievers on the evaluation set given the corpus:
**ElasticsearchRetriever vs. EmbeddingRetriever**


```python
from haystack.nodes import BM25Retriever, EmbeddingRetriever, FARMReader, TextConverter
from haystack.pipelines import Pipeline
from haystack.document_stores import ElasticsearchDocumentStore
```


```python
# helper function to create query and index pipeline
def create_pipelines(document_store, preprocessor, retriever, reader):
    query_pipeline = Pipeline()
    query_pipeline.add_node(component=retriever, inputs=["Query"], name="Retriever")
    query_pipeline.add_node(component=reader, inputs=["Retriever"], name="Reader")
    index_pipeline = Pipeline()
    index_pipeline.add_node(component=TextConverter(), inputs=["File"], name="TextConverter")
    index_pipeline.add_node(component=preprocessor, inputs=["TextConverter"], name="Preprocessor")
    index_pipeline.add_node(component=retriever, inputs=["Preprocessor"], name="Retriever")
    index_pipeline.add_node(component=document_store, inputs=["Retriever"], name="DocumentStore")
    return query_pipeline, index_pipeline
```


```python
# Name of the experiment in MLflow
EXPERIMENT_NAME = "haystack-tutorial-5"
```

#### Run using BM25Retriever


```python
document_store = ElasticsearchDocumentStore(host=host, index="sparse_index", recreate_index=True)
preprocessor = PreProcessor(
    split_length=200,
    split_overlap=0,
    split_respect_sentence_boundary=False,
    clean_empty_lines=False,
    clean_whitespace=False,
)
es_retriever = BM25Retriever(document_store=document_store)
reader = FARMReader("deepset/roberta-base-squad2", top_k=3, return_no_answer=True, batch_size=8)
query_pipeline, index_pipeline = create_pipelines(document_store, preprocessor, es_retriever, reader)

sparse_eval_result = Pipeline.execute_eval_run(
    index_pipeline=index_pipeline,
    query_pipeline=query_pipeline,
    evaluation_set_labels=evaluation_set_labels,
    corpus_file_paths=file_paths,
    corpus_file_metas=file_metas,
    experiment_name=EXPERIMENT_NAME,
    experiment_run_name="sparse",
    corpus_meta={"name": "nq_dev_subset_v2.json"},
    evaluation_set_meta={"name": "nq_dev_subset_v2.json"},
    pipeline_meta={"name": "sparse-pipeline"},
    add_isolated_node_eval=True,
    experiment_tracking_tool="mlflow",
    experiment_tracking_uri="https://public-mlflow.deepset.ai",
    reuse_index=True,
)
```

#### Run using EmbeddingRetriever


```python
document_store = ElasticsearchDocumentStore(host=host, index="dense_index", recreate_index=True)
emb_retriever = EmbeddingRetriever(
    document_store=document_store,
    model_format="sentence_transformers",
    embedding_model="sentence-transformers/multi-qa-mpnet-base-dot-v1",
    batch_size=8,
)
query_pipeline, index_pipeline = create_pipelines(document_store, preprocessor, emb_retriever, reader)

dense_eval_result = Pipeline.execute_eval_run(
    index_pipeline=index_pipeline,
    query_pipeline=query_pipeline,
    evaluation_set_labels=evaluation_set_labels,
    corpus_file_paths=file_paths,
    corpus_file_metas=file_metas,
    experiment_name=EXPERIMENT_NAME,
    experiment_run_name="embedding",
    corpus_meta={"name": "nq_dev_subset_v2.json"},
    evaluation_set_meta={"name": "nq_dev_subset_v2.json"},
    pipeline_meta={"name": "embedding-pipeline"},
    add_isolated_node_eval=True,
    experiment_tracking_tool="mlflow",
    experiment_tracking_uri="https://public-mlflow.deepset.ai",
    reuse_index=True,
    answer_scope="context",
)
```

You can now open MLflow (e.g. https://public-mlflow.deepset.ai/ if you used the public one hosted by deepset) and look for the haystack-eval-experiment experiment. Try out mlflow's compare function and have fun...

Note that on our public mlflow instance we are not able to log artifacts like the evaluation results or the piplines.yaml file.

## Evaluation of Individual Components: Retriever
Sometimes you might want to evaluate individual components, for example, if you don't have a pipeline but only a retriever or a reader with a model that you trained yourself.
Here we evaluate only the retriever, based on whether the gold_label document is retrieved.


```python
## Evaluate Retriever on its own
# Note that no_answer samples are omitted when evaluation is performed with this method
retriever_eval_results = retriever.eval(top_k=5, label_index=label_index, doc_index=doc_index)
# Retriever Recall is the proportion of questions for which the correct document containing the answer is
# among the correct documents
print("Retriever Recall:", retriever_eval_results["recall"])
# Retriever Mean Avg Precision rewards retrievers that give relevant documents a higher rank
print("Retriever Mean Avg Precision:", retriever_eval_results["map"])
```

Just as a sanity check, we can compare the recall from `retriever.eval()` with the multi hit recall from `pipeline.eval(add_isolated_node_eval=True)`.
These two recall metrics are only comparable since we chose to filter out no_answer samples when generating eval_labels and setting document_scope to `"document_id"`. Per default `calculate_metrics()` has document_scope set to `"document_id_or_answer"` which interprets documents as relevant if they either match the gold document ID or contain the answer.


```python
metrics = eval_result_with_upper_bounds.calculate_metrics(document_scope="document_id")
print(metrics["Retriever"]["recall_multi_hit"])
```

## Evaluation of Individual Components: Reader
Here we evaluate only the reader in a closed domain fashion i.e. the reader is given one query
and its corresponding relevant document and metrics are calculated on whether the right position in this text is selected by
the model as the answer span (i.e. SQuAD style)


```python
# Evaluate Reader on its own
reader_eval_results = reader.eval(document_store=document_store, label_index=label_index, doc_index=doc_index)
top_n = reader_eval_results["top_n"]
# Evaluation of Reader can also be done directly on a SQuAD-formatted file without passing the data to Elasticsearch
# reader_eval_results = reader.eval_on_file("../data/nq", "nq_dev_subset_v2.json", device=device)

# Reader Top-N-Accuracy is the proportion of predicted answers that match with their corresponding correct answer including no_answers
print(f"Reader Top-{top_n}-Accuracy:", reader_eval_results["top_n_accuracy"])
# Reader Top-1-Exact Match is the proportion of questions where the first predicted answer is exactly the same as the correct answer including no_answers
print("Reader Top-1-Exact Match:", reader_eval_results["EM"])
# Reader Top-1-F1-Score is the average overlap between the first predicted answers and the correct answers including no_answers
print("Reader Top-1-F1-Score:", reader_eval_results["f1"])
# Reader Top-N-Accuracy is the proportion of predicted answers that match with their corresponding correct answer excluding no_answers
print(f"Reader Top-{top_n}-Accuracy (without no_answers):", reader_eval_results["top_n_accuracy_text_answer"])
# Reader Top-N-Exact Match is the proportion of questions where the predicted answer within the first n results is exactly the same as the correct answer excluding no_answers (no_answers are always present within top n).
print(f"Reader Top-{top_n}-Exact Match (without no_answers):", reader_eval_results["top_n_EM_text_answer"])
# Reader Top-N-F1-Score is the average overlap between the top n predicted answers and the correct answers excluding no_answers (no_answers are always present within top n).
print(f"Reader Top-{top_n}-F1-Score (without no_answers):", reader_eval_results["top_n_f1_text_answer"])
```

Just as a sanity check, we can compare the top-n exact_match and f1 metrics from `reader.eval()` with the exact_match and f1 from `pipeline.eval(add_isolated_node_eval=True)`.
These two approaches return the same values because pipeline.eval() calculates top-n metrics per default. Small discrepancies might occur due to string normalization in pipeline.eval()'s answer-to-label comparison. reader.eval() does not use string normalization.


```python
metrics = eval_result_with_upper_bounds.calculate_metrics(eval_mode="isolated")
print(metrics["Reader"]["exact_match"])
print(metrics["Reader"]["f1"])
```

## About us

This [Haystack](https://github.com/deepset-ai/haystack/) notebook was made with love by [deepset](https://deepset.ai/) in Berlin, Germany

We bring NLP to the industry via open source!  
Our focus: Industry specific language models & large scale QA systems.  
  
Some of our other work: 
- [German BERT](https://deepset.ai/german-bert)
- [GermanQuAD and GermanDPR](https://deepset.ai/germanquad)
- [FARM](https://github.com/deepset-ai/FARM)

Get in touch:
[Twitter](https://twitter.com/deepset_ai) | [LinkedIn](https://www.linkedin.com/company/deepset-ai/) | [Discord](https://haystack.deepset.ai/community/join) | [GitHub Discussions](https://github.com/deepset-ai/haystack/discussions) | [Website](https://deepset.ai)

By the way: [we're hiring!](https://www.deepset.ai/jobs)
