 # TakeMeter — r/television Post Classifier

A text classifier that sorts r/television posts into three categories: recommendation, opinion, and news. Compares a fine-tuned DistilBERT model against a zero-shot Groq (llama-3.3-70b) baseline.

## Community

**Subreddit:** r/television

**Description:**
I chose r/television, a large active community where people discuss, debate, and discover TV shows. My labels are Recommendation (users asking what shows to watch), Opinion (users stating their own take on a show and inviting debate), and News/Updates (sourced factual information about shows like renewals, cancellations, and casting announcements). These distinctions matter because a user looking for a show recommendation is in a different mindset than someone wanting to debate a hot take or someone checking for factual show updates — they are looking for completely different things from the community.


## Labels
**General rule for mixed posts:** When a post contains both a fact and an opinion (e.g. "Season 2 got confirmed but I think it peaked"), the dominant intent wins. If the point of the post is the take, label it Opinion even if it mentions a news fact.

### Recommendation
**Definition:** Users simply asking what shows they should watch.
**Clear example 1:** "What are you watching and what do you recommend? (Week of June 19, 2026)"
**Clear example 2:** "Should I watch love island?"
**Hard case:** "My TV Show List" / ranked lists with reasons — gives ranked opinions AND watch suggestions. Rule: if the post is giving or asking for watch suggestions (even with opinions attached), it's Recommendation. A ranked list that tells you "why you should watch the top 5" is Recommendation.

### Opinion
**Definition:** Users stating their own take on a show and inviting debate.
**Clear example 1:** "Don't Understand the hype around Attack on Titan?"
**Clear example 2:** "After rewatching Jentry Chau vs. the Underworld, I still think it deserved more room to grow"
**Hard case:** "What are the best love stories in TV history?" — sits between Opinion and Recommendation. Rule: if asking people to evaluate shows they've seen, it's Opinion.

### News
**Definition:** Sourced facts about shows — renewals, cancellations, casting announcements, premiere dates. Does not include fan-made statistics or arguments presented as facts.
**Clear example 1:** "Jimmy Kimmel taps Rosie O'Donnell as rotating guest host during two-month hiatus"
**Clear example 2:** "'Devil May Cry' Renewed for Third and Final Season at Netflix"
**Hard case:** Unsourced casting/renewal speculation (e.g. "did anyone else hear X is casting for season 3?"). Rule: News must be sourced/reported facts. If the post is asking whether others heard something or speculating without a source, it's Opinion, not News.
---

## Dataset

209 labeled posts collected manually from r/television. Label distribution:
- opinion: 80
- news: 69
- recommendation: 60

## Data Collection & Labeling

I collected 209 posts manually from r/television by browsing the Hot, New, and Top feeds and copying post titles and short excerpts into a CSV. I used manual collection rather than the Reddit API (PRAW) because Reddit's current Responsible Builder Policy requires approval and prohibits using Reddit data to train ML models. I labeled each post myself using my planning.md definitions, applying my hard-case rules for ambiguous posts. I did not use an LLM to pre-label.

Label distribution: opinion 80, news 69, recommendation 60. No single label exceeds 70% of the dataset.

Three difficult-to-label examples and my decisions are documented in planning.md (Hard Edge Cases section): a ranked show list with reasons (ruled recommendation), an unsourced casting rumor (ruled opinion not news), and "best love stories in TV history" (ruled opinion).

## Fine-Tuning Approach

I fine-tuned `distilbert-base-uncased` with a 3-class classification head on a 70/15/15 train/validation/test split (146/31/32 examples), stratified to preserve label balance across splits.

Training setup: 3 epochs, learning rate 2e-5, batch size 16, weight decay 0.01, max sequence length 256. I kept the notebook's default hyperparameters because they are tuned for datasets of 100–500 examples, and mine (209) falls in that range. I specifically did not increase the number of epochs beyond 3, since more epochs risk overfitting on a dataset this small — with only 146 training examples, additional passes would memorize the training set rather than generalize.

The input text combined each post's title and body into a single field, since many posts are title-only and the title carries most of the classification signal.

## Baseline

The baseline is a zero-shot classifier using Groq's `llama-3.3-70b-versatile`. I wrote a system prompt that names the community (r/television), defines each of my three labels in plain language, gives one example post per label, and instructs the model to output only the label name. Each of the 32 test posts was sent to the model individually with temperature 0 for deterministic output. The notebook parsed each response by matching it against my valid label strings. All 32 responses were parseable (0 unparseable).

