---
layout: default
title: "What is AI security"
date: 2026-05-15
description: "Description of what AI security is"
tags: [ai, cybersecurity]
permalink: /blog/what-is-ai-security/
cover: /images/blog/2026-05-22-what-is-ai-security/cover.png
---

## Introduction

If you've been in offensive security for a while, you've probably noticed the shift happening around you. AI is showing up everywhere — in products, in infrastructure, in security tooling itself. And with it comes a question that didn't really exist five years ago: _what does it mean to hack an AI system?_

This article is a ground-level introduction to AI cybersecurity for people coming from a traditional security background. Not the research paper version. Not the vendor marketing version. The practitioner version — what the attack surface actually looks like, why it's different from what you already know, and why it matters.

---

## Table of Contents

- [Introduction](#introduction)
- [What We Mean by "AI Cybersecurity"](#what-we-mean-by-ai-cybersecurity)
- [The AI Attack Surface](#the-ai-attack-surface)
  - [Attacking the Data](#attacking-the-data)
  - [Attacking the Model](#attacking-the-model)
  - [Attacking the Inputs](#attacking-the-inputs)
  - [Attacking the Outputs](#attacking-the-outputs)
  - [Attacking the Infrastructure](#attacking-the-infrastructure)
- [How This Differs from Traditional Security](#how-this-differs-from-traditional-security)
- [Offensive vs Defensive AI Security](#offensive-vs-defensive-ai-security)
- [Where to Start](#where-to-start)
- [Key Takeaways](#key-takeaways)

---

## What We Mean by "AI Cybersecurity"

"AI cybersecurity" is a term used in two very different ways, and conflating them will confuse you early on.

**The first meaning** is using AI as a tool in cybersecurity — think AI-assisted vulnerability discovery, LLM-powered phishing, or ML-based anomaly detection in a SIEM. This is interesting but it's mostly AI applied _to_ security work.

**The second meaning** — and the one this blog focuses on — is treating AI systems themselves as targets. The question isn't "how does AI help attackers?" but "how do you attack an AI system, and how do you defend one?"

This second framing is what's known as **adversarial machine learning** or **AI red teaming**. It's a relatively young field, the tooling is still maturing, and the knowledge base is spread across academic papers, CTF writeups, and vendor advisories in a way that's hard to navigate for practitioners.

That's what we're here to figure out together.

---

## The AI Attack Surface

The attack surface of an AI system doesn't look like a web app or a network. There's no login form, no SQL query, no service listening on a port. Instead, the vulnerabilities live in the _pipeline_ — the sequence of steps that takes raw data and turns it into a deployed model making real decisions.

Here's a map of where attackers can intervene:

| Stage             | Target            | Attack Category                                    |
| ----------------- | ----------------- | -------------------------------------------------- |
| Training Data     | Dataset integrity | Data poisoning, label flipping, backdoor injection |
| Training Process  | ML pipeline       | Supply chain attacks, dependency tampering         |
| Model Weights     | Trained model     | Model theft (extraction), model inversion          |
| Inference API     | Input handling    | Adversarial examples, jailbreaks, prompt injection |
| Output / Decision | Downstream logic  | Output manipulation, hallucination exploitation    |

### Attacking the Data

Training data is the foundation everything else is built on. If you can tamper with it, you can tamper with everything the model learns — permanently, and in ways that are hard to detect after the fact.

**Data poisoning** is the broad category here. The attacker's goal is to inject malicious samples into the training set, steering the model's behavior in a direction that benefits them. This can look like:

- **Label flipping** — changing the ground-truth labels of training samples so the model learns wrong associations (we covered this in depth in the [Data Attacks writeup](/writeups/htb-ai-red-teamer-ai-data-attacks/))
- **Backdoor attacks** — embedding a hidden trigger into training data so the model behaves normally on clean inputs but misbehaves whenever it sees a specific pattern
- **Model skewing** — poisoning enough samples to shift the model's overall behavior on a class or distribution without introducing an obvious trigger

The scary part: once a poisoned model is deployed, the corruption is baked into the weights. You can't just patch it. You have to retrain.

### Attacking the Model

Sometimes the attack isn't about corrupting training — it's about the model itself.

**Model theft (extraction)** is the process of reconstructing a model's behavior by querying it repeatedly and using the outputs to train a surrogate. You never see the weights directly. You just ask enough questions until you've mapped the decision boundary well enough to replicate it. This matters for IP theft, competitive intelligence, or as a stepping stone to craft better adversarial inputs.

**Model inversion** goes the other direction — instead of reconstructing the model, you try to reconstruct the training data from the model's outputs. If a model was trained on sensitive medical records, a model inversion attack might recover statistical information about those records just from its predictions.

### Attacking the Inputs

This is the category most people encounter first, especially with LLMs.

**Adversarial examples** are inputs that are specifically crafted to cause a model to make wrong predictions. The classic demonstration is an image of a panda that, after tiny imperceptible pixel changes, gets confidently classified as a gibbon. The human eye can't see the difference. The model is completely fooled.

In the LLM world, the equivalent is **jailbreaking** — crafting prompts that bypass the model's safety training and get it to produce outputs it's supposed to refuse. And beyond jailbreaks, there are **prompt injection attacks**, where malicious content embedded in external data (a webpage, a document, a tool result) hijacks the model's behavior in an agentic context.

**Adversarial examples** work by exploiting the fact that machine learning models don't perceive the world the way humans do. They're pattern matchers operating in high-dimensional feature spaces, and those spaces have blind spots that are invisible to us but trivially exploitable once you know how to look.

### Attacking the Outputs

When a model's output is used to make a real decision — approve a loan, flag a transaction, generate a recommendation — that output becomes an attack vector in its own right.

Attackers can try to manipulate outputs through input crafting (see above), but they can also exploit how outputs are consumed downstream. An LLM that generates code can be prompted to produce subtly vulnerable code. A classification system that feeds into a business logic layer can be manipulated to produce specific misclassifications on demand.

This is increasingly relevant as AI is embedded into automated pipelines with minimal human review.

### Attacking the Infrastructure

Finally, AI systems run on infrastructure, and infrastructure has the same vulnerabilities it always has. ML models are often served through APIs, loaded from cloud storage, or pulled from model registries. All of these are attack surfaces:

- **Supply chain attacks** on model repositories or datasets
- **Insecure deserialization** in model file formats (pickle files are a particularly well-known example in the Python ML ecosystem)
- **API vulnerabilities** in model serving endpoints
- **Excessive permissions** on MLOps pipelines

This last category is the most familiar for practitioners coming from traditional pentesting. The tooling and techniques map directly. What changes is the target.

---

## How This Differs from Traditional Security

If you're coming from web or network pentesting, here's what will feel unfamiliar:

**There's no source of truth for "correct" behavior.** When you find a SQL injection, there's no ambiguity — that's a vulnerability. When you craft an adversarial example, you're exploiting a property of how the model generalizes. Whether that's a "bug" depends on the threat model, the deployment context, and what guarantees the system claims to provide.

**Attacks are probabilistic.** You don't get a shell or you don't. You shift a probability distribution. A successful data poisoning attack might cause a 40% misclassification rate on a target class — not 100%. Evaluation requires statistical reasoning, not just checking if a flag appeared.

**The "patch" is retraining.** There's no CVE, no hotfix, no configuration change. If a model is fundamentally compromised, the fix is going back to the data and the training process. This has real operational cost implications.

**The attack surface is the math.** Understanding _why_ an attack works requires at least a conceptual understanding of how the model learns. You can execute techniques without deep ML knowledge, but you'll hit a ceiling quickly if you treat models as black boxes with no interest in what's inside.

---

## Offensive vs Defensive AI Security

Like traditional security, AI security has two sides.

**Offensive AI security** — AI red teaming — is about finding vulnerabilities in AI systems before attackers do. This includes:

- Adversarial robustness testing
- Data pipeline security assessments
- Prompt injection and jailbreak testing on LLM-based products
- Model extraction and inversion testing
- Threat modeling for AI-integrated systems

**Defensive AI security** covers the countermeasures:

- Adversarial training (training models on adversarial examples to make them robust)
- Data sanitization and provenance tracking
- Input validation and output filtering for deployed models
- Model monitoring for distribution shift or anomalous query patterns
- Secure MLOps practices

As an offensive practitioner, the goal is to understand both sides well enough to tell defenders where their real exposures are — not just to run techniques, but to communicate the actual risk.

---

## Where to Start

The honest answer is: the resources are scattered, and the field is moving fast. Here's what's actually useful:

- **HackTheBox AI Red Teamer path** — hands-on, practical, covers the core attack categories with real challenges. This blog documents my journey through it.
- **Adversarial Robustness Toolbox (ART)** by IBM — open source Python library that implements most of the major attack and defense techniques. Good for experimentation.
- **OWASP Top 10 for LLMs** — if your focus is LLM-based applications specifically, this is the most practitioner-friendly framework currently available.
- **Papers with Code** — for when you want to understand the research behind a technique, with implementations attached.

The fastest way to learn this space is the same as every other security domain: find a challenge, get stuck, figure out why it works, write about it. That's the loop this blog runs on.

---

## Key Takeaways

**1. AI cybersecurity is about attacking AI systems, not just using AI for attacks.**
The field treats models, datasets, and ML pipelines as targets with their own attack surfaces and vulnerability classes.

**2. The attack surface spans the entire ML pipeline.**
Data, model, inputs, outputs, and infrastructure all have distinct threat categories. A complete security assessment covers all of them.

**3. This is different from traditional security in important ways.**
Probabilistic outcomes, no patch without retraining, and math-dependent attack logic require shifting some mental models — but the practitioner mindset transfers well.

**4. The field is young and the opportunity is real.**
Most organizations deploying AI have not assessed it from a security perspective. The practitioners who understand both offensive security and ML are rare, and that gap is widening as AI adoption accelerates.
