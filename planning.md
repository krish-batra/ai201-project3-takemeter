TakeMeter — Planning

Project: AI201 Project 3 — TakeMeter
Community: r/Naruto
Author: Krish Batra


1. Community

I chose r/Naruto, the main subreddit for the Naruto/Shippuden/Boruto franchise.

It's a strong fit for a discourse-quality classifier because the series is complete, which means the community produces an unusually wide range of post types at once: long retrospective lore breakdowns and power-scaling arguments, confident "hot take" opinion posts (the sub is full of "unpopular opinion" and "am I the only one" threads), and a constant stream of pure emotional reactions in episode/chapter discussion threads and fan-art posts. That spread — sustained argument vs. bare assertion vs. in-the-moment feeling — is exactly the distinction this project asks me to measure, and it's a distinction regulars in the community already recognize and argue about.

I can also collect well over 200 public posts and comments from it, which satisfies the data-volume requirement.


2. Labels

Three mutually exclusive labels, adapted from the project's strong-taxonomy example for an anime-discussion context:

analysis

The post makes a structured argument supported by specific evidence — arc references, character observations, thematic comparisons, or narrative logic. The reasoning is developed, not just asserted.


"Nagato's arc only works because Kishimoto spent 200+ chapters establishing the cycle-of-hatred theme through Jiraiya. Pain's invasion hits differently when you trace every beat back to that foundation."
"Sending Akatsuki members in pairs is what let Konoha isolate, study, and beat them one at a time instead of facing a combined force — the pacing was a strategic blunder in-universe, not just a writing choice."


hot_take

A bold, confident opinion stated without supporting evidence. The claim might be defensible, but the post asserts rather than argues. Often contrarian or provocative.


"Sasuke's redemption was completely unearned. He gets forgiven way too easily and the fandom pretends the last 100 chapters didn't happen."
"Minato was a terrible Hokage and an irresponsible father."


reaction

An immediate emotional response to a scene, arc, episode, or moment. Little to no argument — the post is expressing a feeling, hype, shock, or disappointment.


"Just finished the Pain arc for the first time. I actually cried. Best moment in shonen history, no debate."
"BRO that chapter. I was NOT ready."



3. Hard Edge Cases

The genuinely hard boundary is analysis vs. hot_take. Both are opinion-bearing; the difference is whether the post argues or merely asserts.

A representative ambiguous case:


"Luffy's Gear 5 was rushed — the buildup just wasn't there compared to Gear 4." (Naruto equivalent: "Gear 5 / Baryon Mode was rushed.")



This cites a comparison but doesn't develop it.

Decision rule: If the post references a specific narrative element (a chapter, arc, character beat) as evidence — even briefly — and that reference does real work in supporting the claim, label it analysis. If the post states the opinion with no grounded reference, or the reference is decorative (just enough to sound credible, not actually reasoning), label it hot_take. One vague gesture like "the buildup" with no specifics → hot_take.

A second, less obvious boundary surfaced during the project: reaction vs. the other two. Discussion-style questions ("How did Hashirama die? There are a lot of theories…") look like curiosity/reaction but invoke analysis-adjacent content, and they turned out to be a major source of confusion (see README). The rule I settled on: a post is reaction only if its primary purpose is to express a feeling — if it's posing a question to provoke discussion or theorizing, it leans analysis.


4. Data Collection Plan

Source: Public posts and top-level comments from r/Naruto, pulled via Reddit's public JSON endpoints (old.reddit.com/.../*.json) accessed through an authenticated logged-in browser session — no private channels, no content behind authentication.

Sortings sampled: top/all, top/year, hot, controversial/all, plus targeted searches (unpopular opinion, theory OR analysis) to surface the rarer argumentative classes.

Target: ~70 per label (210 total), to clear the 200-example minimum with a balanced distribution.

If a label is underrepresented: Pull additional sortings biased toward that class. In practice reaction was massively over-represented (art shares, hype, questions) and hot_take/analysis were rare, so I added controversial and the search-based pulls specifically to top up the two argumentative classes until each reached 70. Final dataset is balanced 70/70/70.


5. Evaluation Metrics

Accuracy alone is not enough for this task because the failure mode I most care about is a model that quietly ignores one class. On a balanced 3-class set, a model could post a mediocre-but-not-terrible accuracy while completely failing on one label.

So I use:


Overall accuracy — headline number, compared against the zero-shot baseline and the 33% random-guess floor.
Per-class precision / recall / F1 — the real diagnostic. F1 per class tells me whether each distinction is being learned. A class with F1 ≈ 0 is the signal that the model can't learn that boundary at all.
Confusion matrix — shows the direction of errors (e.g. analysis → hot_take), which points at exactly which boundary is broken.


These matter for this specific task because the whole point is distinguishing three kinds of discourse; a metric that hides per-class collapse would let me declare success while the classifier is useless for one category.


6. Definition of Success

For this classifier to be genuinely useful as a community tool (e.g. auto-tagging posts by discourse type), I'd want:


Overall accuracy meaningfully above the zero-shot baseline — fine-tuning should add something over just prompting a big model.
All three per-class F1 ≥ 0.65, with no class collapsed to near-zero. A tool that never recognizes one of the three discourse types isn't deployable.


"Good enough" for a real deployment would be ~0.75+ accuracy with balanced per-class F1. Below that, or with any class near zero, it's a research artifact rather than a usable tool.

(Outcome, for the record: the fine-tuned model did not hit this bar — see README. The honest analysis of why is the most valuable part of this project.)


7. AI Tool Plan

This project has no application code to generate, so AI assistance is concentrated in three places:

Label stress-testing

Before committing to the taxonomy, I used an LLM to generate borderline analysis/hot_take posts to check whether my definitions held up. Where it produced posts I couldn't cleanly classify, that flagged the boundary as needing the explicit decision rule in Section 3.

Annotation assistance — used, and disclosed

I used an LLM (Claude) to pre-label the scraped posts: I gave it the label definitions above and a batch of unlabeled posts, and it assigned one label per post. This is the optional pre-labeling workflow the spec allows.

Honest note on scope: I did not run a full manual correction pass over every pre-assigned label before training. As the evaluation shows, that decision had a direct, visible cost — the reaction class was labeled inconsistently (questions, theories, art-shares, and one-line reactions all landed in it), and the fine-tuned model collapsed on that class. This is documented as both an AI-usage disclosure and the core finding of the evaluation.

Failure analysis

After evaluation, I pasted the misclassified examples to an LLM and asked it to identify patterns (which label pair was being confused, whether short/ambiguous posts clustered, etc.), then verified the patterns against the confusion matrix myself. It identified the "reaction never predicted, analysis over-predicted" collapse, which I confirmed directly.