## Evaluation Report

The fine-tuned DistilBERT model was compared against a zero-shot Groq (llama-3.3-70b) baseline on the same 32-post test set.

### Overall Accuracy

| Model | Accuracy |
|-------|----------|
| Zero-shot baseline (Groq llama-3.3-70b) | 93.8% |
| Fine-tuned DistilBERT | 75.0% |

The baseline outperformed the fine-tuned model by 18.8 percentage points.

### Per-Class Metrics — Fine-Tuned DistilBERT

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|-----|---------|
| recommendation | 1.00 | 0.11 | 0.20 | 9 |
| opinion | 0.60 | 1.00 | 0.75 | 12 |
| news | 1.00 | 1.00 | 1.00 | 11 |

### Per-Class Metrics — Zero-Shot Baseline (Groq)

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|-----|---------|
| recommendation | 0.89 | 0.89 | 0.89 | 9 |
| opinion | 0.92 | 0.92 | 0.92 | 12 |
| news | 1.00 | 1.00 | 1.00 | 11 |

### Confusion Matrix — Fine-Tuned DistilBERT (Test Set)

| True ↓ / Pred → | recommendation | opinion | news |
|-----------------|----------------|---------|------|
| **recommendation** | 1 | 8 | 0 |
| **opinion** | 0 | 12 | 0 |
| **news** | 0 | 0 | 11 |

The model predicted opinion for 8 of 9 recommendation posts — a complete collapse of the recommendation class. News is perfect for both models.

### Did the Model Meet the Success Criteria?

My planning.md defined success as overall accuracy ≥ 75% and a minimum F1 of 65% on every class. The fine-tuned model exactly hit the accuracy target (75.0%) but badly failed the per-class F1 floor — recommendation scored only 0.20 F1, far below the 0.65 threshold. With a recall of just 0.11, the model correctly identified only 1 of 9 recommendation posts, predicting opinion for the rest. By my own criteria the model is not deployable, specifically because it almost entirely fails to detect recommendation posts.

## Failure Analysis

The fine-tuned model made 8 errors on the 32-post test set, and **every single one was the same directional error: recommendation predicted as opinion.** The model correctly identified only 1 of 9 recommendation posts (recall 0.11), dumping the other 8 into opinion. All errors had low confidence (0.35–0.38), meaning the model was guessing near the recommendation/opinion boundary rather than confidently wrong.

**Example 1 — "Suggestions? So far I've finished: Skins (UK), Lost, South Park, Heroes, Merlin, Sherlock, and Alphas."**
True: recommendation | Predicted: opinion (0.35)
The user lists shows they've already watched. This is the same surface pattern as an opinion post, where users name shows they've seen before sharing a take. The model learned "naming watched shows = opinion," and that signal overpowered the actual request for suggestions. This is a data/boundary problem, not a labeling error — the post is correctly labeled, but recommendation posts in r/television frequently include watch history as context, which structurally overlaps with opinion language. Fix: more recommendation training examples that include watch-history lists.

**Example 2 — "Any good western tv suggestions? I'm currently watching Westworld but I would also like something a little less futuristic."**
True: recommendation | Predicted: opinion (0.37)
Same failure type as Example 1. The phrase "I'm currently watching Westworld" names a specific show, which reads as opinion language, even though the post's actual intent is asking for suggestions. The fact that all 8 errors share this exact pattern shows the recommendation→opinion confusion is fully systematic, driven by show-name mentions, not random noise. Fix: more recommendation examples where the user names shows they're watching.

**Example 3 — "Need a new series to watch, could anyone suggest one? I recently finished re-watching Breaking Bad and im looking for a new series..."**
True: recommendation | Predicted: opinion (0.38)
Again the same pattern — the user names a show they finished (Breaking Bad) while explicitly asking for a suggestion. Even with the direct request "could anyone suggest one," the show-name mention pulled the prediction toward opinion. This confirms the model is keying almost entirely on whether specific shows are named, rather than on whether the post contains a request. Fix: tighter, more diverse recommendation examples, or a larger training set so the model can learn the request signal instead of the show-name shortcut.

## Sample Classifications

Example posts run through the fine-tuned DistilBERT model, showing predicted label and confidence.

