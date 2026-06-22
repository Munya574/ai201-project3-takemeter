# planning.md — TakeMeter (AI201 Project 3)

## 1. Community

I chose **r/sewing**, specifically its recurring weekly **Simple Questions** threads.

I picked this community because I sew myself (Brother XM2701), so I can read the discourse the way a regular would and judge what actually counts as a useful question. I considered the more common "hot take vs. analysis" subs (r/nba, r/television) but rejected them: after reading real threads I found r/sewing isn't an opinion space at all. The Simple Questions thread is a help forum — people ask, people answer, people say thanks. That ruled out any quality axis based on *boldness of opinion*.

What makes the discourse varied enough to classify is the **substance of the question itself**. Some askers give a machine model, what they tried, measurements, and constraints; others post a photo with "help, it's broken" and almost no written context. That gap is real, it's visible, and it matters to the community — detailed questions get fast, accurate answers while low-effort ones get a string of "can you post more detail?" replies. That's the distinction my classifier measures.

## 2. Labels

This is a **2-label** task. I label only the **question posts** in each thread — answers and thank-you replies are dropped, since they aren't part of a question-quality task. The two labels are mutually exclusive and cover essentially all question posts (no "other" bucket needed).

### `detailed_question`
The written text alone gives enough to answer or meaningfully engage: a named machine model plus symptoms and what was tried, a named pattern or specific features, measurements, or a clear stated constraint (budget, fabric, deadline, skill level used as real context).

- *Example:* "Singer 1304 jams with a thread nest, bobbin numbered" — names the machine, the symptom, and a relevant detail.
- *Example:* "Janome HD3000 vs DC3050, current specs, prices, goals" — names both machines and states the deciding factors in writing.

### `low_effort_question`
The post leans on a photo with little written context, is vague ("it's broken, help"), or is a bare "how do I make this?" / "what's this?" with no constraints.

- *Example:* "What's this line by the buttons? (photo only)" — photo-reliant identification request with no written specifics.
- *Example:* "What brand should I buy? no constraints given" — no budget, no use case, no context.

## 3. Hard edge cases

The genuinely ambiguous case is the **photo-plus-some-text post**: an asker attaches a photo *and* writes a sentence or two. Does the writing carry the question, or is the photo doing the work?

**Decision rule (locked before annotating):** A photo never substitutes for written detail. I label on the written text only. If the text alone names specifics (machine, measurements, what was tried, a constraint), it's `detailed_question` even with a photo attached. If removing the photo would leave the question unanswerable ("is it the tension?" + photo), it's `low_effort_question`. I apply this consistently — e.g. "Thread loops when turning corners, settings shown (photo)" is detailed because the settings are written out, while "Is it the tension? (photo only)" is low-effort.

Three specific cases that gave me pause, and what I decided:

1. **"Specific use case (oven mitts), no prior research shown"** — has a concrete project but no research, no machine, no constraint beyond the use case. Decided `low_effort_question`: a use case alone isn't enough to engage with; it reads as "how do I make oven mitts?"
2. **"Named machine, brief symptom, minimal detail"** — names the machine (a detailed-question signal) but the symptom is one thin phrase. Decided `low_effort_question`: a name without a real, described problem doesn't give an answerer enough to work with. The name alone is decorative.
3. **"Jersey stretch problem, general but some context (months sewing)"** — states skill level (some context) but the actual ask stays general. Decided `low_effort_question`: stating "I've sewn for a few months" is background, not a specific question; the boundary is whether the *question* is specific, not whether the asker added biography.

The throughline of all three: **context about the asker is not the same as specifics about the question.** That's the sharpest version of my boundary.

## 4. Data collection plan

**Source:** r/sewing weekly Simple Questions threads, collected manually from old.reddit.com. Threads sampled across multiple dates (Jan 2025, June 2026, Dec 2021, Nov 2021, and others) to avoid any single thread's quirks dominating.

