---
layout: default
title: "HTB - AI Red Teamer - AI Evasion Foundations"
date: 2026-05-27
description: "Complete write-up about the AI Evasion Foundations Skills Assessment from HTB AI Red Teamer path — GoodWords attacks against Naive Bayes classifiers in white-box and black-box settings"
tags:
  [
    hackthebox,
    ctf,
    writeup,
    naive-bayes,
    adversarial-ml,
    evasion-attack,
    goodwords,
  ]
permalink: /writeups/htb-ai-red-teamer-ai-evasion-foundations/
---

## Introduction

This write-up covers the Skills Assessment from HackTheBox's **AI Evasion — Foundations** module. The goal was to execute **GoodWords attacks** against two Multinomial Naive Bayes classifiers used for movie review sentiment analysis.

A GoodWords attack is a type of **inference-time evasion**: instead of tampering with the model during training, we manipulate the input at prediction time. The constraint is append-only — we can only add words to the end of a review, never modify or remove the original text. The result is input that fools the classifier while the original content remains intact.

The challenge is split into two phases:

- **Phase 1 (White-Box)**: Full model access — flip 10 positive reviews to negative, up to 30 added words per review.
- **Phase 2 (Black-Box)**: API only — flip 10 negative reviews to positive, up to 40 added words per review.

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding the Challenge](#understanding-the-challenge)
  - [The Setup](#the-setup)
  - [The Objective](#the-objective)
- [What is a GoodWords Attack?](#what-is-a-goodwords-attack)
- [Phase 1 — White-Box Attack](#phase-1--white-box-attack)
  - [Step 1 — Fetching the Challenge and the Model](#step-1--fetching-the-challenge-and-the-model)
  - [Step 2 — Exploiting the Model Weights](#step-2--exploiting-the-model-weights)
  - [Step 3 — Building and Submitting Solutions](#step-3--building-and-submitting-solutions)
- [Phase 2 — Black-Box Attack](#phase-2--black-box-attack)
  - [Step 1 — Fetching the Challenge](#step-1--fetching-the-challenge)
  - [Step 2 — Measuring Word Impact](#step-2--measuring-word-impact)
  - [Step 3 — The Stubborn Reviews](#step-3--the-stubborn-reviews-bb_1-and-bb_6)
  - [Step 4 — Final Complete Solution](#step-4--final-complete-solution)
- [Key Takeaways](#key-takeaways)

---

## Understanding the Challenge

### The Setup

We are given:

- A running sentiment analysis service exposing a `/predict` endpoint
- Two separate challenge endpoints: `/challenge/whitebox` and `/challenge/blackbox`
- A `/model/download` endpoint available only for the white-box phase
- An evaluation API at `/submit/whitebox` and `/submit/blackbox` that checks whether the flips succeeded

The target model is a **Multinomial Naive Bayes** classifier trained on movie reviews and predicting one of two labels: `positive` or `negative`.

{% include image.html src="/images/writeups/2026-05-27-AI-evasion-foundation/screen1.png" width="500px" %}

### The Objective

| Phase     | Direction           | Budget                  | Access              |
| --------- | ------------------- | ----------------------- | ------------------- |
| White-Box | Positive → Negative | 30 words max per review | Full model download |
| Black-Box | Negative → Positive | 40 words max per review | API queries only    |

Both phases share the same constraint: the original review text must not be modified. Words can only be appended.

---

## What is a GoodWords Attack?

Before diving into the solution, it's worth understanding what makes this attack possible.

Multinomial Naive Bayes computes the probability of a class as:
P(class | text) ∝ P(class) × ∏ P(word | class)

In practice, log-probabilities are used to avoid numerical underflow:
log P(class | text) = log P(class) + Σ log P(word | class)

**The key consequence**: every word added to a text contributes additively to the class score. If you append enough words that are strongly associated with a target class, you can shift the prediction — even if the original content points in the opposite direction.

This is called a **GoodWords attack**: inject words that are "good" for the class you want to steer toward. The original text is never touched, which makes the manipulation harder to catch with simple diff-based detection.

In real-world scenarios, this could be used to:

- Bypass spam or toxicity filters by padding malicious content with innocuous-looking words
- Manipulate content moderation systems at inference time
- Evade any bag-of-words or frequency-based classifier without touching the core payload

---

## Phase 1 — White-Box Attack

### Step 1 — Fetching the Challenge and the Model

The first step is to retrieve the 10 positive reviews to flip and download the full model bundle.

```python
import os, requests, pickle
import numpy as np

BASE_URL = os.getenv("BASE_URL", "http://localhost:8080")

# Verify the instance is running
r = requests.get(f"{BASE_URL}/health")
print(r.json())  # {"status": "healthy", "service": "skills_assessment_lab"}

def predict(text):
    """Query the black-box model"""
    return requests.post(
        f"{BASE_URL}/predict",
        json={"text": text}
    ).json()
    # Returns: {"label": "positive/negative",
    #           "negative_probability": 0.xx,
    #           "positive_probability": 0.xx}

# Fetch the reviews to flip
challenge = requests.get(f"{BASE_URL}/challenge/whitebox").json()
reviews = challenge["reviews"]          # 10 positive reviews to make negative
budget  = challenge["max_added_words"]  # 30 words max per review
print(f"Budget: {budget} words")

# Download the full model
r = requests.get(f"{BASE_URL}/model/download")
with open("model.pkl", "wb") as f:
    f.write(r.content)

# Load the bundle
with open("model.pkl", "rb") as f:
    bundle = pickle.load(f)

clf        = bundle["classifier"]    # The Naive Bayes model
vectorizer = bundle["vectorizer"]    # The TF vectorizer (vocabulary)
features   = bundle["feature_names"] # List of all known words
classes    = list(bundle["classes"]) # ["negative", "positive"]
```

{% include image.html src="/images/writeups/2026-05-27-AI-evasion-foundation/screen2.png" width="700px" %}

### Step 2 — Exploiting the Model Weights

With full model access, we can read `feature_log_prob_` directly — the matrix that stores the log-score of each word for each class. No need to test words one by one; we can rank all of them instantly.

```python
# Class indices
neg_idx = classes.index("negative")
pos_idx = classes.index("positive")

# feature_log_prob_ shape: (n_classes, n_features)
# For each word, compute the delta: how much more "negative" than "positive" it is
delta = (clf.feature_log_prob_[neg_idx]
       - clf.feature_log_prob_[pos_idx])

# Sort from most "negative" to least
ranked = np.argsort(delta)[::-1]
top_negative_words = [features[i] for i in ranked[:60]]
print("Top negative words:", top_negative_words[:20])
# e.g.: ["terrible", "awful", "worst", "boring", "waste", ...]
```

**Why this works:** We're reading the actual learned weights, not estimating them. The delta `log P(word | negative) - log P(word | positive)` tells us exactly how much each word shifts the balance toward the negative class. The higher the delta, the more useful the word is for our attack.

{% include image.html src="/images/writeups/2026-05-27-AI-evasion-foundation/screen3.png" width="700px" %}

### Step 3 — Building and Submitting Solutions

With the ranked word list in hand, appending the top words up to the budget is all it takes.

```python
solutions = []

for review in reviews:
    rid  = review["id"]
    text = review["text"]
    aug  = text

    # Append the most negative words up to the budget
    words_added = 0
    for word in top_negative_words:
        if words_added >= budget:
            break
        aug += " " + word
        words_added += 1

    solutions.append({"id": rid, "augmented_text": aug})
    print(f"{rid}: +{words_added} words added")

# Submit
result = requests.post(
    f"{BASE_URL}/submit/whitebox",
    json={"solutions": solutions}
).json()
print(result)
```

{% include image.html src="/images/writeups/2026-05-27-AI-evasion-foundation/screen4.png" width="700px" %}

---

## Phase 2 — Black-Box Attack

In this phase we have no model access — only the `/predict` endpoint and the probabilities it returns. The strategy shifts from reading weights directly to **estimating them empirically**.

### Step 1 — Fetching the Challenge

```python
challenge = requests.get(f"{BASE_URL}/challenge/blackbox").json()
reviews   = challenge["reviews"]        # 10 negative reviews to make positive
budget    = challenge["max_added_words"] # 40 words max

reviews_dict = {r["id"]: r["text"] for r in reviews}
```

### Step 2 — Measuring Word Impact

Since we can't read the model weights, we probe them: for each candidate word, we measure how much it reduces `negative_probability` when appended. This gives us an empirical estimate of each word's contribution.

```python
positive_vocab = [
    "excellent", "wonderful", "brilliant", "masterpiece", "outstanding",
    "beautiful", "perfect", "amazing", "fantastic", "superb",
    "love", "loved", "enjoyed", "great", "best", "favorite",
    "moving", "touching", "inspiring", "heartwarming", "delightful",
    "recommend", "recommended", "incredible", "remarkable", "stunning",
    "compelling", "engaging", "captivating", "charming", "memorable",
    "magnificent", "exceptional", "splendid", "terrific", "marvelous",
    "hilarious", "entertaining", "exciting", "thrilling", "powerful",
    "classic", "timeless", "genius", "revolutionary", "unforgettable",
    "joy", "pleasure", "treasure", "celebrate", "triumph",
    "happy", "smile", "laugh", "warm", "sweet",
]

solutions = []

for review in reviews:
    rid  = review["id"]
    text = review["text"]

    base_p = predict(text)["negative_probability"]
    print(f"\n{rid} — base neg_prob: {base_p:.4f}")

    # Measure the impact of each candidate word
    impacts = []
    for w in positive_vocab:
        p2 = predict(text + " " + w)["negative_probability"]
        delta = base_p - p2  # positive = reduces negative score
        impacts.append((w, delta))

    impacts.sort(key=lambda x: x[1], reverse=True)
    print("Top 5:", [(w, f"{d:.4f}") for w, d in impacts[:5]])

    # Greedy construction: add the best words one by one
    aug = text
    for w, _ in impacts:
        if len(aug.split()) - len(text.split()) >= budget:
            break
        aug += " " + w
        if predict(aug)["label"] == "positive":
            print(f"  Flipped!")
            break

    solutions.append({"id": rid, "augmented_text": aug})
```

{% include image.html src="/images/writeups/2026-05-27-AI-evasion-foundation/screen5.png" width="600px" %}

### Step 3 — The Stubborn Reviews (bb_1 and bb_6)

Some reviews returned `negative_probability = 1.0000` and no candidate word produced any measurable delta — every word scored `0.0000`.

This looked like the vocabulary probe was useless, but the actual cause was more subtle: **the API was rounding to 1.0**, but the effect of each word was real and just too small to show up at 4 decimal places. The Naive Bayes model was still accumulating the log-probabilities — the shift simply wasn't large enough to cross the display threshold with a single word.

The diagnostic was straightforward: repeat a single strong word 40 times and observe whether the label eventually flips.

```python
for rid in ["bb_1", "bb_6"]:
    text = reviews_dict[rid]
    print(f"\n=== {rid} ===")
    print(f"Text: '{text[:150]}'")

    for word in ["perfect", "brilliant", "outstanding"]:
        combo = (word + " ") * 40
        aug = text + " " + combo.strip()
        r = predict(aug)
        print(f"  '{word}' x40: neg={r['negative_probability']:.10f} label={r['label']}")
```

{% include image.html src="/images/writeups/2026-05-27-AI-evasion-foundation/screen6.png" width="600px" %}

**Key lesson:** The API was rounding to `1.0` but the effect existed. Because Naive Bayes sums log-probabilities, repeating a word 40 times multiplies its influence by 40 — enough to push even the most stubborn review over the classification threshold.

### Step 4 — Final Complete Solution

```python
solutions = []

for review in reviews:
    rid  = review["id"]
    text = review["text"]

    # Special handling for stubborn reviews
    if rid in ["bb_1", "bb_6"]:
        words = ("perfect " * budget).strip().split()[:budget]
        aug = text + " " + " ".join(words)
        r = predict(aug)
        print(f"{rid}: neg={r['negative_probability']:.6f} label={r['label']}")
        solutions.append({"id": rid, "augmented_text": aug})
        continue

    # Standard case: greedy approach
    base_p = predict(text)["negative_probability"]
    impacts = []
    for w in positive_vocab:
        p2 = predict(text + " " + w)["negative_probability"]
        impacts.append((w, base_p - p2))
    impacts.sort(key=lambda x: x[1], reverse=True)

    aug = text
    for w, _ in impacts:
        if len(aug.split()) - len(text.split()) >= budget:
            break
        aug += " " + w
        if predict(aug)["label"] == "positive":
            break

    solutions.append({"id": rid, "augmented_text": aug})
    print(f"{rid}: flipped")

# Submit everything in one call
result = requests.post(
    f"{BASE_URL}/submit/blackbox",
    json={"solutions": solutions}
).json()
print(result)
# {"flag": "HTB{...}", "completed": 10, "required": 10}
```

{% include image.html src="/images/writeups/2026-05-27-AI-evasion-foundation/screen7.png" width="500px" %}

---

## Key Takeaways

**1. GoodWords attacks exploit the additive nature of bag-of-words models.**
Naive Bayes sums log-probabilities across all tokens. Every appended word shifts the score — there is no context awareness that could "cancel out" the injected words against the original content.

**2. White-box access makes the attack trivial.**
Reading `feature_log_prob_` directly gives a perfect ranking of all words with zero API calls. The entire attack collapses to sorting an array and appending the top entries.

**3. Black-box probing is a reliable fallback.**
Without model access, measuring the delta in `negative_probability` for each candidate word empirically reconstructs a usable word ranking. It costs more queries but converges quickly.

**4. API rounding can hide real effects.**
A probability of `1.0000` doesn't mean the input is unshiftable — it may just mean the shift is smaller than the display precision. Always verify with high-repetition tests before concluding a word has zero impact.

**5. The defense is feature-aware input validation.**
Simple mitigations include: limiting the total token count of inputs, flagging inputs where appended content is semantically unrelated to the original, or using models that account for token position and context (e.g. transformers) rather than treating all tokens as an unordered bag.