| Post | Predicted | Confidence | Correct? |
|------|-----------|------------|----------|
| "'Devil May Cry' Renewed for Third and Final Season at Netflix" | news | 0.38 | ✅ |
| "Suggestions? So far I've finished: Skins, Lost, South Park..." | opinion | 0.35 | ❌ (true: recommendation) |
| "Any good western tv suggestions? I'm currently watching Westworld..." | opinion | 0.37 | ❌ (true: recommendation) |
| "Need a new series to watch, could anyone suggest one?..." | opinion | 0.38 | ❌ (true: recommendation) |
| "What are some shows you guys suggest?..." | opinion | 0.36 | ❌ (true: recommendation) |

**Why the correct prediction is reasonable:** "'Devil May Cry' Renewed for Third and Final Season at Netflix" is correctly predicted as news with the model's confidence concentrated on that class. The post contains the distinctive vocabulary the model learned for news — a show name in quotes, a network name (Netflix), and a factual event (a renewal). These structural signals are consistent across news posts and absent from opinion and recommendation posts, which is why news was the model's strongest category (perfect F1 of 1.00 this run).

**What the confidence scores reveal:** Every misclassification had low confidence (0.35–0.38), meaning the model was uncertain and essentially guessing near the recommendation/opinion boundary. It was never confidently wrong — the errors cluster exactly where the label boundary is genuinely fuzzy.

## Reflection: What the Model Captured vs. What I Intended

I intended my three labels to capture user *intent*: is the person asking for something to watch (recommendation), sharing a take and inviting debate (opinion), or reporting a sourced fact (news)? The model captured this well for news but only partially for recommendation and opinion.

**What it captured well:** News was the model's strongest class (F1 0.90). News posts contain distinctive vocabulary — "renewed," "cancelled," "premiere date," show names in quotes, network names — that is structurally distinct from the other two classes. The model learned this vocabulary cleanly and rarely confused news with anything else.

**What it overfit to:** The model learned a surface shortcut — "if a post names specific shows the user has watched, it's an opinion." Most of my opinion posts do name shows the user has seen, so the model latched onto that pattern. But this is a surface feature, not the intent I meant to capture. It overfit to *show-name mentions* instead of learning to detect *the request* that defines a recommendation.

**What it missed:** It failed to learn the intent behind recommendation posts. My definition was based on the user asking what to watch, but recommendation posts in r/television almost always include the user's watch history as context ("I've finished Lost, Skins, Sherlock — any suggestions?"). That watch-history language looks identical to opinion language, so the model couldn't separate "naming shows because I want a take" from "naming shows because I want a suggestion." The recommendation/opinion boundary is genuinely fuzzy at the surface level, and DistilBERT with ~200 examples couldn't learn the deeper intent signal that distinguishes them.

## Spec Reflection

**One way the spec helped:** Writing planning.md first forced me to define my three labels precisely, with clear examples and hard-case rules, before I collected any data. This made annotation much faster and more consistent — when I hit an ambiguous post during collection, I already had a rule for it (e.g., "if the post names shows the user has seen and asks for evaluation, it's opinion; if it asks what to watch next, it's recommendation"). Without those rules written down in advance, I would have labeled inconsistently across 200+ posts.

**One way the implementation diverged:** My data collection plan originally specified using PRAW (the Reddit API) to pull posts. When I went to set it up, Reddit's current Responsible Builder Policy requires approval and explicitly prohibits using Reddit data to train ML models, so I switched to manual collection — browsing r/television and copying posts into a CSV by hand. My target label balance also shifted: I planned for 70 per label but ended with 80 opinion / 69 news / 60 recommendation, because recommendation posts were genuinely less common in the feeds I browsed. I kept the mild imbalance since my evaluation plan already used per-class F1 to account for it.

## AI Usage

**Instance 1 — Label stress-testing:** I asked an AI tool to generate 10 boundary posts that sat on the cusp between two of my categories. I classified each one myself using my label definitions, then compared my answers to find where my definitions were ambiguous. This surfaced two real problems: a ranked list with "why you should watch" reasons (which I had to rule as recommendation, not opinion) and an unsourced casting-rumor post (which I ruled as opinion, not news, because it wasn't sourced). Based on this, I tightened my label definitions and added two explicit hard-case rules to planning.md before annotating my full dataset.

**Instance 2 — Failure analysis:** After running my fine-tuned model, I gave the AI my list of misclassified test posts and asked it to identify patterns. It surfaced that 5 of my 7 errors were the same directional error — recommendation predicted as opinion — and that all errors had low confidence (0.34–0.40). I verified this myself by re-reading the actual posts, and confirmed the pattern: every misclassified recommendation post named specific shows the user had watched, which the model read as opinion language. I used this verified pattern as the basis for my failure analysis rather than accepting the AI's summary at face value.