> **Note on the `text` column:** the dataset stores a faithful *condensed summary* (gist) of each question post rather than verbatim Reddit text — e.g. "Corset neckline gapes when enlarging the bust" rather than the original full wording. This is disclosed in the README's AI usage / data section. The label reflects the original post's level of written detail, not the length of my summary.

**Target per label:** No fixed quota. The class balance is an artifact of the community — r/sewing askers are structurally thorough because community norms push them to give detail, so `detailed_question` naturally dominates. I aimed for at least 200 total and as much `low_effort_question` as the real threads produced.

**If a label is underrepresented after 200:** Options were (a) accept the real distribution if it's under the 70% ceiling, (b) pull additional threads to surface more low-effort posts, or (c) trim borderline `detailed` calls. I chose (a) — see distribution below.

**Final distribution (222 examples):**
- `detailed_question`: 180 (81.1%)
- `low_effort_question`: 42 (18.9%)

This clears the 200 minimum and sits well under the spec's 70%-majority red flag. The 18.9% minority is just under the "aim for 20%" guideline; I accepted it rather than fabricating or force-trimming, because 222 real examples at this ratio is a defensible dataset and chasing the exact percentage was diminishing returns.

## 5. Evaluation metrics

Accuracy alone is misleading here **because of the class imbalance**: a model that predicts `detailed_question` every single time would score ~81% accuracy while being useless at the actual job (catching low-effort questions). So I need metrics that expose per-class behavior:

- **Overall accuracy** — headline number, but interpreted against the 81% majority-class baseline, not against 50%.
- **Per-class precision, recall, and F1** — especially for `low_effort_question`, the minority class. Recall on `low_effort_question` is the metric I care most about: of all the genuinely low-effort posts, how many did the model catch? A model that never flags low-effort questions has near-zero recall there regardless of its accuracy.
- **Confusion matrix** — to see the *direction* of errors. The failure mode I expect is the model collapsing toward the majority class (predicting `detailed` on true `low_effort` posts), which would show up as a heavy off-diagonal cell.

## 6. Definition of success

This classifier would be genuinely useful as a community tool that auto-nudges askers ("consider adding your machine model / what you've tried") before they post a low-effort question. For that:

- **Minimum bar:** the fine-tuned model must beat the majority-class baseline (81% accuracy) *and* beat the zero-shot Groq baseline on overall accuracy. Beating 81% by predicting the majority class doesn't count — see below.
- **The metric that actually defines success:** `low_effort_question` **F1 ≥ 0.55**, with recall ≥ 0.50. If the model can't catch at least half of low-effort questions with reasonable precision, it can't do the one job that matters (the majority class takes care of itself). This is my own threshold, not a spec default: I set it at 0.55 because with only ~42 minority examples and ~6 in the test split, perfect minority performance is unrealistic, but catching more than half with decent precision would make the nudge feature worth shipping.

I will judge the project against these specific numbers at the end, not against "it works well."

## AI Tool Plan

There's no implementation code in this project, so AI assistance applies at three points:

**Label stress-testing.** Before annotating, I gave Claude my two definitions and the photo-plus-text edge case and had it surface where the boundary was thin. This is what produced the locked "photo never substitutes for written detail" rule and the "context about the asker ≠ specifics about the question" sharpening.

**Annotation assistance.** I used Claude to pre-label batches of collected posts against my definitions, then reviewed and corrected every label myself — particularly the borderline photo-plus-text and named-machine-thin-symptom cases, where my own judgment overrode the first-pass label. This is disclosed in the README AI usage section. Every row in the final CSV passed my own review.

**Failure analysis (planned, Milestone 6).** After fine-tuning I'll paste the misclassified test examples into Claude and ask it to identify systematic patterns (e.g. "misses low-effort posts that happen to name a machine"), then verify each pattern by re-reading the examples myself before writing it up.

---

### Open items before I lock this
- [ ] Confirm the Section 6 thresholds (0.55 F1 / 0.50 recall) are *my* call — adjust if I disagree after seeing the baseline.
- [ ] Swap the example posts in Section 2 for ones I personally like best from the dataset if these aren't my favorites.# planning.md — TakeMeter (AI201 Project 3)

