---
layout: default
title: "Building a Spam Classifier — The Target Model"
date: 2026-06-19
description: "Step-by-step guide to building a Naive Bayes email spam classifier with Python. The model we will use in future articles to demonstrate AI/ML attacks."
tags: [machine-learning, python, spam-detection, naive-bayes, nlp, beginner]
permalink: /blog/building-spam-classifier-target-model/
---

Note : You can find the file and dataset [On my Github](https://github.com/UsualTitan/Model-Spam-ham-email-classifier)

## Introduction

Before you can attack a model, you need a model worth attacking.

This article walks through building a simple but realistic **email spam classifier** from scratch using Python. No prior machine learning experience required — just a working Python environment and a bit of curiosity.

The goal here is not to build the most sophisticated spam filter ever written. The goal is to build a **clean, understandable baseline model** that we can use in future articles to demonstrate how machine learning systems can be attacked: data poisoning, adversarial inputs, model theft, and more.

By the end of this article, you will have:

- A trained Multinomial Naive Bayes classifier that distinguishes spam from ham
- A saved model and vectorizer you can reload without retraining
- A clear understanding of every step and why it matters

---

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
  - [Environment Setup](#environment-setup)
  - [Installing Dependencies](#installing-dependencies)
- [The Dataset](#the-dataset)
- [Step 1 — Loading and Exploring the Data](#step-1--loading-and-exploring-the-data)
  - [Understanding the Class Imbalance](#understanding-the-class-imbalance)
- [Step 2 — Preprocessing the Text](#step-2--preprocessing-the-text)
  - [Why Preprocessing Matters](#why-preprocessing-matters)
  - [The Cleaning Pipeline](#the-cleaning-pipeline)
- [Step 3 — Vectorization](#step-3--vectorization)
  - [From Words to Numbers](#from-words-to-numbers)
- [Step 4 — Training the Model](#step-4--training-the-model)
  - [Why Naive Bayes for Text?](#why-naive-bayes-for-text)
  - [Saving the Model](#saving-the-model)
- [Step 5 — Evaluating the Model](#step-5--evaluating-the-model)
  - [Reading the Confusion Matrix](#reading-the-confusion-matrix)
  - [Reading the Classification Report](#reading-the-classification-report)
- [Step 6 — Testing on New Emails](#step-6--testing-on-new-emails)
- [Key Takeaways](#key-takeaways)
- [What's Next](#whats-next)

---

## Prerequisites

### Environment Setup

You need a working Python environment. Two options:

**Option A — Jupyter Notebook (local)**
This is what I use. Install it via pip or Anaconda:

```bash
pip install notebook
jupyter notebook
```

Then open a new notebook and you are ready to go.

**Option B — Google Colab (browser-based)**
If you don't want to install anything locally, [Google Colab](https://colab.research.google.com) is the simplest way to get started. It's free, runs in your browser, and comes with most scientific Python libraries pre-installed. Just upload the dataset, create a new notebook, and follow along.

Either environment will work for everything in this article.

### Installing Dependencies

We will use the following libraries. Most are available by default in Colab; for a local setup, install them if needed:

```bash
pip install pandas numpy matplotlib scikit-learn seaborn nltk
```

Then download the NLTK stop words corpus (only needed once):

```python
import nltk
nltk.download('stopwords')
```

---

## The Dataset

We are using a public email dataset containing **5,728 emails**, each labelled as either spam (1) or ham (0). The dataset is a single CSV file with two columns:

- `text` — the raw email content, including the subject line
- `spam` — a binary label: 1 for spam, 0 for legitimate email

{% include image.html src="/images/blog/2026-06-19-building-classifier-model-to-attack/screen1.png" width="600px" %}

You can download the dataset here: `[LINK — emails.csv]`

Place it in the same directory as your notebook (or upload it to Colab using the file upload sidebar).

---

## Step 1 — Loading and Exploring the Data

Start by importing the libraries we will use throughout the notebook:

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
```

Now load the dataset and take a quick look at what we have:

```python
data = pd.read_csv('emails.csv')
print(data.head(5))
```

{% include image.html src="/images/blog/2026-06-19-building-classifier-model-to-attack/screen1.png" width="600px" %}

```python
print(data.info())
```

![image](/images/blog/2026-06-19-building-classifier-model-to-attack/screen2.png)

```python
print(data.describe())
```

![image](/images/blog/2026-06-19-building-classifier-model-to-attack/screen3.png)

The dataset is clean: no missing values, two columns, binary labels.

### Understanding the Class Imbalance

Let's split the data by class and count:

```python
ham = data[data['spam'] == 0]
spam = data[data['spam'] == 1]
print(f"ham count is {len(ham)} and spam count is {len(spam)}")
```

You will see something like:

```
ham count is 4360 and spam count is 1368
```

This is a **~76% / 24% split** — there are more than three legitimate emails for every spam one. This is actually quite realistic; in the wild, most email is not spam (depending heavily on the mailbox).

Why does this matter? A model that simply predicts "ham" for every single email would already be 76% accurate. That sounds good until you realize it would let every piece of spam through. **Accuracy alone is a misleading metric here**, which is why we will use precision, recall, and F1 when evaluating.

During training, we will use `stratify=Y` when splitting the data to ensure both the training and test sets preserve this ratio. More on that in Step 4.

---

## Step 2 — Preprocessing the Text

Raw email text cannot go directly into a machine learning model. Models work with numbers, not words. But before we even get to numbers, we need to **clean the text** so that the model can learn from it without being confused by noise.

### Why Preprocessing Matters

Consider these two email fragments:

```
"WIN a FREE prize!!! CLAIM NOW!!!"
"win a free prize claim now"
```

To a human these are obviously the same phrase. To a model looking at raw text, the exclamation marks and capitalization make them look completely different. Preprocessing collapses that gap.

Here is what we do and why:

| Step                             | What it does                               | Why                                                                                        |
| -------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------ |
| Remove non-alphabetic characters | Strips punctuation, numbers, and symbols   | `!!!`, `$$$`, and `1000` add noise without discriminative value                            |
| Lowercase                        | Converts everything to lowercase           | `"FREE"` and `"free"` are the same word — treat them identically                           |
| Remove stop words                | Drops common words like `the`, `is`, `and` | These appear in every email — they carry no signal for spam vs. ham                        |
| Stemming                         | Reduces words to their root form           | `"prizes"` → `"prize"`, `"winning"` → `"win"` — fewer unique tokens, better generalization |

### The Cleaning Pipeline

```python
from nltk.corpus import stopwords
import re
import string
from nltk.stem.porter import PorterStemmer

ps = PorterStemmer()

cleaned_emails = []
for i in range(len(data)):
    cleaned_email = re.sub('[^a-zA-Z]', ' ', data['text'][i])  # Remove non-alpha
    cleaned_email = cleaned_email.lower()                        # Lowercase
    cleaned_email = [
        ps.stem(word)
        for word in cleaned_email.split()
        if word not in stopwords.words('english')               # Remove stop words + stem
    ]
    cleaned_email = ' '.join(cleaned_email)
    cleaned_emails.append(cleaned_email)
```

After this loop, `cleaned_emails` is a list of 5,728 cleaned strings — one per email. Each is a normalized, stemmed, stop-word-free version of the original.

> **Note:** You will notice later that we duplicate this exact cleaning logic when processing new emails. This works, but it is not ideal — ideally, this would be refactored into a reusable function. It is left as-is here to keep things explicit and easy to follow.

---

## Step 3 — Vectorization

### From Words to Numbers

Machine learning models are mathematical functions — they operate on numbers. We need to convert our cleaned text into a numerical representation. We use **CountVectorizer** from scikit-learn for this.

CountVectorizer works in two steps:

1. **Fit** — it scans all the emails and builds a vocabulary: a list of every unique word that appears across the entire dataset
2. **Transform** — for each email, it creates a vector of counts: how many times does each vocabulary word appear in this email?

The result is a matrix where each row is an email and each column is a word from the vocabulary. If the word `"free"` appears twice in email #42, then row 42, column `"free"` has the value 2.

```python
from sklearn.feature_extraction.text import CountVectorizer

cv = CountVectorizer()

X = cv.fit_transform(cleaned_emails).toarray()
Y = data.iloc[:, -1].values
```

`X` is now a matrix of shape `(5728, n_features)` where `n_features` is the size of the vocabulary. `Y` is the array of labels.

You can inspect the learned vocabulary:

```python
print(cv.get_feature_names_out())
```

![image](/images/blog/2026-06-19-building-classifier-model-to-attack/screen4.png)

> **Important:** We use `fit_transform` here — this both learns the vocabulary AND applies it. When we later want to classify new emails, we must use `cv.transform()` only (not `fit_transform`), so that we use the exact same vocabulary the model was trained on. A different vocabulary would produce meaningless results.

---

## Step 4 — Training the Model

### Why Naive Bayes for Text?

We use a **Multinomial Naive Bayes** classifier. It is the classic choice for text classification, and for good reasons:

- It is fast — training takes milliseconds even on large corpora
- It is interpretable — you can understand exactly why it makes a decision
- It performs surprisingly well on text data, even with a simple bag-of-words representation
- It handles the class imbalance gracefully without any special tricks

The "naive" part refers to the simplifying assumption that all words in an email are independent of each other. This is obviously false in reality (the word "free" is much more likely to appear next to "prize" in spam), but it turns out this assumption rarely hurts performance in practice for classification tasks.

Under the hood, the model learns: given a spam email, what is the probability that the word `"free"` appears? It does this for every word in the vocabulary, for both classes. At prediction time, it combines those word probabilities to decide: is this a spam email or a ham email?

### Splitting the Data

Before training, we split our data into a training set (80%) and a test set (20%). The test set is data the model will never see during training — it is our honest estimate of real-world performance.

```python
from sklearn.model_selection import train_test_split

X_train, X_test, Y_train, Y_test = train_test_split(
    X, Y,
    test_size=0.2,
    random_state=0,
    stratify=Y      # Preserves the 76/24 spam ratio in both splits
)
```

The `stratify=Y` argument is important here. Without it, random splitting might accidentally concentrate most spam examples in the training set (or the test set), making evaluation unreliable.

### Training

```python
from sklearn.naive_bayes import MultinomialNB

NB_classifier = MultinomialNB()
NB_classifier.fit(X_train, Y_train)
```

That is it. The model is trained.

### Saving the Model

We save both the model and the vectorizer using `pickle`. This is critical: you need **both** to classify new emails later. The model without the vectorizer is useless — it expects input in the exact vocabulary format the vectorizer learned.

```python
import pickle

pickle.dump(cv, open("vectorizer.pkl", "wb"))
pickle.dump(NB_classifier, open("model.pkl", "wb"))
```

To reload them later:

```python
cv = pickle.load(open("vectorizer.pkl", "rb"))
NB_classifier = pickle.load(open("model.pkl", "rb"))
```

---

## Step 5 — Evaluating the Model

We evaluate performance on the test set only — data the model has never seen.

### Reading the Confusion Matrix

```python
from sklearn.metrics import classification_report, confusion_matrix
import seaborn as sns

y_pred_test = NB_classifier.predict(X_test)

cm = confusion_matrix(Y_test, y_pred_test)
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues',
            xticklabels=['Predicted Ham', 'Predicted Spam'],
            yticklabels=['Actual Ham', 'Actual Spam'])
plt.title('Confusion Matrix — Test Set')
plt.tight_layout()
plt.show()
```

![image](/images/blog/2026-06-19-building-classifier-model-to-attack/screen5.png)

The confusion matrix is a 2×2 grid with four cells:

|                 | Predicted Ham      | Predicted Spam     |
| --------------- | ------------------ | ------------------ |
| **Actual Ham**  | True Negatives ✅  | False Positives ⚠️ |
| **Actual Spam** | False Negatives ❌ | True Positives ✅  |

For a spam filter:

- **False Positives** (ham wrongly flagged as spam) are annoying — the user misses a legitimate email
- **False Negatives** (spam that slipped through) are the primary failure we care about

### Reading the Classification Report

```python
print(classification_report(Y_test, y_pred_test))
```

![image](/images/blog/2026-06-19-building-classifier-model-to-attack/screen6.png)

The three key metrics for each class:

- **Precision** — of all emails predicted as spam, what fraction actually was spam? High precision means fewer false alarms.
- **Recall** — of all actual spam emails, what fraction did we catch? High recall means fewer missed spam.
- **F1-score** — the harmonic mean of precision and recall. A good single number to compare models.

For spam detection, **recall on the spam class** is the number to watch. A spam filter with 98% precision but 60% recall lets 40% of spam through — that is not a great filter.

Our Naive Bayes model should perform well on both metrics, landing above 95% accuracy with solid recall on the spam class.

---

## Step 6 — Testing on New Emails

Let's validate the end-to-end pipeline by running the model on two hand-crafted emails:

```python
new_emails = [
    "Subject: Congratulations! You won a FREE prize!!! Congratulations you have won a free prize click now to claim your reward",
    "Subject: Meeting tomorrow. Reminder we have a meeting tomorrow at 10am please review the document"
]
```

We apply the exact same cleaning pipeline we used during training:

```python
new_cleaned_emails = []
for i in range(len(new_emails)):
    cleaned_email = re.sub('[^a-zA-Z]', ' ', new_emails[i])
    cleaned_email = cleaned_email.lower()
    cleaned_email = [
        ps.stem(word)
        for word in cleaned_email.split()
        if word not in stopwords.words('english')
    ]
    cleaned_email = ' '.join(cleaned_email)
    new_cleaned_emails.append(cleaned_email)
```

Then vectorize using `transform` (not `fit_transform`):

```python
X_new = cv.transform(new_cleaned_emails).toarray()
```

And predict:

```python
predictions_new = NB_classifier.predict(X_new)
for email, pred in zip(new_emails, predictions_new):
    label = "SPAM" if pred == 1 else "HAM"
    print(f"{label}: {email}")
```

![image](/images/blog/2026-06-19-building-classifier-model-to-attack/screen7.png)

The model correctly identifies the first email as spam and the second as legitimate.

---

## Key Takeaways

**1. The full pipeline is: raw text → clean → vectorize → train → evaluate.**
Each step has a clear purpose. Skipping or misapplying any of them breaks the model.

**2. Save both the vectorizer and the model.**
The vectorizer encodes the vocabulary learned from your training data. Any new email must be transformed using that same vocabulary. A fresh `fit_transform` on new data would produce an incompatible feature space.

**3. Accuracy is not enough — use precision, recall, and F1.**
With a 76/24 class imbalance, a lazy model could reach 76% accuracy by predicting "ham" for everything. The classification report tells the full story.

**4. Naive Bayes is a strong baseline for text.**
It is not the most powerful classifier available, but it is fast, interpretable, and performs well for this task. That makes it a good starting point — and a good attack target, since its decision-making is easy to reason about.

**5. This model is intentionally simple.**
We have not tuned hyperparameters, explored alternative algorithms, or implemented advanced NLP. The simplicity is deliberate: what we care about is the attack surface, not squeezing out an extra 0.5% F1.

---

## What's Next

Now that we have a working spam classifier, we can start breaking it.

Upcoming articles will use this exact model to demonstrate:

- **Data poisoning attacks** — what happens if an attacker can inject malicious examples into our training dataset before training?
- **Adversarial inputs** — can we craft spam emails that fool the model at inference time, without touching the training data?
- **Model evasion** — how do spammers adapt their text to slip past classifiers like this one?

Each attack will be implemented end-to-end, with the same notebook-first approach used here. Stay tuned.

---

_Learning out loud. Breaking things responsibly._
