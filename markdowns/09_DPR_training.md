---
layout: tutorial
colab: https://colab.research.google.com/github/deepset-ai/haystack-tutorials/blob/main/tutorials/09_DPR_training.ipynb
toc: True
title: "Training Your Own Dense Passage Retrieval Model"
last_updated: 2022-10-12
level: "advanced"
weight: 110
description: Learn about training a Dense Passage Retrieval model and the data needed to do so.
category: "QA"
aliases: ['/tutorials/train-dpr']
---
    

# Training Your Own "Dense Passage Retrieval" Model

Haystack contains all the tools needed to train your own Dense Passage Retrieval model.
This tutorial will guide you through the steps required to create a retriever that is specifically tailored to your domain.


```python
# Install the latest release of Haystack in your own environment
#! pip install farm-haystack

# Install the latest main of Haystack
!pip install --upgrade pip
!pip install git+https://github.com/deepset-ai/haystack.git#egg=farm-haystack[colab]
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


```python
# Here are some imports that we'll need

from haystack.nodes import DensePassageRetriever
from haystack.utils import fetch_archive_from_http
from haystack.document_stores import InMemoryDocumentStore
```

## Training Data

DPR training performed using Information Retrieval data.
More specifically, you want to feed in pairs of queries and relevant documents.

To train a model, we will need a dataset that has the same format as the original DPR training data.
Each data point in the dataset should have the following dictionary structure.

``` python
    {
        "dataset": str,
        "question": str,
        "answers": list of str
        "positive_ctxs": list of dictionaries of format {'title': str, 'text': str, 'score': int, 'title_score': int, 'passage_id': str}
        "negative_ctxs": list of dictionaries of format {'title': str, 'text': str, 'score': int, 'title_score': int, 'passage_id': str}
        "hard_negative_ctxs": list of dictionaries of format {'title': str, 'text': str, 'score': int, 'title_score': int, 'passage_id': str}
    }
```

`positive_ctxs` are context passages which are relevant to the query.
In some datasets, queries might have more than one positive context
in which case you can set the `num_positives` parameter to be higher than the default 1.
Note that `num_positives` needs to be lower or equal to the minimum number of `positive_ctxs` for queries in your data.
If you have an unequal number of positive contexts per example,
you might want to generate some soft labels by retrieving similar contexts which contain the answer.

DPR is standardly trained using a method known as in-batch negatives.
This means that positive contexts for a given query are treated as negative contexts for the other queries in the batch.
Doing so allows for a high degree of computational efficiency, thus allowing the model to be trained on large amounts of data.

`negative_ctxs` is not actually used in Haystack's DPR training so we recommend you set it to an empty list.
They were used by the original DPR authors in an experiment to compare it against the in-batch negatives method.

`hard_negative_ctxs` are passages that are not relevant to the query.
In the original DPR paper, these are fetched using a retriever to find the most relevant passages to the query.
Passages which contain the answer text are filtered out.

If you'd like to convert your SQuAD format data into something that can train a DPR model,
check out the utility script at [`haystack/utils/squad_to_dpr.py`](https://github.com/deepset-ai/haystack/blob/main/haystack/utils/squad_to_dpr.py)

## Using Question Answering Data

Question Answering datasets can sometimes be used as training data.
Google's Natural Questions dataset, is sufficiently large
and contains enough unique passages, that it can be converted into a DPR training set.
This is done simply by considering answer containing passages as relevant documents to the query.

The SQuAD dataset, however, is not as suited to this use case since its question and answer pairs
are created on only a very small slice of wikipedia documents.

## Download Original DPR Training Data

WARNING: These files are large! The train set is 7.4GB and the dev set is 800MB

We can download the original DPR training data with the following cell.
Note that this data is probably only useful if you are trying to train from scratch.


```python
# Download original DPR data
# WARNING: the train set is 7.4GB and the dev set is 800MB

doc_dir = "data/tutorial9"

