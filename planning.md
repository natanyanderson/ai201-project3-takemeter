# TakeMeter — planning.md

## Community

**Subreddit:** r/television

**Description:**
I chose r/television, a large active community where people discuss, debate, and discover TV shows. My labels are Recommendation (users asking what shows to watch), Opinion (users stating their own take on a show and inviting debate), and News/Updates (sourced factual information about shows like renewals, cancellations, and casting announcements). These distinctions matter because a user looking for a show recommendation is in a different mindset than someone wanting to debate a hot take or someone checking for factual show updates — they are looking for completely different things from the community.

---

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

### News/Updates
**Definition:** Sourced facts about shows — renewals, cancellations, casting announcements, premiere dates. Does not include fan-made statistics or arguments presented as facts.
**Clear example 1:** "Jimmy Kimmel taps Rosie O'Donnell as rotating guest host during two-month hiatus"
**Clear example 2:** "'Devil May Cry' Renewed for Third and Final Season at Netflix"
**Hard case:** Unsourced casting/renewal speculation (e.g. "did anyone else hear X is casting for season 3?"). Rule: News must be sourced/reported facts. If the post is asking whether others heard something or speculating without a source, it's Opinion, not News.
---

## Data Collection Plan

**Source:** r/television — browsing Hot, New, and Top (This Month) feeds manually
**Target:** 210 total examples, minimum 70 per label
**Format:** CSV with columns: post_title, post_text, label
**Collection method:** Manual — browsing r/television and copying post titles and short excerpts directly into the CSV. Used manual collection instead of PRAW due to Reddit's current API restrictions on ML training use cases.
**If a label is underrepresented:** Will search r/television specifically using terms like "renewed," "cancelled," "casting" for News/Updates, or filter by post flair if available, to reach the 70 minimum per label.

---

## Model Plan

**Baseline:** [fill in when you get to the notebook]
**Fine-tuned model:** [fill in when you get to the notebook]
**Evaluation approach:** [fill in when you get to the notebook]

## Evaluation Metrics

Accuracy alone is not enough because my dataset is imbalanced — it is skewed toward Opinion and News, with fewer Recommendation posts. A model could score high accuracy just by predicting the two largest classes well while ignoring Recommendation entirely, and accuracy would hide that failure.

Instead I will use per-class precision, recall, and F1 score alongside overall accuracy:
- **Precision** tells me, when the model predicts a label, how often it is correct (catches false alarms).
- **Recall** tells me, out of all the actual posts in a label, how many the model correctly caught (catches misses).
- **F1 score** combines precision and recall into one number per label.

Looking at these per label rather than averaged reveals whether the model handles all three labels well or just the two largest ones. This matters most for the Recommendation class, which has the fewest examples and is most at risk of being ignored.

## Definition of Success

I would consider the classifier good enough for deployment if it reaches:
- **Overall accuracy of at least 75%** across all three labels.
- **A minimum F1 score of 65% on every class**, including the weakest (Recommendation).

The per-class floor matters more than the overall number. A model could hit 75% accuracy while almost completely failing on Recommendation — the per-class F1 floor prevents that by requiring the model to perform reasonably on all three post types, not just the two largest. Hitting both targets would mean the tool can reliably sort posts into all three categories well enough to be useful in a real community setting, rather than just defaulting to the most common labels.

## AI Tool Plan

This project has no code to generate, so AI tools help in three specific places:

**Label stress-testing:** I will give an LLM my three label definitions and my identified hard cases, and ask it to generate 5–10 posts that sit at the boundary between two labels. If I can't classify the generated posts cleanly using my definitions, that signals my definitions need tightening — I'll revise them before relying on the dataset.

**Annotation assistance:** I chose NOT to use an LLM to pre-label my data. I hand-labeled all 210 posts myself so I would stay close to the data and apply my own annotation rules consistently, especially on the hard cases. This was a deliberate decision to keep the labeling quality high rather than reviewing machine-generated labels.

**Failure analysis:** After the model runs, I will take the list of misclassified posts and give them to an LLM, asking it to identify patterns in the errors. Specifically I will look for confusion between labels (for example, Opinion vs Recommendation) and whether the model fails on the same hard cases I flagged during planning. I will verify any patterns the LLM suggests myself by checking the actual posts rather than trusting its summary.
