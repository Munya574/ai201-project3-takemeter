# TakeMeter — Question-Quality Classifier for r/sewing

A fine-tuned text classifier (AI201 Project 3) that distinguishes **detailed** from **low-effort** questions in the r/sewing *Simple Questions* threads. This README is the final report; `planning.md` holds the design notes and decisions made before and during annotation.

# DEMO Video Link
https://www.loom.com/share/e4f047de6674423ab2b8d70d4d159997

## Community

I chose **r/sewing**, specifically its recurring weekly *Simple Questions* threads. I sew myself (Brother XM2701), so I can read the discourse the way a regular would.

After reading real threads I found r/sewing isn't an opinion space — the *Simple Questions* thread is a help forum, so a "hot take vs. analysis" axis doesn't apply. The real variation is the **substance of the question itself**: some askers give a machine model, what they tried, measurements, and constraints; others post a photo with "help, it's broken" and almost no written context. That gap matters to the community, because detailed questions get fast, accurate answers while low-effort ones get a string of "can you add more detail?" replies. That is the distinction this classifier measures.

## Label taxonomy

This is a **2-label** task. Only **question posts** are labeled — answers and thank-you replies are dropped. The labels are mutually exclusive and cover essentially all question posts (no "other" bucket).

**`detailed_question`** — the written text alone gives enough to answer or meaningfully engage: a named machine model with symptoms and what was tried, a named pattern or specific features, measurements, or a clear stated constraint (budget, fabric, deadline, skill level used as real context).
- *Example:* "Singer 1304 jams with a thread nest, bobbin numbered"
- *Example:* "Janome HD3000 vs DC3050, current specs, prices, goals"

**`low_effort_question`** — the post leans on a photo with little written context, is vague ("it's broken, help"), or is a bare "how do I make this?" / "what's this?" with no constraints.
- *Example:* "What's this line by the buttons? (photo only)"
- *Example:* "What brand should I buy? no constraints given"

## Data collection, labeling, and distribution

**Source.** r/sewing weekly *Simple Questions* threads, collected manually from old.reddit.com, sampled across multiple dates (Jan 2025, June 2026, Dec 2021, Nov 2021, and others) so no single thread's quirks dominate.

**Labeling process.** I labeled each question post against the definitions above. I used Claude to pre-label batches against my definitions, then reviewed and corrected every label myself (disclosed in the AI Usage section). Each row passed my own review. The `text` column stores a faithful **condensed summary** of each post rather than verbatim Reddit text (e.g. "Corset neckline gapes when enlarging the bust"); the label reflects the original post's level of written detail. The CSV has three columns: `text`, `label`, `notes` (notes filled for borderline cases).

**Distribution (222 examples):**

| Label | Count | Share |
|---|---|---|
| detailed_question | 180 | 81.1% |
| low_effort_question | 42 | 18.9% |

This clears the 200 minimum and sits under the spec's 70%-majority red flag. The minority class is just under the "aim for 20%" guideline; I accepted the real distribution rather than fabricating or force-trimming examples. **This imbalance turned out to be the central factor in the result** (see Evaluation).

**Three difficult-to-label examples (decisions in `planning.md`):**
1. *"Specific use case (oven mitts), no prior research shown"* → `low_effort_question`. A use case alone isn't enough to engage with.
2. *"Named machine, brief symptom, minimal detail"* → `low_effort_question`. A name without a described problem is decorative; the answerer still can't work with it.
3. *"Jersey stretch problem, general but some context (months sewing)"* → `low_effort_question`. Stating skill level is background, not a specific question.

The throughline: **context about the asker is not the same as specifics about the question.**

## Fine-tuning approach

- **Base model:** `distilbert-base-uncased` with a 2-label classification head.
- **Training setup:** Google Colab T4 GPU; 70/15/15 stratified train/val/test split (34-example test set); the unedited starter pipeline.
- **Hyperparameter decision:** I kept the default **3 epochs** (learning rate 2e-5, batch size 16). With only ~155 detailed and ~29 low-effort examples in the training split, more epochs risk overfitting, so I left the conservative default in place. As the results show, the limiting factor was not epoch count but class imbalance.

## Baseline

Zero-shot classification with Groq `llama-3.3-70b-versatile`. The prompt gave both label definitions (copied from `planning.md`), one example per label, and instructed the model to output only the label name. It classified all 34 test examples; every response was parseable (34/34).

## Evaluation report

### Headline metrics

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq) | 0.676 |
| Fine-tuned DistilBERT | 0.794 |

On accuracy alone, fine-tuning "improved" the model by 11.8 points. **That number is misleading**, and the per-class metrics show why.

### Per-class metrics

**Zero-shot baseline (Groq):**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| detailed_question | 1.00 | 0.59 | 0.74 | 27 |
| low_effort_question | 0.39 | 1.00 | 0.56 | 7 |
| **macro avg** | 0.69 | 0.80 | 0.65 | 34 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| detailed_question | 0.79 | 1.00 | 0.88 | 27 |
| low_effort_question | 0.00 | 0.00 | 0.00 | 7 |
| **macro avg** | 0.40 | 0.50 | 0.44 | 34 |

The fine-tuned model's **macro-F1 (0.44) is *worse* than the baseline's (0.65)**, despite higher accuracy. The accuracy gain is an illusion created by the class imbalance.

### Confusion matrix (fine-tuned model)

| | Predicted: detailed | Predicted: low_effort |
|---|---|---|
| **True: detailed** | 27 | 0 |
| **True: low_effort** | 7 | 0 |

