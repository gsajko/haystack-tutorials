---
layout: tutorial
colab: https://colab.research.google.com/github/deepset-ai/haystack-tutorials/blob/main/tutorials/08_Preprocessing.ipynb
toc: True
title: "Preprocessing Your Documents"
last_updated: 2022-10-26
level: "beginner"
weight: 25
description: Start converting, cleaning, and splitting Documents using Haystack’s preprocessing capabilities.
category: "QA"
aliases: ['/tutorials/preprocessing']
---
    

# Preprocessing

Haystack includes a suite of tools to extract text from different file types, normalize white space
and split text into smaller pieces to optimize retrieval.
These data preprocessing steps can have a big impact on the systems performance and effective handling of data is key to getting the most out of Haystack.

Ultimately, Haystack expects data to be provided as a list documents in the following dictionary format:
``` python
docs = [
    {
        'content': DOCUMENT_TEXT_HERE,
        'meta': {'name': DOCUMENT_NAME, ...}
    }, ...
]
```

This tutorial will show you all the tools that Haystack provides to help you cast your data into this format.


```bash
%%bash

pip install --upgrade pip
pip install git+https://github.com/deepset-ai/haystack.git#egg=farm-haystack[colab,ocr]

# For Colab/linux based machines:
!wget https://dl.xpdfreader.com/xpdf-tools-linux-4.04.tar.gz
!tar -xvf xpdf-tools-linux-4.04.tar.gz && sudo cp xpdf-tools-linux-4.04/bin64/pdftotext /usr/local/bin

# For macOS machines:
# !wget https://dl.xpdfreader.com/xpdf-tools-mac-4.03.tar.gz
# !tar -xvf xpdf-tools-mac-4.03.tar.gz && sudo cp xpdf-tools-mac-4.03/bin64/pdftotext /usr/local/bin
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
from haystack.utils import fetch_archive_from_http


# This fetches some sample files to work with
doc_dir = "data/tutorial8"
s3_url = "https://s3.eu-central-1.amazonaws.com/deepset.ai-farm-qa/datasets/documents/preprocessing_tutorial8.zip"
fetch_archive_from_http(url=s3_url, output_dir=doc_dir)
```

## Converters

Haystack's converter classes are designed to help you turn files on your computer into the documents
that can be processed by the Haystack pipeline.
There are file converters for txt, pdf, docx files as well as a converter that is powered by Apache Tika.
The parameter `valid_languages` does not convert files to the target language, but checks if the conversion worked as expected. Here are some examples of how you would use file converters:


```python
from haystack.nodes import TextConverter, PDFToTextConverter, DocxToTextConverter, PreProcessor


converter = TextConverter(remove_numeric_tables=True, valid_languages=["en"])
doc_txt = converter.convert(file_path="data/tutorial8/classics.txt", meta=None)[0]

converter = PDFToTextConverter(remove_numeric_tables=True, valid_languages=["en"])
doc_pdf = converter.convert(file_path="data/tutorial8/bert.pdf", meta=None)[0]

converter = DocxToTextConverter(remove_numeric_tables=False, valid_languages=["en"])
doc_docx = converter.convert(file_path="data/tutorial8/heavy_metal.docx", meta=None)[0]
```

Haystack also has a convenience function that will automatically apply the right converter to each file in a directory:


```python
from haystack.utils import convert_files_to_docs


all_docs = convert_files_to_docs(dir_path=doc_dir)
```

## PreProcessor

The PreProcessor class is designed to help you clean text and split text into sensible units.
File splitting can have a very significant impact on the system's performance and is absolutely mandatory for Dense Passage Retrieval models.
In general, we recommend you split the text from your files into small documents of around 100 words for dense retrieval methods
and no more than 10,000 words for sparse methods.
Have a look at the [Preprocessing](https://docs.haystack.deepset.ai/docs/preprocessor)
and [Optimization](https://docs.haystack.deepset.ai/docs/optimization) pages on our website for more details.


```python
from haystack.nodes import PreProcessor


# This is a default usage of the PreProcessor.
# Here, it performs cleaning of consecutive whitespaces
# and splits a single large document into smaller documents.
# Each document is up to 1000 words long and document breaks cannot fall in the middle of sentences
# Note how the single document passed into the document gets split into 5 smaller documents

preprocessor = PreProcessor(
    clean_empty_lines=True,
    clean_whitespace=True,
    clean_header_footer=False,
    split_by="word",
    split_length=100,
    split_respect_sentence_boundary=True,
)
docs_default = preprocessor.process([doc_txt])
print(f"n_docs_input: 1\nn_docs_output: {len(docs_default)}")
```

## Cleaning

- `clean_empty_lines` will normalize 3 or more consecutive empty lines to be just a two empty lines
- `clean_whitespace` will remove any whitespace at the beginning or end of each line in the text
- `clean_header_footer` will remove any long header or footer texts that are repeated on each page

## Splitting
By default, the PreProcessor will respect sentence boundaries, meaning that documents will not start or end
midway through a sentence.
This will help reduce the possibility of answer phrases being split between two documents.
This feature can be turned off by setting `split_respect_sentence_boundary=False`.


```python
# Not respecting sentence boundary vs respecting sentence boundary

preprocessor_nrsb = PreProcessor(split_respect_sentence_boundary=False)
docs_nrsb = preprocessor_nrsb.process([doc_txt])

print("RESPECTING SENTENCE BOUNDARY")
end_text = docs_default[0].content[-50:]
print('End of document: "...' + end_text + '"')
print()
print("NOT RESPECTING SENTENCE BOUNDARY")
end_text_nrsb = docs_nrsb[0].content[-50:]
print('End of document: "...' + end_text_nrsb + '"')
```

A commonly used strategy to split long documents, especially in the field of Question Answering,
is the sliding window approach. If `split_length=10` and `split_overlap=3`, your documents will look like this:

- doc1 = words[0:10]
- doc2 = words[7:17]
- doc3 = words[14:24]
- ...

You can use this strategy by following the code below.


```python
# Sliding window approach

preprocessor_sliding_window = PreProcessor(split_overlap=3, split_length=10, split_respect_sentence_boundary=False)
docs_sliding_window = preprocessor_sliding_window.process([doc_txt])

doc1 = docs_sliding_window[0].content[:200]
doc2 = docs_sliding_window[1].content[:100]
doc3 = docs_sliding_window[2].content[:100]

print('Document 1: "' + doc1 + '..."')
print('Document 2: "' + doc2 + '..."')
print('Document 3: "' + doc3 + '..."')
```

## Bringing it all together


```python
all_docs = convert_files_to_docs(dir_path=doc_dir)
preprocessor = PreProcessor(
    clean_empty_lines=True,
    clean_whitespace=True,
    clean_header_footer=False,
    split_by="word",
    split_length=100,
    split_respect_sentence_boundary=True,
)
docs = preprocessor.process(all_docs)

print(f"n_files_input: {len(all_docs)}\nn_docs_output: {len(docs)}")
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

