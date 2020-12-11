# Studying Taxonomy Enrichment on Diachronic WordNet Versions

This repository contains code and dataset for the Taxonomy Enrichment task for English and Russian. For a new word, our methods predict the top-10 candidate hypernyms from an existing taxonomy.

Detailed description of the methods is available in the original paper:

* [Original paper](https://www.aclweb.org/anthology/2020.coling-main.276.pdf)
* [Poster](https://underline.io/events/54/posters/1346/poster/5862-2143---studying-taxonomy-enrichment-on-diachronic-wordnet-versions)
* [ArXiv](https://arxiv.org/abs/2011.11536) version

The picture below illustrates the main idea of the underlying approach:

![approach image](img/model_taxonomy.jpg)

If you use the method please cite the following paper:

```
@InProceedings{nikishina-etal-2020-studying,
  author    = {Nikishina, Irina  and  Panchenko, Alexander  and  Logacheva, Varvara  and  Loukachevitch, Natalia},
  title     = {Studying Taxonomy Enrichment on Diachronic WordNet Versions},
  booktitle = {Proceedings of the 28th International Conference on Computational Linguistics},
  month     = {December},
  year      = {2020},
  address   = {Barcelona, Spain},
  publisher = {Association for Computational Linguistics},
  pages     = {},
  url       = {},
  abstract  = {Ontologies, taxonomies, and thesauri are used in many NLP tasks. %have always been in high demand in a large number of NLP tasks. However, most studies are focused on the creation of these lexical resources rather than the maintenance of the existing ones. Thus, we address the problem of taxonomy enrichment. We explore the possibilities of taxonomy extension in a resource-poor setting and present methods which are applicable to a large number of languages. We create novel English and Russian datasets for training and evaluating taxonomy enrichment models and describe a technique of creating such datasets for other languages.}
}
```

## Datasets

We build two diachronic datasets: one for English, another one for Russian based respectively on Princeton WordNet and RuWordNet taxonomies. Each dataset consists of a taxonomy and a set of novel words not included in the taxonomy. 

### Datasets can be downloaded [here](https://doi.org/10.5281/zenodo.4279821)

Statistics:

|     Dataset       | Nouns    | Verbs |
| :-------------:   | :---:    | :---: |
| WordNet 1.6-3.0   |  17 043  |  755  |
| WordNet 1.7-3.0   |  6 161   |  362  |
| WordNet 2.0-3.0   |  2 620   |  193  |
| RuWordNet 1.0-2.0 |  14 660  | 2 154 |


## Usage

```
git clone --recursive https://github.com/skoltech-nlp/diachronic-wordnets.git
```

FastText models can be downloaded from [here](https://fasttext.cc/docs/en/crawl-vectors.html): [ru](https://dl.fbaipublicfiles.com/fasttext/vectors-crawl/cc.ru.300.bin.gz), [en](https://dl.fbaipublicfiles.com/fasttext/vectors-crawl/cc.en.300.bin.gz)

### Preprocessing

* create database (for Russian)

```
python3 ruwordnet/create_ruwordnet.py <input-dir> <output-path.db>
```

* vectorize WordNet

```
python3 fasttext_vectorize_en.py <input_dir> <in_version> <out_version>
```
The following arguments are required: `<input_dir>` `<in_version>` 
Default `<out_version>`: 3.0

* vectorize RuWordNet

```
python3 fasttext_vectorize_ru.py models/cc.ru.300.bin ../../datasets/ruwordnet.db models/vectors/fasttext/ru/ ../../datasets/ru/
```

The following arguments are required: `<fasttext_path>` `<ruwordnetdb_path>` `<output_path>` `<data_path>`

* vectorize Wiktionary

### Model run

To start the model you should run the `main.py` file with one required parameter: `<path-to-config-file>`

```
python3 main.py configs/fasttext/fasttext_private_nouns.json
```

Examples of config files are provided below for each method

### Baseline

In this method, top <img src="https://render.githubusercontent.com/render/math?math=k=10"> nearest neighbours of the input word are taken from the pre-trained embedding model (according to the above considerations they should be co-hyponyms). Subsequently,  hypernyms  of those co-hyponyms are extracted from the taxonomy. These hypernyms can also be considered hypernyms of the input word. 

```{json}
{
  "synsets_vectors_path": "models/vectors/fasttext/en/nouns_wordnet_fasttext_2.0-3.0.txt",
  "data_vectors_path": "models/vectors/fasttext/en/nouns_fasttext_2.0-3.0.txt",
  "test_path": "../../datasets/en/no_labels_nouns_en.2.0-3.0.tsv",
  "output_path": "predictions/en/fasttext/predicted_nouns_en_2.0-3.0_baseline.tsv",
  "db_path": null,
  "wordnet_path": "../../datasets/WNs/WN2.0",
  "model": "baseline",
  "language": "en"
}
```

### Ranking

We improve the described model by ranking the generated synset candidates. In addition to that, we extend the list of candidates with second-order hypernyms (hypernyms of each hypernym). The direct hypernyms of the word's nearest neighbours can be too specific, whereas second-order hypernyms are likely to be more abstract concepts, which the input word and its neighbours have in common. After forming a list of candidates, we score each of them using the following equation:

<img src="https://render.githubusercontent.com/render/math?math=score_{h_{i}} = n \cdot sim(v_o, v_{h_{i}}),">

where <img src="https://render.githubusercontent.com/render/math?math=v_x"> is a vector representation of a word or a synset <img src="https://render.githubusercontent.com/render/math?math=x">, <img src="https://render.githubusercontent.com/render/math?math=h_{i}"> is a hypernym, <img src="https://render.githubusercontent.com/render/math?math=n"> is the number of occurrences of this hypernym in the merged list, <img src="https://render.githubusercontent.com/render/math?math=sim(v_{o}, v_{h_{i}})"> is the cosine similarity of the vector of the orphan word <img src="https://render.githubusercontent.com/render/math?math=o"> and hypernym vector <img src="https://render.githubusercontent.com/render/math?math=h_{i}">.

```{json}
{
  "synsets_vectors_path": "models/vectors/fasttext/ru/ruwordnet_nouns.txt",
  "data_vectors_path": "models/vectors/fasttext/ru/nouns_private.txt",
  "test_path": "../../datasets/ru/nouns_private_no_labels.tsv",
  "output_path": "predictions/predicted_private_nouns_ranked.tsv",
  "db_path": "../../datasets/ruwordnet.db",
  "wordnet_path": null,
  "model": "ranked",
  "language": "ru"
}
```

### Ranking + wiki 

We implement the following Wiktionary features:
* candidate is present in the Wiktionary hypernyms list for the input word (binary feature),
* candidate is present in the Wiktionary synonyms list (binary feature),
* candidate is present in the Wiktionary definition (binary feature),
* average cosine similarity between the candidate and the Wiktionary hypernyms of the input word.

We extract lists of hypernym synset candidates using the baseline procedure and compute the 4 Wiktionary features for them. In addition to that, we use the score from the previous approach as a feature. To define the feature weights, we train a Logistic Regression model with L2 regularisation on a training dataset which we construct from the older (known) versions of WordNet. 

Wiki jsonlines can be founded [here](https://doi.org/10.5281/zenodo.4281515).

```{json}
{
  "synsets_vectors_path": "models/vectors/fasttext/ru/ruwordnet_nouns.txt",
  "data_vectors_path": "models/vectors/fasttext/ru/nouns_private.txt",
  "test_path": "../../datasets/ru/nouns_private_no_labels.tsv",
  "output_path": "predictions/predicted_private_nouns_lr.tsv",
  "db_path": "../../datasets/ruwordnet.db",
  "wordnet_path": null,
  "wiki_path": "../../datasets/wiki/wiki_ru.jsonlines",
  "wiki_vectors_path": "models/vectors/fasttext/ru/wiki.txt",
  "model": "lr",
  "language": "ru"
}
```