The model predicted `detailed_question` for **all 34 test examples** and never once predicted `low_effort_question`. It scored 79.4% purely because 27/34 of the test set is detailed — it learned to always output the majority class. This is **majority-class collapse**, the exact risk flagged in `planning.md`'s evaluation section.

### Against the success criteria

`planning.md` defined success as: beat the baseline on accuracy **and** reach `low_effort_question` F1 ≥ 0.55 with recall ≥ 0.50. The model **cleared the accuracy bar and failed the one that mattered** (F1 = 0.00, recall = 0.00). I set those thresholds specifically to catch a model that wins on accuracy while failing the real task — and they did their job.

### Three wrong predictions, analyzed

All 7 errors were `low_effort_question` posts predicted as `detailed_question`. Three:

1. **"Fabric ID, link only, no context"** — true low_effort, predicted detailed (confidence 0.54). Unambiguously low-effort by my locked rule: a bare link with no written specifics. The model had no basis to call it detailed except its default.
2. **"How do I practice sewing curves? beginner"** — true low_effort, predicted detailed (confidence 0.53). A general how-to with no project context — a clean low-effort case the model missed.
3. **"Gift budget question, no real sewing content"** — true low_effort, predicted detailed (confidence 0.53). No sewing specifics at all, yet still labeled detailed.

**Which boundary is hard, and why.** It is not that one specific boundary confuses the model — the model learned *no* boundary. Every error runs in one direction (low_effort → detailed), and every confidence sits at 0.51–0.54, barely above chance. The model isn't confidently mistaken; it is uncertain and defaults to the majority class. **This is a data-distribution problem, not a labeling problem:** all three examples are consistently and obviously low-effort, so the issue is that ~29 minority training examples are too few for DistilBERT to learn the minority class against a 4:1 majority in 3 epochs.

**The strongest finding:** the zero-shot baseline caught **100%** of low-effort questions (recall 1.00) with no task-specific training, while the fine-tuned model caught **none**. For the only job this tool exists to do — flag low-effort posts so askers can be nudged to add detail — **the untrained baseline is the more useful model**, even though it scores 12 points lower on accuracy.

### Sample classifications (fine-tuned model)

| Post | Predicted | Confidence | Correct? |
|---|---|---|---|
| "Janome HD3000 vs DC3050, current specs, prices, goals" | detailed_question | (high) | ✅ |
| "Singer 1304 jams with a thread nest, bobbin numbered" | detailed_question | (high) | ✅ |
| "Fabric ID, link only, no context" | detailed_question | 0.54 | ❌ (true: low_effort) |
| "How do I practice sewing curves? beginner" | detailed_question | 0.53 | ❌ (true: low_effort) |
| "ID two unknown presser feet (photo only)" | detailed_question | 0.53 | ❌ (true: low_effort) |

The first prediction is reasonable: the post names two specific machines and the deciding factors in writing, which is exactly the signal `detailed_question` is meant to capture — the model gets the majority class right because it predicts it for everything.

## Reflection: what the model learned vs. what I intended

I intended the model to learn the **concept**: does a question contain written specifics? It instead learned the **base rate**: detailed is the safe bet, so predict it every time. The decision boundary it formed is not "specifics vs. no specifics" — it is effectively a constant function.

The give-away is the confidence distribution. On the 7 misses, confidence sat at 0.51–0.54 — the model was near-maximally uncertain and still chose the majority class, because that minimizes training loss on an imbalanced set. It didn't overfit to a spurious feature; it underfit the minority class entirely. What it "captured" was my dataset's distribution, not my labeling concept. The fix isn't a tighter definition (the definitions worked — Groq applied them perfectly from the same prompt); it's more minority examples or loss weighting to counteract the imbalance. That gap — between a label scheme that a zero-shot model can follow and a fine-tuned model that ignored it for the base rate — is the real lesson of this project.

## Spec reflection

**One way the spec helped:** the spec's warning that >95% accuracy on a hard subjective task is a red flag, plus its insistence on per-class metrics over raw accuracy, is exactly why I built per-class success thresholds into `planning.md` before training. Without that framing I would have seen 79.4% and called it a win. The spec trained me to distrust the headline number — which is the only reason I caught the collapse.

**One way my implementation diverged:** the spec frames the task around "hot take vs. analysis" opinion discourse. After reading real threads I diverged to a question-quality (detailed vs. low-effort) axis, because r/sewing is a help forum, not an opinion space, and the opinion framing simply didn't fit the data. The spec explicitly allows choosing your own community and labels, so this is a sanctioned divergence — but it's a real one, and it's why my taxonomy looks nothing like the project page's running example.

## AI usage

**1. Label stress-testing and edge-case design.** I gave Claude my two definitions and the ambiguous photo-plus-text case and had it surface where the boundary was thin. This produced the locked rule that a photo never substitutes for written detail, and the sharpening that "context about the asker ≠ specifics about the question." I made the final calls.

**2. Annotation assistance.** I used Claude to pre-label batches of collected posts against my definitions, then reviewed and corrected every label myself — overriding the first-pass label on borderline named-machine-thin-symptom and photo-plus-text cases. Every row in the final CSV passed my own review.

**3. Failure-pattern analysis.** After the run I gave Claude the misclassified examples and the confusion matrix and asked it to identify the pattern. It surfaced the one-directional collapse and the 0.51–0.54 confidence signature; I verified both by re-reading the 7 misses and checking the confidence values myself before writing this report.

## Files in this repo

- `planning.md` — design notes, label definitions, edge-case rules, success criteria
- `sewing_questions.csv` — 222 labeled examples (`text`, `label`, `notes`)
- `evaluation_results.json` — accuracy summary exported from the notebook
- `confusion_matrix.png` — fine-tuned model confusion matrix (supplementary to the table above)
