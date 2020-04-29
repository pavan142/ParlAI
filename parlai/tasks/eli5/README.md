# ELI5
**Adapted from Facebook Research's ELI5 found at https://github.com/facebookresearch/ELI5/**

![GitHub license](https://img.shields.io/badge/license-BSD-blue.svg)


Read the Paper: https://arxiv.org/abs/1907.09190

Read the Blog Post: https://ai.facebook.com/blog/longform-qa/

Explore the Dataset: https://facebookresearch.github.io/ELI5/explore.html

--------------------------------------------------------------------------------

# Data creation

We provide a suite of scripts to download paired questions and answers from the ELI5 subreddit along with supporting documents from the CommonCrawl

### FAQ: can you provide the processed data?

No, we are not allowed to host processed Reddit or CommonCrawl data. While we are aware that it would make life a little easier, making your own version of the dataset following the instructions here is pretty straightforward if you have access to a SLURM cluster. **We're also happy to work with you if your cluster works on another operating system, open an issue and we'll get on it!** Since the creation process involves downloading, filtering and tokenizing a full CommonCrawl dump however, using a single machine is impractical.


## Overview of the data creation process

The process consists of four steps. *Steps 1 and 2 should be run in parallel.*

1. **Run build.py to setup the directory and download pre-computed files**. This only takes a few minutes.
2. **Downloading and filtering the Reddit data.** This can be run on a single machine and may take up to 72 hours.
3. **Downloading and tokenizing the CommonCrawl pages.** This part requires access to a cluster. We provide a sample SLURM script using 100 threads, which on our cluster finishes in under 48 hours.
4. **Selecting passages from the downloaded pages to create the final support document.** After running steps 1 and 2, this part uses our TFIDF heuristic to create the final ~1000 words support document, and create a train/valid/test split of Question-Document-Answer triplets.

If you are having trouble with any of these, please open an issue stating which step is failing in the title, and attach the Python error text, if available.

## Downloading pre-computed files for support documents
Running `python examples/display_data.py -t eli5` to run ELI5's `build.py` file
will create the directory structure and download pre-computed files such as the list of CommonCrawl IDs for supporting documents needed for the dataset. Unlike a normal `build.py` file, this will not download the ELI5 datset, and instead tell you to follow these instructions.

## Downloading the Reddit Data

The first step consists in downloading the Reddit Data from the files provided at pushshift.io for all months from 07/2011 to 07/2018. This is done by running the following
from the top-level ParlAI directory:

```
python parlai/tasks/eli5/download_reddit_qalist.py -Q
python parlai/tasks/eli5/download_reddit_qalist.py -A
```

The first line takes about 6 hours on one machine to download the questions, and the second less than 48 hours for the answers. Pushshift files are automatically removed after they've been processed, so space shouldn't be an issue there. The final product should be 689MB.


## Downloading support documents from the CommonCrawl

We provide a list of CommonCrawl IDs for supporting documents for each of the questions.
These are obtained from `build.py` which we went over above.


The next step than consists in reading through the CommonCrawl WET files to gather the text of pages which are used as support documents. In order to gather the documents for each QA pair (up to 100 per pair, sometimes less after deduplicating) using a SLURM cluster and 100 threads, run:
```
cd slurm_scripts
./eli_download_docs_launcher.sh
```
This should run in less than 48 hours. Be advised that the result is upwards of 100GB.

When you have downloaded the selected pages from all of the CommonCrawl slices, simply merge the slices from all the threads with:
```
./eli_merge_docs_launcher.sh
cd ..
python merge_support_docs.py -n explainlikeimfive -f finalize
```

### FAQ: my SLURM cluster is not very stable and some of the threads are interrupted. Do I need to re-run everything?

To check whether all slices are finished, simply look at the collected\_docs/tmp/counts\_\*.json files. All the files corresponding to completed slices should only have the string *finished*, so for example:
```
ELI5/data_creation/processed_data$ grep finished collected_docs/tmp/counts_* | wc
```
If the number of finished thread is less than 100, simply relaunch the missing ones, the code will restart them from their last saved space. To launch a single thread (e.g. 34), go to the data\_creation folder and run:
```
python download_support_docs.py -ns 34
```

## Finalizing the dataset

All that remains to do now is to map the collected passages to the question-answer pairs and, apply our provided heuristic to make a single support document to select relevant passages. From `parlai/tasks/eli5/data_creation`:
```
cd slurm_scripts
./eli_select_docs_launcher.sh
```

And finally, make the train, valid and test split with:
```
cd ..
python finalize_qda.py
rm processed_data/selected_15_1/explainlikeimfive/selected_slice_*.json
```

Congrats, you can now start working on your very own Long-Form Question Answering systems!
```
ELI5/data_creation$ ls -lh processed_data/selected_15_1/explainlikeimfive*.json
-rw-rw-r-- 1 username username 142M Oct 15 06:18 processed_data/selected_15_1/explainlikeimfive_test.json
-rw-rw-r-- 1 username username 1.3G Oct 15 06:18 processed_data/selected_15_1/explainlikeimfive_train.json
-rw-rw-r-- 1 username username  57M Oct 15 06:18 processed_data/selected_15_1/explainlikeimfive_valid.json
```

# Citation

Please cite as:
```bibtex
@inproceedings{fan2019eli5,
  title = {ELI5: Long Form Question Answering},
  author = {Angela Fan and Yacine Jernite and Ethan Perez and David Grangier and Jason Weston and Michael Auli},
  booktitle = {Proceedings of ACL 2019},
  year = {2019},
}
```


# Downloading General Reddit or Common Crawl Data

## download_reddit_qalist.py

The following download the post text or comments, respectively, of reddit posts with base36 ids [post_ids.json]. [post_ids.json] should be .json file of a list of ids.
```
python download_reddit_qalist.py -Q -id_l [post_ids.json]
python download_reddit_qalist.py -A --id_list [post_ids.json]
```

## download_support_docs.py
The following download Common Crawl text of URLs contained in [url_list.json] that are in Common Crawl dumps in `wet.paths`. [url_list.json] should be .json file of a list of URLs.
```
python download_support_docs.py -u [url_list.json]
python download_support_docs.py --urls [url_list.json]
```

The following download Common Crawl text of URLs contained in [id_list.json] that are in Common Crawl dumps in `wet.paths`. [id_list.json] should be .json file of a list of URLs.
```
python download_support_docs.py -ids [id_list.json]
python download_support_docs.py --ccuids [id_list.json]
```


