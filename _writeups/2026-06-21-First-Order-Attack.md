---
layout: default
title: "HTB - AI Red Teamer - First Order Attacks"
date: 2026-06-21
description: "Complete write up about the First Order Attacks module of HTB AI Red Teamer path, covering FGSM and DeepFool on MNIST and CIFAR-10"
tags:
  [
    hackthebox,
    ctf,
    writeup,
    adversarial-examples,
    fgsm,
    deepfool,
    pytorch,
    ai-security,
  ]
permalink: /writeups/htb-ai-red-teamer-first-order-attacks/
---

## Introduction

This write-up covers the **First Order Attacks** module from HackTheBox's AI Red Teamer path. The module contains two guided challenges and two skills assessments, all revolving around **adversarial examples** — inputs deliberately crafted to fool a neural network classifier.

Unlike data poisoning (which corrupts the training phase), adversarial attacks happen **at inference time**. The model is already trained and deployed. We manipulate the input pixels in a way that's nearly invisible to a human observer but causes the model to predict the wrong class.

The four exercises progress from simple to complex:

| #   | Dataset               | Attack               | Distance      | Targeted    | Threshold |
| --- | --------------------- | -------------------- | ------------- | ----------- | --------- |
| C1  | MNIST 28×28 grayscale | FGSM (1 step)        | L∞            | No          | ε = 0.25  |
| C2  | MNIST 28×28 grayscale | DeepFool (iterative) | L2            | Yes (→ 6)   | 0.75      |
| SA1 | CIFAR-10 32×32 RGB    | I-FGSM (iterative)   | L∞            | Yes (→ cat) | 8/255     |
| SA2 | CIFAR-10 32×32 RGB    | DeepFool (iterative) | L2 normalized | No          | 3.5       |

All work is done in Jupyter Notebooks with PyTorch and the `htb_ai_library` helper provided by the platform.

---

## Table of Contents

