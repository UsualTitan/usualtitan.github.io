---
layout: default
title: "HTB - AI Red Teamer - Data Attacks"
date: 2026-05-17
description: "Complete write up about LLM Output attacks module of HTB Red team AI including skill assessment"
tags: [hackthebox, ctf, writeup, linux, label-flipping, ai-data-attacks]
permalink: /writeups/htb-ai-red-teamer-ai-data-attacks/
---

## Introduction

This write-up covers the Skills Assessment from HackTheBox's AI Security module on data poisoning attacks. The goal was to implement a **label flipping attack** against a multi-class logistic regression classifier, causing it to systematically misclassify samples from a specific class — without the model appearing completely broken.

This is a classic example of a **training-time attack**: instead of tampering with the model itself, we corrupt the data it learns from. The result is a model that looks functional but has a hidden weakness we deliberately introduced.

---

## Understanding the Challenge

### The Setup

We are given:

- A dataset (`assessment_dataset.npz`) with 4 balanced classes (1500 training samples, 500 test samples, 2 features)
- A notebook template with most of the code pre-written
- A single cell to implement the attack (Cell 4)
- An evaluation API that checks whether the attack was successful
  `[SCREENSHOT — Dataset structure output showing shapes and class distribution]`

The model is a **One-vs-Rest (OvR) Logistic Regression** classifier, trained with fixed hyperparameters we cannot change:

```python
base_estimator_config = {
    "random_state": SEED,
    "solver": "liblinear",
    "C": 1.0,
    "max_iter": 200,
}
baseline_model = OneVsRestClassifier(LogisticRegression(**base_estimator_config))
```

### The Objective

The API evaluates three specific criteria for a successful attack:

| Criterion                    | What it means                                                    |
| ---------------------------- | ---------------------------------------------------------------- |
| `class1_to_class0_ambiguity` | Enough Class 1 samples must be predicted as Class 0              |
| `class1_to_class2_ambiguity` | Enough Class 1 samples must be predicted as Class 2              |
| `class3_recall_maintained`   | Class 3 must still be detected correctly (model stays plausible) |

This is subtle — the attack must **simultaneously** create confusion in two directions for Class 1, while keeping the rest of the model functional.

`[SCREENSHOT — Baseline model confusion matrix showing clean performance]`

---

## What is a Label Flipping Attack?

Before diving into the solution, it's worth understanding what we're actually doing.

In a standard machine learning pipeline, the training dataset is assumed to be trustworthy. The model learns decision boundaries based on the feature-label pairs it sees. If an attacker can **modify the labels** of some training samples before training happens, they can steer the model's learned boundaries wherever they want.

```
Normal training:
  Sample (features=X, label=1)  →  model learns "this region = Class 1"

After label flipping:
  Sample (features=X, label=0)  →  model learns "this region = Class 0"
```

The features (the actual data points) don't change — only their assigned labels do. This makes the attack hard to detect by just looking at the data visually.

In real-world scenarios, this could happen through:

- A compromised data collection pipeline
- A malicious insider modifying training datasets
- Poisoned data scraped from untrusted sources

---

## Attempt 1 — Flipping Everything (Failed)

My first instinct was to flip 100% of Class 1 labels to Class 0. Maximum confusion, right?

```python
class1_indices = np.where(y_train_orig == 1)[0]
y_train_poisoned[class1_indices] = 0  # Flip everything
```

The API immediately rejected this:

```json
{
  "attack_successful": false,
  "message": "Model was for 3 classes, expected 4."
}
```

**Why it failed:** By flipping every single Class 1 sample, I removed Class 1 entirely from the training set. The OvR classifier trains one binary classifier per class — if Class 1 never appears, it never trains a Class 1 estimator. The resulting model only knows 3 classes, and the API expects 4.

**Lesson:** The attack must be subtle enough that the model still _appears_ complete and functional.

---

## Attempt 2 — Flipping 90% to Class 0 (Partially Successful)

The fix was to keep a small fraction of Class 1 samples untouched, so the model still learns a Class 1 boundary:

```python
rng = np.random.default_rng(SEED)
n_to_flip = int(len(class1_indices) * 0.90)
indices_to_flip = rng.choice(class1_indices, size=n_to_flip, replace=False)
y_train_poisoned[indices_to_flip] = 0
```

The API response this time was much more informative:

```json
{
  "attack_successful": false,
  "criteria_met": {
    "class1_to_class0_ambiguity": true,
    "class1_to_class2_ambiguity": false,
    "class3_recall_maintained": true
  }
}
```

