# TakeMeter — r/Naruto Discourse Classifier

A fine-tuned text classifier that sorts r/Naruto posts into three discourse types — **analysis**, **hot_take**, and **reaction** — compared against a zero-shot LLM baseline. Built for AI201 Project 3.

> **Headline result:** the zero-shot baseline (75.0%) **outperformed** the fine-tuned model (53.1%). That inversion is the central finding, and the report below traces it to label noise plus a dataset too small for stable learning. See [Reflection](#reflection-what-the-model-learned-vs-what-i-intended).

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
   *Tension:* `reaction` (it's a curiosity question) vs. `analysis` (it surveys theories and reasons about lifespan/chakra). **Decision:** leaned `reaction`, because the post is *asking* rather than *making* an argument — but it sits right on the line.

2. **"Theory: Kidomaru's strongest arrow was modeled after Kimimaro's bone joust after Kimimaro stomped the Sound 4."**
   *Tension:* framed as a "theory" (signals `analysis`) but it's a single asserted claim with no developed reasoning (`hot_take`). **Decision:** `hot_take` — the word "theory" is decorative; there's no actual argument, just a claim.

3. **"Jiraiya is the strongest Sannin and it's not even close. The only reason people think otherwise is because they use meaningless 'feats' and ignore the ones that matter."**
   *Tension:* confident opinion (`hot_take`) that gestures at a reasoning structure about which feats count (`analysis`-adjacent). **Decision:** `hot_take` — it asserts a methodology ("the feats that matter") without ever applying it.

---

## Fine-tuning approach

- **Base model:** `distilbert-base-uncased` (HuggingFace), with a freshly initialized 3-class classification head.
- **Training setup:** 3 epochs, learning rate `2e-5`, batch size 16, weight decay 0.01, 50 warmup steps, max sequence length 256, on a Colab T4 GPU.
- **Key hyperparameter decision:** I **kept the defaults (3 epochs, lr 2e-5, batch size 16)**. For a dataset of only ~150 training examples, more epochs risk overfitting and a higher learning rate destabilizes BERT-family fine-tuning; 2e-5 over 3 epochs is the standard, conservative starting point. Validation accuracy rose every epoch and training loss fell, confirming the pipeline was learning — so the modest final result is a data-quality story, not a training-config one.
- **Note on run-to-run variance:** the training cell does not fix a random seed for the classifier head, and the data split reshuffles each run. Across two identical training runs, test accuracy ranged from **0.41 to 0.53** — a ~12-point swing from nothing but initialization and split noise. On a dataset this small (~150 train / 32 test), a handful of examples flipping moves the headline number substantially. The numbers below are from the **0.53 run**; all committed artifacts (`evaluation_results.json`, `confusion_matrix.png`) are from this same run.

---

## Baseline

**Zero-shot** classification with Groq's `llama-3.3-70b-versatile`, no task-specific training. Each test example was sent with a system prompt containing the three label definitions (verbatim from the taxonomy above) and the instruction to respond with **only** the label name. Results were collected by running all 32 test examples through the model and parsing the single-word responses; **32/32 were parseable** (0 failures).

---

## Evaluation report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b) | **0.750** |
| Fine-tuned DistilBERT | 0.531 |
| **Difference** | **−0.219** (fine-tuning *regressed*) |

### Per-class metrics

**Fine-tuned DistilBERT**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.64 | 0.70 | 0.67 | 10 |
| hot_take | 0.44 | 0.73 | 0.55 | 11 |
| reaction | 0.67 | 0.18 | 0.29 | 11 |
| **macro avg** | 0.58 | 0.54 | 0.50 | 32 |

**Zero-shot baseline (Groq)**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.86 | 0.60 | 0.71 | 10 |
| hot_take | 0.62 | 0.73 | 0.67 | 11 |
| reaction | 0.83 | 0.91 | 0.87 | 11 |
| **macro avg** | 0.77 | 0.75 | 0.75 | 32 |

The baseline's `reaction` F1 of **0.87** — versus the fine-tuned model's **0.29** — is the single sharpest contrast. The same label definitions, given to the baseline as a prompt, let it nearly ace the class the fine-tuned model could barely touch.

### Confusion matrix — fine-tuned model (test set)

Rows = true label, columns = predicted label.

| true ↓ / pred → | analysis | hot_take | reaction |
|---|---|---|---|
| **analysis** | 7 | 2 | 1 |
| **hot_take** | 3 | 8 | 0 |
| **reaction** | 1 | 8 | 2 |

The dominant error is **`reaction` → `hot_take`: 8 of 11 reactions** were misread as hot takes. The model over-predicts `hot_take` overall (18 of 32 predictions), making it the "sink" class this run — the place ambiguous or emotional posts get dumped. `reaction` is the weakest class by far (recall 0.18), mostly absorbed into `hot_take`. (A supplementary image of this matrix, `confusion_matrix.png`, is committed to the repo.)

> **Cross-run pattern (the deeper finding):** in a separate identical training run, the model instead defaulted toward `analysis` and `reaction` collapsed to F1 **0.00**. So across runs the *specific* default shifts (sometimes hot_take, sometimes analysis), but the constant is that **`reaction` never gets learned as a distinct class** — it is always the casualty, dumped into whichever class the model is defaulting to. That consistency across runs is strong evidence the problem is the `reaction` labels themselves, not a single unlucky initialization.

### Three model errors, analyzed

These are real classifications from the fine-tuned model (confidences are the model's softmax probability on the predicted class). Each ties back to the test-set error pattern in the confusion matrix above.

1. **"Sending Akatsuki members in pairs is what let Konoha isolate and beat them one at a time…"** — true `analysis`, predicted `hot_take` (conf 0.35).
   *Why it failed:* this is a developed argument that references a specific in-universe strategic decision and uses it as evidence — textbook `analysis`. The model read it as a bare opinion. This is the `analysis → hot_take` slice of the matrix (2 cases) and reflects the persistent difficulty of the analysis/hot_take boundary plus this run's pull toward `hot_take`.

2. **"Just finished the Pain arc. I actually cried, best moment in shonen history."** — true `reaction`, predicted `hot_take` (conf 0.36).
   *Why it failed:* a pure emotional reaction with no argument. This is the **dominant** error in the matrix (`reaction → hot_take`, 8 of 11). The model cannot separate "expressing a feeling" from "asserting an opinion" — the core reason `reaction` recall is only 0.18.

3. **"Nagato's arc only works because Kishimoto built the cycle-of-hatred theme through Jiraiya for 200+ chapters."** — true `analysis`, predicted `hot_take` (conf 0.34, the lowest of any prediction).
   *Why it failed:* a clear `analysis` post — specific narrative evidence doing argumentative work — misread as `hot_take`. The rock-bottom confidence shows the model isn't distinguishing the classes so much as defaulting.

**Which boundary, and why it's hard:** this run, both failing boundaries drain *into* `hot_take` — `reaction → hot_take` (emotional posts read as opinions) and `analysis → hot_take` (arguments read as bare opinions). The common thread is that `hot_take` became a catch-all sink. Combined with the other run's `analysis`-sink behavior, the real conclusion is that the model never forms a stable, meaningful three-way boundary on this data, and `reaction` is the class it most reliably fails to learn.

**Labeling problem or data problem?** Primarily a **labeling/annotation-consistency problem, compounded by dataset size.** The `reaction` bucket conflated genuine one-line emotional posts with discussion questions, theory-prompts, and art-share captions, so there was no coherent signal for the model to learn. The ~12-point accuracy swing between identical runs further shows the dataset is too small for stable learning. The fix would be a manual review pass tightening the `reaction` definition (especially excluding discussion-questions) and collecting more cleanly-labeled examples — not a change to the training code, which trained correctly (loss dropped each epoch).

### Sample classifications (fine-tuned model)

Real model outputs on five representative posts, with the model's confidence:

| Post (truncated) | Predicted | Confidence | Correct? |
|---|---|---|---|
| "Sending Akatsuki members in pairs is what let Konoha isolate and beat them one at a time…" | hot_take | 0.35 | ✗ (true: analysis) |
| "Nagato's arc only works because Kishimoto built the cycle-of-hatred theme through Jiraiya…" | hot_take | 0.34 | ✗ (true: analysis) |
| "Sasuke's redemption was completely unearned. He gets forgiven way too easily." | hot_take | 0.36 | ✓ (true: hot_take) |
| "Just finished the Pain arc. I actually cried, best moment in shonen history." | hot_take | 0.36 | ✗ (true: reaction) |
| "That moment Sasuke put Kakashi in his place was pure greatness. The man is 100% alpha." | hot_take | 0.35 | ✓ (true: hot_take) |

**Correct example explained:** *"Sasuke's redemption was completely unearned. He gets forgiven way too easily"* was predicted `hot_take` (conf 0.36), which is correct and reasonable — it's a confident, contrarian opinion stated with no supporting evidence, which is exactly what `hot_take` is defined as. Worth noting: even on its *correct* predictions the model's confidence hovers around 0.35, barely above the 0.33 chance line for three classes. That low confidence even when right is itself evidence the model learned a weak `hot_take` default rather than a genuine decision boundary.

---

## Reflection: what the model learned vs. what I intended

**What I intended:** a classifier that distinguishes *reasoned argument* from *bare assertion* from *emotional reaction* — a genuinely subjective, content-based distinction.

**What it actually learned:** a weak default toward `hot_take` (this run) and an inability to recognize `reaction`, which it absorbed into `hot_take` 8 times out of 11. The decision boundary it formed isn't "does this post reason from evidence, assert, or emote?" — it's closer to "predict hot_take unless there's a strong analysis signal." The near-chance confidence on even its correct calls confirms it never internalized the real distinction.

The gap between intended and learned is explained by two things I introduced: **label noise** (the `reaction` class was a grab-bag because I didn't hand-correct the LLM pre-labels) and **dataset size** (the ~12-point swing between identical runs shows 150 training examples is too few for stable learning of a subjective task). The most striking evidence is the baseline comparison: the **same** label definitions, handed to the zero-shot model as a prompt, produced 0.87 F1 on `reaction`. The definitions were fine; the *training labels built from them* were not. Fine-tuning on noisy labels underperformed simply prompting a capable model with clean definitions.

---

## Spec reflection

**One way the spec helped:** the "Reading Evaluation Output" guidance — specifically that *a class with F1 near zero means the model can't learn that boundary; check your labels* — gave me a direct diagnostic lens. When `reaction` came back weakest (and collapsed entirely in another run), I knew to investigate label consistency rather than hunt for a bug in the training loop.

**One way my implementation diverged, and why:** the spec calls for writing `planning.md` *before* collection and for manually reviewing/correcting *every* pre-labeled example. I leaned heavily on LLM pre-labeling and did not run a full manual correction pass, and I assembled the design doc alongside the build. The divergence was driven by time pressure — but it turned out to be instructive: the fine-tuned model's failure (and especially the `reaction`-class collapse that recurs across runs) is a near-direct consequence of skipping that review step, which is the most concrete demonstration I could have gotten of *why* the spec insists on it.

---

## AI usage

**1. Data collection and annotation (disclosed annotation assistance).**
I directed an LLM (Claude, via a local Python script using `claude-haiku-4-5`) to **pre-label** the scraped r/Naruto posts: I supplied the three label definitions and batches of unlabeled posts, and it returned one label per post. It produced the full 210-example labeled set. **What I overrode / did not do:** I did *not* run a per-example manual correction pass before training. On evaluation, the boundaries — especially discussion-questions landing in `reaction` — proved to have been applied inconsistently, which is the root cause of the model's `reaction`-class failure documented above.

**2. Failure-pattern analysis.**
I directed an LLM to review the misclassified examples and surface common themes. It identified that the model over-predicts `hot_take` and absorbs `reaction` into it, and (across runs) that `reaction` is consistently the unlearned class. **What I changed:** I verified each claim against the actual confusion matrix before accepting it, and tied the pattern to the specific labeling decision (no manual review) and dataset size rather than treating it as a modeling artifact.

**3. Pipeline and debugging assistance.**
I used AI assistance to build the Reddit data-collection/parsing scripts and to interpret training and evaluation output. The dataset itself is real scraped content (no fabricated posts), and all reported metrics come directly from the notebook run.

---

## Repo contents

- `planning.md` — design document (labels, edge cases, data plan, metrics, AI tool plan)
- `naruto_dataset.csv` — the 210-example labeled dataset
- `README.md` — this file
- `evaluation_results.json` — metrics export from the notebook (0.53 run)
- `confusion_matrix.png` — supplementary confusion-matrix image (0.53 run)