- [Introduction](#introduction)
- [Background — What Are First Order Attacks?](#background--what-are-first-order-attacks)
  - [The Gradient Intuition](#the-gradient-intuition)
  - [L∞ vs L2 — Why the Distance Metric Matters](#l-vs-l2--why-the-distance-metric-matters)
- [Challenge 1 — FGSM on MNIST (Non-targeted, L∞)](#challenge-1--fgsm-on-mnist-non-targeted-l)
  - [The Setup](#the-setup)
  - [Final Solution](#challenge-1--final-solution)
- [Challenge 2 — DeepFool on MNIST (Targeted, L2)](#challenge-2--deepfool-on-mnist-targeted-l2)
  - [The Setup](#the-setup-1)
  - [Attempt 1 — Classic DeepFool (Wrong Direction)](#attempt-1--classic-deepfool-wrong-direction)
  - [Attempt 2 — Correct Direction, Insufficient Margin](#attempt-2--correct-direction-insufficient-margin)
  - [Attempt 3 — Full Margin with All-Class Comparison (Success)](#attempt-3--full-margin-with-all-class-comparison-success)
  - [Final Solution](#challenge-2--final-solution)
- [Skills Assessment 1 — I-FGSM on CIFAR-10 (Targeted, L∞)](#skills-assessment-1--i-fgsm-on-cifar-10-targeted-l)
  - [The Setup](#the-setup-2)
  - [Errors Along the Way](#errors-along-the-way-1)
  - [Final Solution](#skills-assessment-1--final-solution)
- [Skills Assessment 2 — DeepFool on CIFAR-10 (Non-targeted, L2 Normalized)](#skills-assessment-2--deepfool-on-cifar-10-non-targeted-l2-normalized)
  - [The Setup](#the-setup-3)
  - [Final Solution](#skills-assessment-2--final-solution)
- [Key Takeaways](#key-takeaways)

---

## Background — What Are First Order Attacks?

### The Gradient Intuition

Neural networks are trained by computing gradients — the direction in which to adjust the model's weights to reduce the loss. But gradients can be computed with respect to the **input** as well. If you know how changing each pixel affects the loss, you can move those pixels in the direction that increases it: pushing the model toward the wrong answer.

"First order" refers to the fact that these attacks only use **first-order information** (the gradient itself), not second-order curvature. This makes them computationally cheap and practically powerful.

```
Standard training:
  gradient w.r.t. weights  →  adjust weights to reduce loss

Adversarial attack:
  gradient w.r.t. input    →  adjust pixels to increase loss (or target a specific class)
```

### L∞ vs L2 — Why the Distance Metric Matters

All adversarial attacks are constrained by a budget: you can only perturb the input so much before the change becomes visible (or the challenge simply rejects it). The shape of that budget determines which math you use.

**L∞ (infinity norm):** limits the maximum change per pixel. Every pixel can move up to ε. The `sign()` of the gradient is used because we want to push every pixel by exactly ε in its most harmful direction, ignoring magnitude.

**L2 (Euclidean norm):** limits the total energy of the perturbation. A few pixels can move a lot, or many pixels can move a little — as long as the overall norm stays within the threshold. The gradient is divided by its norm to get a unit-direction vector, and we step along that direction.

This difference is why FGSM uses `sign(gradient)` and DeepFool uses `gradient / ‖gradient‖`.

---

## Challenge 1 — FGSM on MNIST (Non-targeted, L∞)

### The Setup

The course first walks through building three functions in the notebook:

- `_forward_and_loss` — runs the image through the model and returns logits + loss
- `_input_gradient` — computes the gradient of the loss with respect to the input image (not the weights)
- `fgsm_attack` — adds `ε × sign(gradient)` to the image and clips to [0, 1]

The challenge then provides an image, its true label, and an epsilon value via a REST API. We download the model weights, run FGSM locally, and submit the adversarial image.

The constraint is L∞ ≤ 0.25, which is generous — the perturbation is visible but the digit is still recognizable.

### Challenge 1 — Final Solution

The standalone attack script (self-contained, runs in a single cell):

```python
import os, io, base64, numpy as np, requests, torch, torch.nn as nn
from PIL import Image

BASE_URL = "http://<IP>:<PORT>"
MNIST_MEAN, MNIST_STD = 0.1307, 0.3081

def x01_from_b64(b64):
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw)).convert("L")
    return np.clip(np.asarray(img, dtype=np.float32) / 255.0, 0, 1)

def b64_from_x01(x):
    x255 = np.clip((x * 255.0).round(), 0, 255).astype(np.uint8)
    buf = io.BytesIO()
    Image.fromarray(x255, mode="L").save(buf, format="PNG")
    return base64.b64encode(buf.getvalue()).decode()

class SimpleClassifier(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(1, 32, 3, 1)
        self.conv2 = nn.Conv2d(32, 64, 3, 1)
        self.dropout1 = nn.Dropout(0.25)
        self.dropout2 = nn.Dropout(0.5)
        self.fc1 = nn.Linear(9216, 128)
        self.fc2 = nn.Linear(128, 10)
    def forward(self, x):
        x = (x - MNIST_MEAN) / MNIST_STD
        x = torch.relu(self.conv1(x))
        x = torch.relu(self.conv2(x))
        x = torch.max_pool2d(x, 2)
        x = self.dropout1(x)
        x = torch.flatten(x, 1)
        x = torch.relu(self.fc1(x))
        x = self.dropout2(x)
        return self.fc2(x)

# 1. Fetch challenge
ch = requests.get(f"{BASE_URL}/challenge").json()
x = x01_from_b64(ch["image_b64"])
lab = int(ch["label"])
eps = float(ch["epsilon"])

# 2. Load model
wt = requests.get(f"{BASE_URL}/weights").content
open("weights.pth", "wb").write(wt)
model = SimpleClassifier().eval()
model.load_state_dict(torch.load("weights.pth", map_location="cpu"))

# 3. FGSM with scale fallback
x_t = torch.from_numpy(x[None, None]).float().requires_grad_(True)
logits = model(x_t)
loss = nn.CrossEntropyLoss()(logits, torch.tensor([lab]))
loss.backward()
grad_sign = x_t.grad.sign().squeeze().numpy()

for scale in [1.0, 0.9, 0.8, 0.7]:
    x_adv = np.clip(x + scale * eps * grad_sign, 0, 1)
    r = requests.post(f"{BASE_URL}/submit", json={"image_b64": b64_from_x01(x_adv)}).json()
    print(f"scale={scale} | {r}")
    if r.get("ok"):
        print("FLAG:", r["flag"])
        break
```

The scale fallback loop isn't strictly necessary here but became a useful habit — sometimes the server rejects at the boundary due to floating point edge cases.

---

## Challenge 2 — DeepFool on MNIST (Targeted, L2)

### The Setup

Goal: take an image of a **4** and make the model predict **6**. Constraint: L2 distance ≤ 0.75.

DeepFool works by finding the closest decision boundary and stepping just across it, iteratively. In the targeted variant, we pick the boundary between the current class and our specific target class.

### Attempt 1 — Classic DeepFool (Wrong Direction)

First try used standard DeepFool geometry: compare the target class against the current prediction, step toward the target boundary. Result:

```
Wrong target: predicted 4, need 6
```

The attack never converged toward class 6. The direction calculation was comparing target vs current in the wrong sense — the step was moving away from the target boundary instead of toward it.

### Attempt 2 — Correct Direction, Insufficient Margin

After fixing the gradient sign (using subtraction instead of addition to update `x_adv`), the score for class 6 started climbing correctly:

```
iter 0   | score_target=1.906
iter 200 | score_target=5.834
✅ Target reached in 232 iterations!
```

But the server still responded with `predicted 4, need 6`. The local score gap was 0.001 (class 6 at 4.917, class 4 at 4.916). When the image was saved as PNG, pixel rounding shifted the balance by just enough to flip the prediction back.

**The lesson:** floating point is your enemy at the boundary. A margin of 0.001 is not a margin — it's a rounding error.

### Attempt 3 — Full Margin with All-Class Comparison (Success)

Two fixes applied together:

**Fix 1:** Instead of checking `score_target > score_original_class`, compare against **all other classes**. The server predicts the argmax — any class with a higher score than the target will override it.

**Fix 2:** Don't stop until `gap ≥ 2.0`. This absorbs PNG rounding noise with room to spare.

### Challenge 2 — Final Solution

```python
import os, io, base64, numpy as np, requests, torch, torch.nn as nn
from PIL import Image

BASE_URL = "http://<IP>:<PORT>"
MNIST_MEAN, MNIST_STD = 0.1307, 0.3081

def x01_from_b64(b64):
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw)).convert("L")
    return np.clip(np.asarray(img, dtype=np.float32) / 255.0, 0, 1)

def b64_from_x01(x):
    x255 = np.clip((x * 255.0).round(), 0, 255).astype(np.uint8)
    buf = io.BytesIO()
    Image.fromarray(x255, mode="L").save(buf, format="PNG")
    return base64.b64encode(buf.getvalue()).decode()

class SimpleClassifier(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(1, 32, 3, 1)
        self.conv2 = nn.Conv2d(32, 64, 3, 1)
        self.dropout1 = nn.Dropout(0.25)
        self.dropout2 = nn.Dropout(0.5)
        self.fc1 = nn.Linear(9216, 128)
        self.fc2 = nn.Linear(128, 10)
    def forward(self, x):
        x = (x - MNIST_MEAN) / MNIST_STD
        x = torch.relu(self.conv1(x))
        x = torch.relu(self.conv2(x))
        x = torch.max_pool2d(x, 2)
        x = self.dropout1(x)
        x = torch.flatten(x, 1)
        x = torch.relu(self.fc1(x))
        x = self.dropout2(x)
        return self.fc2(x)

# 1. Fetch challenge
ch = requests.get(f"{BASE_URL}/challenge").json()
x = x01_from_b64(ch["image_b64"])
lab = int(ch["label"])   # 4
tgt = int(ch["target"])  # 6
thr = float(ch["l2_threshold"])

# 2. Load model
wt = requests.get(f"{BASE_URL}/weights").content
open("deepfool_weights.pth", "wb").write(wt)
model = SimpleClassifier().eval()
model.load_state_dict(torch.load("deepfool_weights.pth", map_location="cpu"))

# 3. Targeted attack with safety margin
def targeted_attack(x_np, model, target, thr, max_iter=3000, margin=2.0):
    x_orig = torch.from_numpy(x_np[None, None]).float()
    x_adv = x_orig.clone()

    for i in range(max_iter):
        x_adv = x_adv.detach().requires_grad_(True)
        logits = model(x_adv)

        score_target = logits[0, target]
        best_other_score = max(
            float(logits[0, j].detach())
            for j in range(10) if j != target
        )
        gap = float(score_target.detach()) - best_other_score

        if gap >= margin:
            print(f"✅ Converged at iteration {i}! gap={gap:.3f}")
            break

        loss = -score_target + logits[0, lab]
        loss.backward()

        grad = x_adv.grad.detach()
        grad_norm = grad / (grad.norm() + 1e-8)

        l2_used = float((x_adv.detach() - x_orig).norm())
        budget_left = thr * 0.98 - l2_used
        if budget_left <= 0:
            print(f"⚠️ L2 budget exhausted at iteration {i}")
            break

        step_size = min(0.005, budget_left * 0.02)
        x_adv = (x_adv.detach() - step_size * grad_norm).clamp(0, 1)

        if i % 200 == 0:
            l2_cur = float((x_adv - x_orig).norm())
            print(f"iter {i} | gap={gap:.3f} | score_6={float(score_target.detach()):.3f} | L2={l2_cur:.4f}")

    return x_adv.detach().squeeze().numpy()

model.eval()
x_adv = targeted_attack(x, model, tgt, thr)

# 4. Verify before submitting
r_pred = requests.post(f"{BASE_URL}/predict", json={"image_b64": b64_from_x01(x_adv)}).json()
print(f"Server predict: {r_pred}")

if r_pred["pred"] == tgt:
    r = requests.post(f"{BASE_URL}/submit", json={"image_b64": b64_from_x01(x_adv)}).json()
    print(r)
    if r.get("ok"):
        print("FLAG:", r["flag"])
```

---

## Skills Assessment 1 — I-FGSM on CIFAR-10 (Targeted, L∞)

### The Setup

Goal: take a **dog** image (class 5) and make the model predict **cat** (class 3). Constraint: L∞ ≤ 8/255 ≈ 0.0314.

Two differences from the MNIST challenges: images are now RGB (3 channels, 32×32), and normalization is per-channel with different mean/std values. The attack is also **iterative** (I-FGSM): instead of one large step of ε, we take many small steps of α and project back into the ε-ball after each one.

### Errors Along the Way

**Error 1 — L∞ constraint violated on the server**

Locally, the perturbation measured at exactly 8/255. But the server rejected it:

```
L∞ constraint violated: 0.035294 > 0.031373
```

The gap: 0.035294 = 9/255, 0.031373 = 8/255. Saving as PNG rounds each pixel value to the nearest integer in [0, 255], which can add up to 1/255 of extra perturbation per pixel.

Fix: use `epsilon_safe = 7/255` internally. The attack still produces a visible perturbation, but after PNG rounding it stays within 8/255 on the server.

**Error 2 — Server predicts "automobile" instead of "cat"**

The attack was checking convergence by comparing `score_cat > score_dog` only. But `score_automobile` was even higher than both. The model picked automobile.

Fix: compare `score_cat` against **all other 9 classes** and stop only when it's the strict maximum with sufficient margin.

### Skills Assessment 1 — Final Solution

```python
import os, io, base64, requests
import torch
import torch.nn.functional as F
import torchvision.transforms as transforms
from PIL import Image
import numpy as np

BASE_URL = os.getenv("BASE_URL", "http://154.57.164.68:32597")  # ← change ça

# ─── HELPERS ──────────────────────────────────────────
def tensor_from_b64_png(b64):
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw))
    return transforms.ToTensor()(img)  # (3, 32, 32) in [0,1]

def b64_png_from_tensor(tensor):
    img_array = (tensor.permute(1, 2, 0).numpy() * 255).astype(np.uint8)
    img = Image.fromarray(img_array)
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    return base64.b64encode(buf.getvalue()).decode("ascii")

# ─── MODELE ───────────────────────────────────────────
weights_path = "cifar10_model_best.pth"
if not os.path.exists(weights_path):
    print("Téléchargement des poids...")
    resp = requests.get(f"{BASE_URL}/model/weights")
    open(weights_path, "wb").write(resp.content)

from model import load_model, NORMALIZATION_MEAN, NORMALIZATION_STD
device = "cuda" if torch.cuda.is_available() else "cpu"
model = load_model(weights_path, device=device)
model.eval()
print(f"Modèle chargé sur {device}")

# ─── CHALLENGE ────────────────────────────────────────
ch = requests.get(f"{BASE_URL}/challenge").json()
x = tensor_from_b64_png(ch["image"])          # (3, 32, 32) in [0,1]
orig_class = int(ch["original_class"])        # 5 = dog
target_class = int(ch["target_class"])        # 3 = cat
epsilon = float(ch["epsilon"])                # 8/255 ≈ 0.0314
mean = torch.tensor(NORMALIZATION_MEAN).view(3,1,1).to(device)
std  = torch.tensor(NORMALIZATION_STD).view(3,1,1).to(device)
print(f"Dog→Cat | epsilon={epsilon:.4f}")

# ─── I-FGSM CIBLÉ ─────────────────────────────────────
# Iterated FGSM : beaucoup de petits pas au lieu d'un grand
# Chaque pas = epsilon/max_iter, mais clampé dans le budget total

def ifgsm_targeted(x, model, target, epsilon, mean, std,
                   max_iter=500, alpha=None, margin=5.0):
    if alpha is None:
        alpha = epsilon / 20  # pas encore plus petit

    x_orig = x.clone().to(device)
    x_adv  = x.clone().to(device)

    for i in range(max_iter):
        x_adv = x_adv.detach().requires_grad_(True)
        x_norm = (x_adv - mean) / std
        logits = model(x_norm.unsqueeze(0))

        score_target = logits[0, target]
        # Comparer contre TOUTES les autres classes, pas juste dog
        best_other = logits[0].clone()
        best_other[target] = -9999
        score_other = best_other.max()
        gap = float((score_target - score_other).detach())

        if gap >= margin:
            print(f"✅ Convergé en {i} itérations! gap={gap:.3f}")
            break

        loss = -score_target + score_other
        loss.backward()

        grad = x_adv.grad.detach()
        x_adv = x_adv.detach() - alpha * grad.sign()
        x_adv = torch.max(torch.min(x_adv, x_orig + epsilon), x_orig - epsilon)
        x_adv = x_adv.clamp(0, 1)

        if i % 100 == 0:
            linf_d = float((x_adv - x_orig).abs().max())
            print(f"iter {i} | gap={gap:.3f} | score_cat={float(score_target.detach()):.2f} | best_other={float(score_other.detach()):.2f} | L∞={linf_d:.4f}")
    else:
        print(f"⚠️ Pas convergé (gap={gap:.3f})")

    return x_adv.detach().cpu()

# Utilise epsilon légèrement réduit pour compenser l'arrondi PNG
epsilon_safe = (7 / 255)  # au lieu de 8/255

x_adv = ifgsm_targeted(x, model, target_class, epsilon_safe, mean, std)

# Vérification locale
linf_dist = float((x_adv - x).abs().max())
print(f"L∞ locale: {linf_dist:.6f} (max: {epsilon:.6f})")

r_pred = requests.post(f"{BASE_URL}/predict",
                       json={"image": b64_png_from_tensor(x_adv)}).json()
print(f"Serveur predict: {r_pred['predicted_class_name']} (conf={r_pred['confidence']:.2f})")

if r_pred["predicted_class"] == target_class:
    r = requests.post(f"{BASE_URL}/submit",
                      json={"image": b64_png_from_tensor(x_adv)}).json()
    print(r)
    if r.get("flag"):
        print(f"\n🏁 FLAG: {r['flag']}")
```

---

## Skills Assessment 2 — DeepFool on CIFAR-10 (Non-targeted, L2 Normalized)

### The Setup

Goal: misclassify a **horse** image (class 7) into any other class. Constraint: L2 distance ≤ 3.5, computed in **normalized space** (after subtracting mean and dividing by std).

This last detail matters — the L2 budget is not in pixel space but in the space the model actually sees. A perturbation that's small in pixel space can be larger in normalized space if it aligns with a dimension that has small std.

### Skills Assessment 2 — Final Solution

```python
import os, io, base64, requests
import torch
import torchvision.transforms as transforms
from PIL import Image
import numpy as np

BASE_URL = "http://<IP>:<PORT>"

def tensor_from_b64_png(b64):
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw))
    return transforms.ToTensor()(img)

def b64_png_from_tensor(tensor):
    img_array = (tensor.permute(1, 2, 0).numpy() * 255).astype(np.uint8)
    img = Image.fromarray(img_array)
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    return base64.b64encode(buf.getvalue()).decode("ascii")

def l2_norm_space(a, b, mean_t, std_t):
    """Compute L2 distance in normalized space."""
    a_n = (a - mean_t) / std_t
    b_n = (b - mean_t) / std_t
    return float(torch.norm(a_n - b_n))

from model import load_model, NORMALIZATION_MEAN, NORMALIZATION_STD
device = "cuda" if torch.cuda.is_available() else "cpu"
model = load_model("cifar10_model_best.pth", device=device)
model.eval()

ch = requests.get(f"{BASE_URL}/challenge").json()
x = tensor_from_b64_png(ch["image"])
orig_class = int(ch["original_class"])   # 7 = horse
l2_threshold = float(ch["l2_threshold"]) # 3.5
mean_t = torch.tensor(ch["normalization"]["mean"]).view(3,1,1)
std_t  = torch.tensor(ch["normalization"]["std"]).view(3,1,1)

def deepfool(x, model, mean_t, std_t, num_classes=10, max_iter=100, overshoot=0.02):
    x_orig = x.clone().to(device)
    x_adv  = x.clone().to(device)
    mean_d = mean_t.to(device)
    std_d  = std_t.to(device)

    for i in range(max_iter):
        x_adv = x_adv.detach().requires_grad_(True)
        logits = model(((x_adv - mean_d) / std_d).unsqueeze(0))[0]
        pred = int(logits.argmax().item())

        if pred != orig_class:
            print(f"✅ Misclassified as {pred} after {i} iterations!")
            break

        logits[orig_class].backward(retain_graph=True)
        grad_orig = x_adv.grad.detach().clone()

        best_dist = float("inf")
        best_w = None

        for k in range(num_classes):
            if k == orig_class:
                continue
            x_adv2 = x_adv.detach().requires_grad_(True)
            logits2 = model(((x_adv2 - mean_d) / std_d).unsqueeze(0))[0]
            logits2[k].backward()
            grad_k = x_adv2.grad.detach()

            w_k = grad_k - grad_orig
            f_k = float((logits2[k] - logits[orig_class]).detach())
            dist_k = abs(f_k) / (w_k.norm() + 1e-8)

            if dist_k < best_dist:
                best_dist = dist_k
                best_w = w_k

        r = (best_dist + 1e-4) * best_w / (best_w.norm() + 1e-8)
        x_adv = (x_adv.detach() + (1 + overshoot) * r).clamp(0, 1)

        l2_cur = l2_norm_space(x_adv.cpu(), x_orig.cpu(), mean_t, std_t)
        if i % 10 == 0:
            print(f"iter {i} | pred={pred} | L2={l2_cur:.4f} / {l2_threshold}")
        if l2_cur > l2_threshold * 0.98:
            print(f"⚠️ L2 budget exhausted at iteration {i}")
            break

    return x_adv.detach().cpu()

x_adv = deepfool(x, model, mean_t, std_t)

l2_dist = l2_norm_space(x_adv, x, mean_t, std_t)
print(f"Normalized L2: {l2_dist:.4f} (max: {l2_threshold})")

r_pred = requests.post(f"{BASE_URL}/predict", json={"image": b64_png_from_tensor(x_adv)}).json()
print(f"Server: {r_pred['predicted_class_name']} conf={r_pred['confidence']:.2f}")

if r_pred["predicted_class"] != orig_class:
    r = requests.post(f"{BASE_URL}/submit", json={"image": b64_png_from_tensor(x_adv)}).json()
    print(r)
    if r.get("flag"):
        print("FLAG:", r["flag"])
```

---

## Key Takeaways

**1. `sign(gradient)` for L∞, `gradient / ‖gradient‖` for L2.**
The distance metric dictates the math. L∞ wants every pixel pushed by the same amount (ε) in the most harmful direction — hence the sign. L2 wants movement along the steepest direction without wasting budget on magnitude — hence the normalization. Getting this wrong means stepping in the wrong shape of space.

**2. PNG rounding is a real attack surface.**
Converting a float tensor to a PNG and back introduces up to 1/255 of error per pixel. For L∞ constraints this can exceed the budget. The fix is always to use `epsilon_safe = (n-1)/255` instead of the nominal value, or to quantize the image locally before measuring the perturbation.

**3. Always compare against all classes, not just the original.**
A targeted attack that checks `score_target > score_original` can still fail if a third class scores even higher. The prediction is an argmax over all classes — your target needs to win against everyone, not just the incumbent.

**4. Build in a score margin.**
A gap of 0.001 between target and runner-up will not survive PNG rounding, floating point conversion, or server-side preprocessing differences. Requiring `gap ≥ 2.0` before stopping is conservative but reliable.

**5. `.unsqueeze(0)` is non-negotiable.**
PyTorch models expect a batch dimension `(N, C, H, W)`. A single image is `(C, H, W)`. Forgetting `.unsqueeze(0)` produces a cryptic `ValueError: expected 4D input (got 3D input)`. Make it a reflex.

**6. Jupyter state is treacherous.**
All imports live in the kernel's memory. A restart wipes them silently. Any function defined in a cell can only be called after that cell has been run in the current session. When something breaks with a `NameError`, the first thing to check is whether the kernel needs a full restart and re-run.