## 1. Community

I chose **r/sewing**, specifically its recurring weekly **Simple Questions** threads.

I picked this community because I sew myself (Brother XM2701), so I can read the discourse the way a regular would and judge what actually counts as a useful question. I considered the more common "hot take vs. analysis" subs (r/nba, r/television) but rejected them: after reading real threads I found r/sewing isn't an opinion space at all. The Simple Questions thread is a help forum — people ask, people answer, people say thanks. That ruled out any quality axis based on *boldness of opinion*.

What makes the discourse varied enough to classify is the **substance of the question itself**. Some askers give a machine model, what they tried, measurements, and constraints; others post a photo with "help, it's broken" and almost no written context. That gap is real, it's visible, and it matters to the community — detailed questions get fast, accurate answers while low-effort ones get a string of "can you post more detail?" replies. That's the distinction my classifier measures.

## 2. Labels

This is a **2-label** task. I label only the **question posts** in each thread — answers and thank-you replies are dropped, since they aren't part of a question-quality task. The two labels are mutually exclusive and cover essentially all question posts (no "other" bucket needed).

### `detailed_question`
The written text alone gives enough to answer or meaningfully engage: a named machine model plus symptoms and what was tried, a named pattern or specific features, measurements, or a clear stated constraint (budget, fabric, deadline, skill level used as real context).

- *Example:* "Singer 1304 jams with a thread nest, bobbin numbered" — names the machine, the symptom, and a relevant detail.
- *Example:* "Janome HD3000 vs DC3050, current specs, prices, goals" — names both machines and states the deciding factors in writing.

### `low_effort_question`
The post leans on a photo with little written context, is vague ("it's broken, help"), or is a bare "how do I make this?" / "what's this?" with no constraints.

- *Example:* "What's this line by the buttons? (photo only)" — photo-reliant identification request with no written specifics.
- *Example:* "What brand should I buy? no constraints given" — no budget, no use case, no context.

## 3. Hard edge cases

The genuinely ambiguous case is the **photo-plus-some-text post**: an asker attaches a photo *and* writes a sentence or two. Does the writing carry the question, or is the photo doing the work?

**Decision rule (locked before annotating):** A photo never substitutes for written detail. I label on the written text only. If the text alone names specifics (machine, measurements, what was tried, a constraint), it's `detailed_question` even with a photo attached. If removing the photo would leave the question unanswerable ("is it the tension?" + photo), it's `low_effort_question`. I apply this consistently — e.g. "Thread loops when turning corners, settings shown (photo)" is detailed because the settings are written out, while "Is it the tension? (photo only)" is low-effort.

Three specific cases that gave me pause, and what I decided:

1. **"Specific use case (oven mitts), no prior research shown"** — has a concrete project but no research, no machine, no constraint beyond the use case. Decided `low_effort_question`: a use case alone isn't enough to engage with; it reads as "how do I make oven mitts?"
2. **"Named machine, brief symptom, minimal detail"** — names the machine (a detailed-question signal) but the symptom is one thin phrase. Decided `low_effort_question`: a name without a real, described problem doesn't give an answerer enough to work with. The name alone is decorative.
3. **"Jersey stretch problem, general but some context (months sewing)"** — states skill level (some context) but the actual ask stays general. Decided `low_effort_question`: stating "I've sewn for a few months" is background, not a specific question; the boundary is whether the *question* is specific, not whether the asker added biography.

The throughline of all three: **context about the asker is not the same as specifics about the question.** That's the sharpest version of my boundary.

## 4. Data collection plan

**Source:** r/sewing weekly Simple Questions threads, collected manually from old.reddit.com. Threads sampled across multiple dates (Jan 2025, June 2026, Dec 2021, Nov 2021, and others) to avoid any single thread's quirks dominating.

