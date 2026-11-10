🇪🇸 [Leer en español](README.es.md)

# NLP Analysis of Infobae News

A project that builds a corpus of Spanish-language news from Infobae's RSS feed
and compares different natural language processing (NLP) techniques on that corpus.

## Question

What can be learned from a Spanish-language news corpus using NLP techniques, and how do
those techniques compare with each other?

## What the project does

1. **Data collection:** a script downloads the RSS feed and accumulates the articles in a CSV, without duplicates.
2. **Preprocessing:** cleaning and lemmatization with spaCy, keeping negations (`no`, `nunca`, `sin`...),
   which spaCy treats as stopwords.
3. **Predicting the newspaper edition** (`america`, `espana`, `mexico`, `peru`, `colombia`) from the text:
   TF-IDF vs. embeddings, both with logistic regression and 5-fold cross-validation, with a control
   for the length of text each method reads.
4. **Discovering topics without labels:** KMeans and BERTopic on embeddings, and a comparison between the two.
5. **Classifying by topic without training** (zero-shot with embeddings), evaluated against the thematic section
   that appears in the URL of some articles.

## Repository structure

```
project_news_infobae/
├── README.md
├── requirements.txt
├── .gitignore
├── src/
│   └── actualizar_corpus.py     # downloads the feed and accumulates articles in data/
├── notebooks/
│   └── analisis.ipynb           # full analysis, explained step by step
└── data/                        # the corpus is not included in the repository
```

## How to reproduce

Tested with Python 3.13.

1. Install the dependencies:

   ```bash
   pip install -r requirements.txt
   python -m spacy download es_core_news_md
   ```

2. Build the corpus by running the script **several times a day for several days**:

   ```bash
   python src/actualizar_corpus.py
   ```

   The feed only returns the latest ~100 articles per request, so the corpus depends on how often the script
   is run. On Windows it can be automated with Task Scheduler (for example, every 30 minutes).

3. Open `notebooks/analisis.ipynb` and run it from top to bottom. It reads `data/noticias_infobae.csv`.

**Important:** the results depend on the corpus. With other articles, the numbers, the number of BERTopic topics
and the numbering of the KMeans groups will change. The `nombres_km` dictionary in the notebook (which names each
group) was assigned by hand for `SEED=42` and for the original corpus, so it must be reviewed
if the corpus is different.

## About the data

The corpus contains the full text of the articles, which is copyrighted, so it is **not included
in the repository**. It is rebuilt with the script. Before using it, it is advisable to review the site's terms of use.

The results in this repository were obtained with **2,500 articles** (2,503 were downloaded; 3 without text were
discarded) published between **October 23, 2026 (19:00 UTC)** and **October 27, 2026 (03:27 UTC)**,
that is, about 80 hours.

## Results

| Task | Method | Metric | Result |
|---|---|---|---|
| Predict the edition (5 classes, 1,852 articles) | Baseline: always `america` | accuracy | 0.42 |
| | TF-IDF + logistic regression (full article) | macro F1 | 0.85 |
| | TF-IDF + logistic regression (first 90 words) | macro F1 | 0.79 |
| | Embeddings + logistic regression | macro F1 | 0.65 |
| Discover topics without labels (2,500 articles) | KMeans | chosen k / silhouette | k=10 / 0.100 |
| | BERTopic | topics / articles without a topic | 24 / 271 (11%) |
| Classify by topic without training (7 classes, 312 articles) | Baseline: always `deportes` | accuracy | 0.20 |
| | Zero-shot with embeddings | accuracy / macro F1 | 0.67 / 0.65 |

What was observed:

- **Edition:** TF-IDF beats embeddings by 14 macro F1 points with text of comparable length
  (0.79 vs. 0.65) and by 20 with the full article (0.85). The main confusion is between `america` and `espana`:
  203 of the 313 errors (65%).
- **Topics:** KMeans with k between 8 and 12 gives a similar silhouette (0.089 to 0.100). With BERTopic, 16 of the 24 topics
  have at least 75% of their articles in a single KMeans group.
- **Topic classification:** sports (`deportes`) and health (`salud`) are classified best; technology and culture lose more than half of
  their articles. The largest errors are technology → culture (15 articles, mostly YouTube, video games and AI-related content)
  and culture → crime (9 articles, four of them about the same event).

## Limitations

- The corpus covers about 80 hours, so several topics correspond to a single event published across several articles
  (a KMeans group of 193 articles is almost entirely about Brazil's runoff election). In cross-validation, articles about the
  same event may be split between training and test, which can inflate the results.
- The feed only returns the latest articles: periods when the script was not run are left uncovered.
- The hyperparameters (`C`, `class_weight`, KMeans `k`, HDBSCAN `min_cluster_size` and `min_samples`)
  were chosen by looking at results on the same corpus, so the numbers may be somewhat optimistic.
- In BERTopic, `min_cluster_size` and `min_samples` had to be set explicitly: with `min_cluster_size=10`, 82 topics
  came out, and with `min_cluster_size=30` and `min_samples=5`, a single topic gathered 514 articles. At an earlier stage,
  with 683 articles, the default value of `min_samples` produced only 4 topics.
- The topic classification labels come from the URL section (the outlet's criterion, not my own annotation),
  cover only 312 articles, and the `crimen` category has just 21.
- KMeans, BERTopic and the zero-shot approach use the same embeddings, which read only the first ~128 tokens of each article.
  Agreement among them is not an independent validation.
- The interpretation of the groups was based on a few titles per group and on keywords: it is a qualitative reading,
  not a measurement.

## Tech stack

Python, requests, BeautifulSoup, pandas, scikit-learn, spaCy, sentence-transformers
(`paraphrase-multilingual-MiniLM-L12-v2`), UMAP, HDBSCAN and BERTopic.

## Possible extensions

- Accumulate more days and evaluate with a temporal split (train on the first days, test on the last).
- Annotate my own labels, with a separate test set.
- Compare the zero-shot approach with an inference model (NLI) on the same 312 articles.
- Fine-tune a Spanish-language model to classify by topic.
- Named entity recognition (NER) to describe each edition.