s3_url_train = "https://dl.fbaipublicfiles.com/dpr/data/retriever/biencoder-nq-train.json.gz"
s3_url_dev = "https://dl.fbaipublicfiles.com/dpr/data/retriever/biencoder-nq-dev.json.gz"

fetch_archive_from_http(s3_url_train, output_dir=doc_dir + "/train")
fetch_archive_from_http(s3_url_dev, output_dir=doc_dir + "/dev")
```

## Option 1: Training DPR from Scratch

The default variables that we provide below are chosen to train a DPR model from scratch.
Here, both passage and query embedding models are initialized using BERT base
and the model is trained using Google's Natural Questions dataset (in a format specialised for DPR).

If you are working in a language other than English,
you will want to initialize the passage and query embedding models with a language model that supports your language
and also provide a dataset in your language.


```python
# Here are the variables to specify our training data, the models that we use to initialize DPR
# and the directory where we'll be saving the model

train_filename = "train/biencoder-nq-train.json"
dev_filename = "dev/biencoder-nq-dev.json"

query_model = "bert-base-uncased"
passage_model = "bert-base-uncased"

save_dir = "../saved_models/dpr"
```

## Option 2: Finetuning DPR

If you have your own domain specific question answering or information retrieval dataset,
you might instead be interested in finetuning a pretrained DPR model.
In this case, you would initialize both query and passage models using the original pretrained model.
You will want to load something like this set of variables instead of the ones above


```python
# Here are the variables you might want to use instead of the set above
# in order to perform pretraining

doc_dir = "PATH_TO_YOUR_DATA_DIR"
train_filename = "TRAIN_FILENAME"
dev_filename = "DEV_FILENAME"

query_model = "facebook/dpr-question_encoder-single-nq-base"
passage_model = "facebook/dpr-ctx_encoder-single-nq-base"

save_dir = "../saved_models/dpr"
```

## Initialization

Here we want to initialize our model either with plain language model weights for training from scratch
or else with pretrained DPR weights for finetuning.
We follow the [original DPR parameters](https://github.com/facebookresearch/DPR#best-hyperparameter-settings)
for their max passage length but set max query length to 64 since queries are very rarely longer.


```python
## Initialize DPR model

retriever = DensePassageRetriever(
    document_store=InMemoryDocumentStore(),
    query_embedding_model=query_model,
    passage_embedding_model=passage_model,
    max_seq_len_query=64,
    max_seq_len_passage=256,
)
```

## Training

Let's start training and save our trained model!

On a V100 GPU, you can fit up to batch size 16 so we set gradient accumulation steps to 8 in order
to simulate the batch size 128 of the original DPR experiment.

When `embed_title=True`, the document title is prepended to the input text sequence with a `[SEP]` token
between it and document text.

When training from scratch with the above variables, 1 epoch takes around an hour and we reached the following performance:

```
loss: 0.046580662854042276
task_name: text_similarity
acc: 0.992524064068483
f1: 0.8804297774366846
acc_and_f1: 0.9364769207525838
average_rank: 0.19631619339984652
report:
                precision    recall  f1-score   support

hard_negative     0.9961    0.9961    0.9961    201887
     positive     0.8804    0.8804    0.8804      6515

     accuracy                         0.9925    208402
    macro avg     0.9383    0.9383    0.9383    208402
 weighted avg     0.9925    0.9925    0.9925    208402

```


```python
# Start training our model and save it when it is finished

retriever.train(
    data_dir=doc_dir,
    train_filename=train_filename,
    dev_filename=dev_filename,
    test_filename=dev_filename,
    n_epochs=1,
    batch_size=16,
    grad_acc_steps=8,
    save_dir=save_dir,
    evaluate_every=3000,
    embed_title=True,
    num_positives=1,
    num_hard_negatives=1,
)
```

## Loading

Loading our newly trained model is simple!


```python
reloaded_retriever = DensePassageRetriever.load(load_dir=save_dir, document_store=None)
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
