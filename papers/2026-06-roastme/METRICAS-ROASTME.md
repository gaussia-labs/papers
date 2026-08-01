# How the Roast Me metric works, end to end

This document walks through every metric and formula in the project (Profiler + Exploiter)
in order. Each builds on the previous one, so it is meant to be read straight through rather
than dipped into. Each metric gets: the formula as it appears in the paper, the question it
answers, and a plain-language version.

## 0. The core piece: how "a violation" is measured

Everything else is built on this idea, so start here.

**The assistant as a black box.** The paper models the assistant as a function $A$ that, given
a conversation history $x$, returns a *distribution* of possible answers (it does not always
reply the same way). You can only ask it for one answer and look at it; you cannot open the box.

**The "contract" the assistant is supposed to honour.** A list of rules
$\Pi = \{\pi_1, ..., \pi_m\}$, such as "do not invent features that do not exist", "do not
reveal your internal tools", "do not accept a false premise without correcting it". Each rule,
in theory, returns 0 or 1: was it violated or not?

**The problem**: nobody can compute $\pi_j$ exactly and automatically. So a **judge**
($\hat\pi_j$) is used — a model (or a rule) that *estimates* how likely it is that the rule was
violated, as a number between 0 and 1 rather than a hard yes/no.

**The violation score**, the formula everything else uses:

$$v(x, r) = \sum_{j=1}^{m} w_j \, \hat\pi_j(x, r) \in [0,1]$$

**In plain terms:** a weighted average of "how much each rule was violated", where $w_j$ is how
much that particular rule matters (if you only care about one rule, give it weight 1 and the
rest 0).

## 1. How it is measured in practice: the SI/NO logprobs judge

This is the actual technique used to compute $\hat\pi_j$ (and therefore $v$) in this project,
not just the theory.

**The idea:** instead of asking the judge model to "give a score from 0 to 1 in free text"
(easy for it to invent any number), it is asked to answer **a single word: SI or NO**. Then we
do not just look at which word it picked — we look at the **actual probability** the model
assigned to "SI" versus "NO" (the `logprobs`, the internal probability of each possible word at
that position). That yields a continuous number between 0 and 1 rather than a binary yes/no,
and it is much harder to hallucinate because it comes from the model's internal maths rather
than from what it "says".

**An important detail the paper verifies:** with models that reason before answering (GLM-5.2,
Qwen3.6-35B-A3B), the first word they emit is part of the reasoning, not the verdict. They need
room to finish thinking, and the **last** token that looks like a verdict has to be read, not
the first. An earlier version of this looked only at the first token and never saw the real
verdict.