Two out of three criteria met. The model was now misclassifying Class 1 as Class 0 successfully, and Class 3 was still intact. But the API also required Class 1 to be misclassified as Class 2 — which wasn't happening because all our flips went in one direction only.

**Lesson:** Read the evaluation criteria carefully. "Ambiguity" means confusion in multiple directions, not just one.

---

## Attempt 3 — Splitting Flips Between Two Classes (Success)

The key insight is that we need to **split the poison** between two target classes. By flipping some Class 1 samples to Class 0 and others to Class 2, the decision boundary for Class 1 gets pulled in both directions simultaneously:

```python
rng = np.random.default_rng(SEED)
shuffled = rng.permutation(class1_indices)

n_total = len(class1_indices)
n_keep = int(n_total * 0.05)            # 5% kept as Class 1 (model stays 4-class)
n_flip_to_0 = int(n_total * 0.50)      # 50% flipped to Class 0
n_flip_to_2 = n_total - n_keep - n_flip_to_0  # 45% flipped to Class 2

indices_to_0 = shuffled[:n_flip_to_0]
indices_to_2 = shuffled[n_flip_to_0:n_flip_to_0 + n_flip_to_2]

y_train_poisoned[indices_to_0] = 0
y_train_poisoned[indices_to_2] = 2
```

**What this does geometrically:** The OvR classifier learns a boundary for each class. By poisoning the Class 1 region with Class 0 labels on one side and Class 2 labels on the other, we squeeze the learned Class 1 boundary until it barely exists. New Class 1 samples at inference time fall into either the Class 0 or Class 2 region.

![image](/images/posts/2026-05-17-data-attacks/data-visualization-poisoned-model.png)

![image](/images/posts/2026-05-17-data-attacks/confusion-mattrix-poisoned-model.png)

The final API response:

```json
{
  "attack_successful": true,
  "criteria_met": {
    "class1_to_class0_ambiguity": true,
    "class1_to_class2_ambiguity": true,
    "class3_recall_maintained": true
  },
  "flag": "HTB{...}"
}
```

![image](/images/posts/2026-05-17-data-attacks/final-response-with-flag.png)

---

## Full Attack Cell

Here is the complete Cell 4 that solves the challenge:

```python
# Implement your attack strategy here.
y_train_poisoned = None  # Initialize

if y_train_orig is not None:
    y_train_poisoned = y_train_orig.copy()

    # Identify all Class 1 indices
    class1_indices = np.where(y_train_orig == 1)[0]

    # Shuffle them randomly for an unbiased split
    rng = np.random.default_rng(SEED)
    shuffled = rng.permutation(class1_indices)

    n_total = len(class1_indices)
    n_keep = int(n_total * 0.05)           # Keep 5% as Class 1 — model stays 4-class
    n_flip_to_0 = int(n_total * 0.50)     # Flip 50% to Class 0
    n_flip_to_2 = n_total - n_keep - n_flip_to_0  # Flip 45% to Class 2

    indices_to_0 = shuffled[:n_flip_to_0]
    indices_to_2 = shuffled[n_flip_to_0:n_flip_to_0 + n_flip_to_2]

    y_train_poisoned[indices_to_0] = 0
    y_train_poisoned[indices_to_2] = 2

    print(f"Flipped {len(indices_to_0)} labels Class 1 → Class 0")
    print(f"Flipped {len(indices_to_2)} labels Class 1 → Class 2")
    print(f"Kept    {n_keep} labels as Class 1")

    print(f"\nOriginal distribution:  {np.bincount(y_train_orig)}")
    print(f"Poisoned distribution:  {np.bincount(y_train_poisoned)}")

    plot_dataset_points(
        X_train_orig,
        y_train_poisoned,
        title="Poisoned Training Data Label Distribution",
    )
else:
    print("Skipping attack implementation as original training data was not loaded.")
```

---

## Key Takeaways

**1. Label flipping is a training-time attack — no model access needed.**
The attacker only needs write access to the training data. Once the poisoned dataset is used for training, the corrupted behavior is baked into the model's weights permanently.

**2. Subtlety is essential.**
A naive 100% flip immediately revealed itself because the model lost a class entirely. Real attacks need to stay below the detection threshold — keeping accuracy plausible while the targeted vulnerability is hidden.

**3. Multi-directional confusion is harder to detect.**
Flipping all Class 1 to Class 0 would show up as a suspicious drop in Class 1 recall. Splitting flips between two classes creates ambiguity that looks more like a "hard classification problem" than deliberate poisoning.

**4. The defense is data validation.**
Techniques like data sanitization, label consistency checks, and influence function analysis can help detect poisoned samples before training. In practice, organizations should treat their training datasets with the same scrutiny as their code.

---