> **Note on the `text` column:** the dataset stores a faithful *condensed summary* (gist) of each question post rather than verbatim Reddit text — e.g. "Corset neckline gapes when enlarging the bust" rather than the original full wording. This is disclosed in the README's AI usage / data section. The label reflects the original post's level of written detail, not the length of my summary.

**Target per label:** No fixed quota. The class balance is an artifact of the community — r/sewing askers are structurally thorough because community norms push them to give detail, so `detailed_question` naturally dominates. I aimed for at least 200 total and as much `low_effort_question` as the real threads produced.

**If a label is underrepresented after 200:** Options were (a) accept the real distribution if it's under the 70% ceiling, (b) pull additional threads to surface more low-effort posts, or (c) trim borderline `detailed` calls. I chose (a) — see distribution below.

**Final distribution (222 examples):**
- `detailed_question`: 180 (81.1%)
- `low_effort_question`: 42 (18.9%)

This clears the 200 minimum and sits well under the spec's 70%-majority red flag. The 18.9% minority is just under the "aim for 20%" guideline; I accepted it rather than fabricating or force-trimming, because 222 real examples at this ratio is a defensible dataset and chasing the exact percentage was diminishing returns.

## 5. Evaluation metrics

Accuracy alone is misleading here **because of the class imbalance**: a model that predicts `detailed_question` every single time would score ~81% accuracy while being useless at the actual job (catching low-effort questions). So I need metrics that expose per-class behavior:

- **Overall accuracy** — headline number, but interpreted against the 81% majority-class baseline, not against 50%.
- **Per-class precision, recall, and F1** — especially for `low_effort_question`, the minority class. Recall on `low_effort_question` is the metric I care most about: of all the genuinely low-effort posts, how many did the model catch? A model that never flags low-effort questions has near-zero recall there regardless of its accuracy.
- **Confusion matrix** — to see the *direction* of errors. The failure mode I expect is the model collapsing toward the majority class (predicting `detailed` on true `low_effort` posts), which would show up as a heavy off-diagonal cell.

## 6. Definition of success

This classifier would be genuinely useful as a community tool that auto-nudges askers ("consider adding your machine model / what you've tried") before they post a low-effort question. For that:

- **Minimum bar:** the fine-tuned model must beat the majority-class baseline (81% accuracy) *and* beat the zero-shot Groq baseline on overall accuracy. Beating 81% by predicting the majority class doesn't count — see below.
- **The metric that actually defines success:** `low_effort_question` **F1 ≥ 0.55**, with recall ≥ 0.50. If the model can't catch at least half of low-effort questions with reasonable precision, it can't do the one job that matters (the majority class takes care of itself). This is my own threshold, not a spec default: I set it at 0.55 because with only ~42 minority examples and ~6 in the test split, perfect minority performance is unrealistic, but catching more than half with decent precision would make the nudge feature worth shipping.

I will judge the project against these specific numbers at the end, not against "it works well."

## AI Tool Plan

There's no implementation code in this project, so AI assistance applies at three points:

**Label stress-testing.** Before annotating, I gave Claude my two definitions and the photo-plus-text edge case and had it surface where the boundary was thin. This is what produced the locked "photo never substitutes for written detail" rule and the "context about the asker ≠ specifics about the question" sharpening.

**Annotation assistance.** I used Claude to pre-label batches of collected posts against my definitions, then reviewed and corrected every label myself — particularly the borderline photo-plus-text and named-machine-thin-symptom cases, where my own judgment overrode the first-pass label. This is disclosed in the README AI usage section. Every row in the final CSV passed my own review.

**Failure analysis (planned, Milestone 6).** After fine-tuning I'll paste the misclassified test examples into Claude and ask it to identify systematic patterns (e.g. "misses low-effort posts that happen to name a machine"), then verify each pattern by re-reading the examples myself before writing it up.

---

### Open items before I lock this
- [ ] Confirm the Section 6 thresholds (0.55 F1 / 0.50 recall) are *my* call — adjust if I disagree after seeing the baseline.
- [ ] Swap the example posts in Section 2 for ones I personally like best from the dataset if these aren't my favorites.