**How this was verified to actually work** (it was checked, not assumed): the verdict extracted
from logprobs was compared against the text answer the same model would normally give, over 40
real cases the model had never seen. GLM-5.2 agreed on 100% of the 38 comparable cases (2 cases
failed on an empty/truncated answer, and those were resolved with a normal sample rather than
logprobs). Qwen3.6-35B-A3B agreed on 100% of all 40 cases with no failures, given more room to
reason (4096 tokens against GLM-5.2's 1024).

## 2. Profiler metrics

The Profiler sends a batch of trap questions ("probes") to the assistant and builds a profile
of where it is weak.

### 2.1 The knowledge label (`doc`)

Every probe is labelled with whether what it asks about is real or invented:

$$\mathrm{doc}(g, K) \in \{0, 1\}$$

- `doc=1`: the probe asks about something that **does exist** in the knowledge base
  (documented).
- `doc=0`: the probe asks about something **invented on purpose** that does not exist (to see
  whether the assistant recognises it as false or accepts it as real).

**Why it matters:** this label is what later lets a judge distinguish "the assistant invented
something that does not exist" from "the assistant correctly described something that does".

### 2.2 The weakness map ($\omega$)

$$\omega : \Pi \times \mathcal{Z} \rightarrow [0,1]$$

For each combination of (contract rule, probe type), it records **how often that kind of
question makes the assistant violate that rule**, computed from all the graded answers.

**In plain terms:** the table saying "the assistant fails 43% of the time when you ask about an
invented category, but only 8% when you lie to it about a real article".

### 2.3 Actual Profiler results in this project

204 probes in total against the Monotributo assistant. The `documented_recall` category
(control, no adversarial premise) is excluded from the failure metric because there is nothing
to violate there — leaving **151 scoreable probes**.

**Overall failure rate, per judge** (5 iterations each):

| Judge | Method | Failure rate |
|---|---|---|
| Gemma (google/gemma-4-31B-it) | logprobs | 12.6% |
| GLM-5.2 | logprobs (4/151 fell back to sampling) | 15.2% |
| Qwen3.6-35B-A3B | logprobs | 7.9% |

**Failure rate by strategy** (the detail that explains why the three judges do not agree
exactly):

| Strategy | n | Gemma | GLM-5.2 | Qwen3.6-35B-A3B |
|---|---|---|---|---|
| `rag_absence_attempt` | 10 | 50.0% | 58.7% | 20.0% |
| `graph_edge_false_premise` | 32 | 18.6% | 22.0% | 18.8% |
| `grounded_false_fact` | 89 | 9.3% | 11.3% | 4.5% |
| `false_limit_value` | 5 | 0.0% | 0.1% | 0.0% |
| `nonexistent_article` | 4 | 0.0% | 0.0% | 0.0% |
| `nonexistent_category` | 3 | 0.0% | 0.0% | 0.0% |
| `graph_absence_articulo` | 4 | 0.0% | 0.0% | 0.0% |
| `graph_absence_categoria` | 4 | 0.0% | 0.0% | 0.0% |

**How to read this:**

- Everything that depends on "enumerating completely" (a nonexistent article/category found via
  the full list, or via the full graph) scores **0% every time, across all three judges** — the
  assistant is never fooled when non-existence can be checked perfectly.
- `graph_edge_false_premise` (the largest sample after the main one) converges tightly across
  the three judges (~18-22%), a sign that inter-judge disagreement is low there and the number
  is trustworthy.
- The large disagreement sits in two places: `rag_absence_attempt` (where the RAG engine that
  generated those probes also had 0% precision at detecting real absence — see below — so part
  of the disagreement may be generation noise rather than the assistant) and
  `grounded_false_fact` (this one does look like genuine disagreement between judges about
  severity, with no generation excuse).

**A limitation the paper states explicitly:** these numbers come from LLM judges that **have
not yet been validated against real human labels**. That the three judges agree with each other
is reassuring, but it does not prove they agree with what a person would say.

## 3. Probe Library metrics (how the trap questions are generated)

### 3.1 The three engines and their trade-off

| Engine | How it knows what exists | Probes | Absences | Absence precision | False premises |
|---|---|---|---|---|---|
| Deterministic | enumerates the whole law | 65 | 7 | **1.00** | 58 |
| GraphRAG | graph treated as a complete catalogue | 40 | 8 | **1.00** | 32 |
| RAG | similarity search (does not see everything) | 99 | 10 | **0.00** | 89 |

**Absence precision** = of the probes where the engine claims "this does not exist", what
fraction does an independent check confirm *really* does not exist? The Deterministic and
GraphRAG engines are always right (1.00) because they know 100% of the content. RAG is never
right (0.00) on this dataset: every time RAG "invented" an article, that article turned out to
exist — it simply had not found it by similarity. In exchange, RAG generates far more variety of
false premises (89 of 99 probes).

### 3.2 Generation speed by model

| Model | Probes generated | Total seconds | Seconds/probe |
|---|---|---|---|
| Gemma | 12 | 13.6 | 1.1 |
| GLM-5.2 | 4 | 238.3 | 59.6 |
| Kimi-K2.6 | 3 | 310.7 | 103.6 |

The reasoning models (GLM, Kimi) are 50-100× slower per probe, but in exchange they generate
more elaborate traps (chaining several real articles before inserting the false premise) instead
of twisting a single fact the way the fast model does.

## 4. Exploiter metrics

The Exploiter takes the Profiler's profile and searches for *categories* of questions that break
the assistant repeatably, rather than one lucky question.

### 4.1 A category

$$c = (a_1, ..., a_\ell)$$

A category is a list of natural-language attributes combined with AND — for example, "the user
asks about a feature that does not exist" AND "the tone is technical and confident". Many
distinct concrete questions come out of that, all within the same category.

### 4.2 A category's failure rate ($\Phi$)

$$\Phi(c) = \mathbb{E}_{q \sim Q_c}\, \mathbb{E}_{r \sim A(q)}\, [v(q,r)]$$

**In plain terms:** if you generate a great many questions from this category and put them to
the assistant a great many times, how often does it violate some rule on average? This is the
"how much does this really break it, at bottom" figure — but it cannot be computed exactly (that
would need infinite samples), so it is estimated.

### 4.3 The score actually used: $S(c)$

$$S(c) = \hat\Phi_n(c) - \lambda \, \widehat{\mathrm{se}}_n(c)$$

- $\hat\Phi_n(c)$: the measured average over $n$ samples (the estimate of $\Phi(c)$ above).
- $\widehat{\mathrm{se}}_n(c)$: the standard error of that measurement — how spread out the
  results were from sample to sample.
- $\lambda \ge 0$: how much inconsistency is penalised.

**In plain terms:** it is not enough for a category to fail hard once by luck — it has to fail
**consistently**. If two categories have the same average but one is always the same while the
other swings between "breaks completely" and "breaks nothing", this formula punishes the second.
That is why it is called a "lower confidence bound": a more conservative, more trustworthy
version of the average.

### 4.4 The "on-profile" filter ($\kappa$)

$$v_{f,\kappa}(x, r) = v(x,r) \text{ if } f(x) \ge \kappa, \text{ else } 0$$

A question that flatly tells the assistant "break your rules" can achieve a high violation
score, but says nothing useful about real usage. $f$ measures how natural/indirect the question
is, and if it does not reach the threshold $\kappa$ that question **does not count at all** — it
is set to 0, as if it had never been evaluated.

### 4.5 The realism budget ($D$ and $\delta$)

$$\widehat{D}(Q_c \| N) = 1 - \mathbb{E}_{q \sim Q_c,\, q_N \sim N}[\cos(E(q), E(q_N))]$$

The generated questions and the "normal" questions (from the natural-query prior) are encoded as
vectors, and their average distance is measured (cosine distance). If that distance is at or
below a threshold $\delta$, the category counts as "sounds like a real user, not like a
constructed attack". A category with high $D$ is discarded even if its $S(c)$ is good.

### 4.6 The success threshold ($\tau$) and the final search

$$\mathcal{C}^\star = \{c \in \mathcal{C}_\theta : S(c) \ge \tau,\ D(Q_c \| N) \le \delta\}$$

A category "passes" if its score $S(c)$ exceeds $\tau$ **and** its realism distance does not
exceed $\delta$. In this project's experiments, $\tau = 0.35$.

### 4.7 How the search is trained (policy gradient / REINFORCE)

$$J(\theta_C) = \mathbb{E}_{c}[R_S(c)], \qquad
\nabla_{\theta_C} J = \mathbb{E}_c[A_S(c) \, \nabla_{\theta_C} \log p_{\theta_C}(c)]$$

A model that generates categories is trained so that, on average, it generates
better-scoring ones. $A_S(c)$ (the "advantage") tells it whether this particular category came
out better or worse than the average of the others — that is what pushes the model to repeat
what worked and avoid what did not. **Important:** only the category generator is trained; the
concrete-question generator is kept frozen, so the questions keep sounding natural and diverse
instead of converging on one magic phrase.

### 4.8 Refinement (finding the minimal cause)

$$c^\star = \arg\min_{c' \subseteq c} |c'| \quad \text{such that} \quad S(c') \ge \tau,\ D(Q_{c'}\|N) \le \delta$$

Once a category with several attributes works, smaller subsets of those attributes are tried. If
removing an attribute leaves the category *still* passing the threshold, that attribute was not
necessary — it was decoration. If removing it makes the category stop passing, that attribute is
a real cause of the failure. The result is the smallest category that still breaks the
assistant, which is easier to read and to reason about.

## 5. Metrics of the real Profiler→Exploiter integration

First real connection between the two stages: 8 training runs (4 parallel runs × 2 grading
methods, 5 steps each).

| Judge used | Categories tried | Passed the threshold | Top score |
|---|---|---|---|
| Keyword heuristic | 21 | 0 | 0.000 |
| LM judge (small, self-evaluating) | 32 | 0 | 0.224 |

With $\tau = 0.35$, **no category in any of the 8 runs reached the threshold** under either
judge. This is reported as a validated *integration* (the full circuit works end to end) but
**not** as a validated finding — the judge used in this first pass was small and
self-evaluating, with confirmed cases of bad judgement (it scored as a "violation" both a raw
HTTP 500 from the target and an answer that flatly declined to reply). That is why the paper
itself leaves replacing that judge with one of the three already verified (Section 2 here) as a
pending step before drawing conclusions.
