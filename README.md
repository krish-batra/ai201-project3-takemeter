# TakeMeter — r/Naruto Discourse Classifier

A fine-tuned text classifier that sorts r/Naruto posts into three discourse types — **analysis**, **hot_take**, and **reaction** — compared against a zero-shot LLM baseline. Built for AI201 Project 3.

> **Headline result:** the zero-shot baseline (75.0%) substantially **outperformed** the fine-tuned model (40.6%). That inversion is the central finding, and the report below traces it to label noise in the training data. See [Reflection](#reflection-what-the-model-learned-vs-what-i-intended).

---

## Community choice and reasoning

**r/Naruto** — the main subreddit for the Naruto franchise. Because the series is complete, the community produces a wide spread of post types simultaneously: developed lore/power-scaling arguments, confident contrarian opinion posts, and pure in-the-moment reactions in discussion and fan-art threads. That spread — *reasoned argument vs. bare assertion vs. emotional reaction* — is exactly the distinction this project measures, and it's one regulars in the community already argue about. It also has more than enough public posts to clear the 200-example minimum.

---

## Label taxonomy

| Label | Definition | Examples |
|---|---|---|
| **analysis** | A structured argument supported by specific evidence — arc references, character observations, thematic comparisons, or narrative logic. Reasoning is developed, not asserted. | • *"Nagato's arc only works because Kishimoto spent 200+ chapters establishing the cycle-of-hatred theme through Jiraiya."* <br>• *"Sending Akatsuki in pairs is what let Konoha isolate and beat them one at a time instead of facing a combined force."* |
| **hot_take** | A bold, confident opinion stated without supporting evidence. May be defensible, but asserts rather than argues. Often contrarian. | • *"Sasuke's redemption was completely unearned. He gets forgiven way too easily."* <br>• *"Minato was a terrible Hokage and an irresponsible father."* |
| **reaction** | An immediate emotional response to a scene, arc, or moment. Little to no argument — feeling, hype, shock, or disappointment. | • *"Just finished the Pain arc. I actually cried. Best moment in shonen history."* <br>• *"BRO that chapter. I was NOT ready."* |

**Decision rule for the hard boundary (`analysis` vs `hot_take`):** if the post references a specific narrative element as evidence *and that reference does real work*, label `analysis`; if the opinion is stated with no grounded reference (or the reference is decorative), label `hot_take`.

---

## Data: source, labeling process, and distribution

**Source.** Real public posts and top-level comments from r/Naruto, pulled from Reddit's public JSON endpoints (`old.reddit.com/.../*.json`) via an authenticated logged-in browser session — no private channels, no synthetic data. Sortings sampled: `top/all`, `top/year`, `hot`, `controversial/all`, plus targeted searches for `unpopular opinion` and `theory OR analysis` to surface the rarer argumentative classes. After combining title + body, filtering to 25–700 characters, dropping `[deleted]`/`[removed]`, and deduplicating, the raw corpus held **438 unique texts**.

**Labeling process.** The 438 texts were **pre-labeled by an LLM (Claude)** using the definitions above (see [AI usage](#ai-usage)). The set was then balanced to **70 per label**. Labels were *not* individually hand-corrected before training — a decision whose cost is analyzed below and disclosed in the AI-usage section.

**Final label distribution (210 examples):**

| Label | Count |
|---|---|
| analysis | 70 |
| hot_take | 70 |
| reaction | 70 |

**Train / val / test split** (70/15/15, stratified): train **147**, validation **31**, test **32** (test = 10 analysis, 11 hot_take, 11 reaction).

### Three genuinely difficult-to-label examples

> *Confirm these reflect calls you'd actually make — they're real ambiguous posts from the dataset.*

1. **"How did Hashirama die? He died during the first great ninja war but it was never stated how… the biggest theory is old age but is it possible he got defeated by…"**
   *Tension:* `reaction` (it's a curiosity question) vs. `analysis` (it surveys theories and reasons about lifespan/chakra). **Decision:** leaned `reaction`, because the post is *asking* rather than *making* an argument — but it sits right on the line, and the model's disagreement here (it predicted `analysis`) is defensible.

2. **"Theory: Kidomaru's strongest arrow was modeled after Kimimaro's bone joust after Kimimaro stomped the Sound 4."**
   *Tension:* framed as a "theory" (signals `analysis`) but it's a single asserted claim with no developed reasoning (`hot_take`). **Decision:** `hot_take` — the word "theory" is decorative; there's no actual argument, just a claim.

3. **"Jiraiya is the strongest Sannin and it's not even close. The only reason people think otherwise is because they use meaningless 'feats' and ignore the ones that matter."**
   *Tension:* confident opinion (`hot_take`) that gestures at a reasoning structure about which feats count (`analysis`-adjacent). **Decision:** `hot_take` — it asserts a methodology ("the feats that matter") without ever applying it.

---

## Fine-tuning approach

- **Base model:** `distilbert-base-uncased` (HuggingFace), with a freshly initialized 3-class classification head.
- **Training setup:** 3 epochs, learning rate `2e-5`, batch size 16, weight decay 0.01, 50 warmup steps, max sequence length 256, on a Colab T4 GPU.
- **Key hyperparameter decision:** I **kept the defaults (3 epochs, lr 2e-5, batch size 16)**. For a dataset of only ~150 training examples, more epochs risk overfitting and a higher learning rate destabilizes BERT-family fine-tuning; 2e-5 over 3 epochs is the standard, conservative starting point. Validation accuracy rose every epoch (**0.355 → 0.419 → 0.484**) and training loss fell (**1.100 → 1.091 → 1.074**), confirming the pipeline was learning — so the modest final result is a data-quality story, not a training-config one.

---

## Baseline

**Zero-shot** classification with Groq's `llama-3.3-70b-versatile`, no task-specific training. Each test example was sent with a system prompt containing the three label definitions (verbatim from the taxonomy above) and the instruction to respond with **only** the label name. Results were collected by running all 32 test examples through the model and parsing the single-word responses; **32/32 were parseable** (0 failures).

---

## Evaluation report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b) | **0.750** |
| Fine-tuned DistilBERT | 0.406 |
| **Difference** | **−0.344** (fine-tuning *regressed*) |

### Per-class metrics

**Fine-tuned DistilBERT**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.42 | 1.00 | 0.59 | 10 |
| hot_take | 0.38 | 0.27 | 0.32 | 11 |
| reaction | 0.00 | 0.00 | **0.00** | 11 |
| **macro avg** | 0.26 | 0.42 | 0.30 | 32 |

**Zero-shot baseline (Groq)**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.86 | 0.60 | 0.71 | 10 |
| hot_take | 0.62 | 0.73 | 0.67 | 11 |
| reaction | 0.83 | 0.91 | 0.87 | 11 |
| **macro avg** | 0.77 | 0.75 | 0.75 | 32 |

### Confusion matrix — fine-tuned model (test set)

Rows = true label, columns = predicted label.

| true ↓ / pred → | analysis | hot_take | reaction |
|---|---|---|---|
| **analysis** | 10 | 0 | 0 |
| **hot_take** | 8 | 3 | 0 |
| **reaction** | 6 | 5 | 0 |

The `reaction` column is **entirely zero** — the model never once predicted `reaction`. Every true reaction was pushed into `analysis` (6) or `hot_take` (5). Meanwhile `analysis` has recall 1.00 because the model defaulted to predicting `analysis` for almost everything. (A supplementary image of this matrix, `confusion_matrix.png`, is committed to the repo.)

### Three wrong predictions, analyzed

1. **"How did Hashirama die? …there are many theories on it…"** — true `reaction`, predicted `analysis` (conf 0.39).
   *Why it failed:* this is the exact boundary problem from the [difficult-examples](#three-genuinely-difficult-to-label-examples) section. The post is a discussion *question*, which I labeled `reaction`, but its theory-surveying content reads like `analysis`. The model isn't unreasonable here — the label itself is contestable. This is an **annotation-consistency** problem: questions like this were not labeled consistently across the dataset.

2. **"That moment Sasuke put Kakashi in his place was pure greatness. The man is 100% alpha."** — true `hot_take`, predicted `analysis` (conf 0.37).
   *Why it failed:* a bare opinion with zero argument — a textbook `hot_take`. The model still guessed `analysis`, reflecting its overall collapse toward predicting `analysis` by default rather than any signal in the text.

3. **"Gaara pov when Rock Lee dropped his weights."** — true `reaction`, predicted `hot_take` (conf 0.36).
   *Why it failed:* a short, meme-style reaction. The model never predicts `reaction` at all, so it had to route this somewhere, and the low confidence (0.36) shows it was essentially guessing.

**Which boundary, and why it's hard:** the dominant error is everything collapsing *away from* `reaction` and *toward* `analysis`. The boundary the model never learned is "what makes something a reaction." That's traceable directly to the data: the `reaction` bucket conflated genuine one-line emotional posts with discussion questions, theory-prompts, and art-share captions, so there was no coherent linguistic signal for the model to latch onto. With only ~11 noisy reaction examples in training-equivalent terms and no consistent pattern, the model rationally abandoned the class.

**Labeling problem or data problem?** Primarily a **labeling/annotation-consistency problem.** The pipeline trained correctly (loss dropped each epoch); the ceiling was set by inconsistent pre-labels that were never hand-corrected. The fix would be a manual review pass tightening the `reaction` definition (especially excluding discussion-questions) and re-labeling, not a change to the training code.

### Sample classifications (fine-tuned model)

| Post (truncated) | Predicted | Confidence | Correct? |
|---|---|---|---|
| "That moment Sasuke put Kakashi in his place was pure greatness. The man is 100% alpha." | analysis | 0.37 | ✗ (true: hot_take) |
| "Gaara pov when Rock Lee dropped his weights." | hot_take | 0.36 | ✗ (true: reaction) |
| "Jiraiya is the strongest Sannin and it's not even close…" | analysis | 0.37 | ✗ (true: hot_take) |
| *(correctly-predicted analysis post — fill in with the snippet below)* | analysis | *0.__* | ✓ |

> **One correct example explained (fill confidence from snippet):** any genuine `analysis` post — e.g. a developed Hashirama-death-cause or Akatsuki-pairing argument — was predicted `analysis`. This prediction is reasonable because such posts contain the exact features the `analysis` label is defined by: specific narrative references doing argumentative work. (The fine-tuned model's `analysis` recall was 1.00, so it caught every real analysis post — at the cost of over-predicting the class.)

To get a real confidence value for a correctly-predicted example, run this in the notebook after training:

```python
import torch
samples = [
    "Sending Akatsuki members in pairs is what let Konoha isolate and beat them one at a time.",
    "Nagato's arc only works because Kishimoto built the cycle-of-hatred theme through Jiraiya for 200+ chapters.",
]
enc = tokenizer(samples, truncation=True, padding=True, max_length=256, return_tensors="pt").to(model.device)
with torch.no_grad():
    probs = torch.softmax(model(**enc).logits, dim=-1)
for s, p in zip(samples, probs):
    i = int(p.argmax())
    print(f"{ID_TO_LABEL[i]:10s}  conf={p[i]:.2f}  | {s[:60]}")
```

---

## Reflection: what the model learned vs. what I intended

**What I intended:** a classifier that distinguishes *reasoned argument* from *bare assertion* from *emotional reaction* — a genuinely subjective, content-based distinction.

**What it actually learned:** a weak default toward `analysis` and a complete inability to recognize `reaction`. The decision boundary it formed isn't "does this post reason from evidence?" — it's closer to "predict analysis, occasionally hot_take, never reaction." The model didn't capture the conceptual distinction I cared about; it overfit to whatever shallow signal survived in the noisier-labeled classes and gave up on the class with the least coherent labels.

The gap between intended and learned is almost entirely explained by **label noise I introduced by not hand-correcting the LLM pre-labels.** The `reaction` class, in particular, was a grab-bag — and a model can only learn distinctions that are *consistently present* in its labels. The most striking evidence is the comparison: the **same** label definitions, handed to the zero-shot baseline as a prompt, produced a 0.87 F1 on `reaction`. The definitions were fine; the *training labels built from them* were not. Fine-tuning on noisy labels underperformed simply prompting a capable model with clean definitions.

---

## Spec reflection

**One way the spec helped:** the "Reading Evaluation Output" guidance — specifically that *one class with F1 ≈ 0 means the model can't learn that boundary; check your labels* — gave me a direct diagnostic lens. The moment `reaction` came back at F1 0.00, I knew to investigate label consistency rather than hunt for a bug in the training loop. Without that pointer I might have wasted time tweaking hyperparameters on what was actually a data problem.

**One way my implementation diverged, and why:** the spec calls for writing `planning.md` *before* collection and for manually reviewing/correcting *every* pre-labeled example. I leaned heavily on LLM pre-labeling and did not run a full manual correction pass, and I assembled the design doc alongside the build rather than strictly before it. The divergence was driven by time pressure — but it turned out to be instructive rather than purely a shortcut: the fine-tuned model's failure is a near-direct consequence of skipping that review step, which is the most concrete demonstration I could have gotten of *why* the spec insists on it.

---

## AI usage

**1. Data collection and annotation (disclosed annotation assistance).**
I directed an LLM (Claude, via a local Python script using `claude-haiku-4-5`) to **pre-label** the scraped r/Naruto posts: I supplied the three label definitions and batches of unlabeled posts, and it returned one label per post. It produced the full 210-example labeled set. **What I overrode / did not do:** I did *not* run a per-example manual correction pass before training. On evaluation, the `analysis`/`hot_take`/`reaction` boundaries — especially around discussion-questions landing in `reaction` — proved to have been applied inconsistently, which is the root cause of the model's `reaction`-class collapse documented above.

**2. Failure-pattern analysis.**
I directed an LLM to review the list of misclassified test examples and surface common themes. It identified the systematic pattern that the model never predicts `reaction` and over-predicts `analysis`, and flagged that the `reaction` training examples were heterogeneous (questions, theories, art captions, one-liners). **What I changed:** I verified each claim against the actual confusion matrix before accepting it, and tied the pattern back to the specific labeling decision (no manual review) rather than treating it as a modeling artifact.

**3. Pipeline and debugging assistance.**
I used AI assistance to build the Reddit data-collection/parsing scripts and to interpret training and evaluation output. The dataset itself is real scraped content (no fabricated posts), and all reported metrics come directly from the notebook run.

---

## Repo contents

- `planning.md` — design document (labels, edge cases, data plan, metrics, AI tool plan)
- `naruto_dataset.csv` — the 210-example labeled dataset
- `README.md` — this file
- `evaluation_results.json` — metrics export from the notebook
- `confusion_matrix.png` — supplementary confusion-matrix image
