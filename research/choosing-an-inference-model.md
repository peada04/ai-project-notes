# Choosing an inference model, and why the fastest one lost

AIP runs a company's research through several inference passes — pull atomic observations
out of the sources, derive metrics from those observations, then reason over the whole
thing to produce inferences and recommended actions.  All of that runs locally on a DGX
Spark, so the choice of model is mine to make and worth actually testing.

In June I compared three: Qwen3.5-35B-A3B, which was the existing baseline,
NVIDIA-Nemotron-3-Nano-30B-A3B in NVFP4, and Qwen3.6-35B-A3B.  Ten financial services
accounts each, with the research step and the evaluation judge held constant so the model
was the only thing changing.  The judge is Gemma 4 scoring each run against that run's own
sources, which is what makes the quality numbers comparable even though the research text
differs between runs.

I'll put the main caveat up front because it matters for reading everything below: each
run did its own fresh web research on a different day, so this is a comparison of
pipelines rather than a controlled model A/B.  The judged quality scores hold up, because
each run is graded against what it actually retrieved.  The raw counts partly reflect
differences in the underlying research.

> **Revisited, September 2026.**  Later work on a related pipeline measured how much the
> retrieved corpus changes between runs, and it's a great deal more than I assumed when I
> wrote this up.  That prompted me to go back and check the recall comparison properly.
> One of the two recall findings holds and one doesn't, and I've corrected it below rather
> than leave it standing.  The conclusion about which model to use is unaffected, because
> it rests on the judged layers and on latency rather than on recall.  Details are in
> [what the arithmetic says about recall](#what-the-arithmetic-says-about-recall).

## The headline

| Per account, n=10 | Qwen3.5 | Nemotron | Qwen3.6 |
|---|---|---|---|
| Faithfulness of extraction (judge) | 0.836 | 0.888 | 0.862 |
| Validity of derivation (judge) | 0.646 | 0.852 | 0.846 |
| Reasoning quality (judge) | 0.851 | 0.534 | 0.875 |
| Observations extracted | 20.6 | 10.2 | 17.3 |
| Distinct signal types | 6.8 | 5.6 | 6.8 |
| Metrics derived from measured evidence | 56 | 8 | 52 |
| Inferences per account | 12.2 | 9.5 | 11.9 |
| Processing time after research | 1,110 s | 262 s | 606 s |
| Tokens per run | 56.0k | 38.6k | 58.5k |

Qwen3.6 was a straightforward upgrade over Qwen3.5.  Better on every judged layer, about
twice as fast, roughly the same token cost, and with recall that I originally read as
slightly lower but can't actually distinguish from the baseline's (see below), while
keeping the same breadth of signal types.  That part needed no interpretation.

Nemotron is the interesting one, because on two of the three judged layers it looks like
the winner.

## The derivation layer, where the numbers mislead

Every metric the pipeline derives is labelled by where its value came from: measured from
evidence, estimated, or recorded as absent because nothing was found.  The judge scores
whether the derivation was valid.

Nemotron scored 0.852 on validity against Qwen3.5's 0.646, which reads as a decisive win.
The value-basis breakdown explains what actually happened:

| Value basis | Qwen3.5 | Nemotron | Qwen3.6 |
|---|---|---|---|
| Measured from evidence | 56 | 8 | 52 |
| Estimated | 23 | 18 | 11 |
| Absent, nothing found | 21 | 34 | 22 |

Nemotron extracted half as many observations as the others, so when it came time to derive
a metric there was frequently nothing to derive it from, and it correctly recorded the
metric as absent.  A confident "not found" is easy to score as valid.  Only 13% of its
metrics came from measured evidence, against 56% for Qwen3.5.

So the validity score was rewarding conservatism, and the thing I actually care about —
metrics with a real number behind them — pointed the other way.  Qwen3.6 is the one that
does both: it derives about as much as Qwen3.5 did while lifting validity from 0.646 to
0.846, and it produces fewer hedged "estimated" values, which I read as it being more
willing to commit to measured or absent rather than splitting the difference.

## The fastest model was confidently wrong

Nemotron's extraction was genuinely good.  Its observations were the most faithful of the
three at 0.888, it ran about four times faster than Qwen3.5, and it used fewer tokens.  On
a precision-only extraction job I would use it.

Its reasoning quality came in at 0.534, against 0.851 and 0.875 for the two Qwen models.
Those inferences are the deliverable — they become the opportunities, risks and
recommended actions that go to an account team — so that gap is the whole decision.

What made it stick was the self-assessment.  The pipeline asks each model to rate its own
reasoning quality, and all three rated themselves at about 0.81: Nemotron 0.808, Qwen3.5
0.807.  The independent judge agreed with Qwen at 0.85 and disagreed with Nemotron at
0.53.  Both models were equally sure of themselves and only one was right, which is a
useful reminder that a model's own confidence carries no information about whether it
should be trusted.

Nemotron's per-account reasoning was also volatile, ranging from 0.20 to 0.84.  It reached
Qwen-level on the one account where the upstream research was richest, which fits the
pattern — it isn't that it reasons badly in general, it's that it had less to reason from
and didn't behave differently as a result.

## Recall is account-dependent, and the averages hide it

The per-account observation counts, with accounts numbered arbitrarily and in no
particular order:

| Account | Qwen3.5 | Nemotron | Qwen3.6 |
|---|---|---|---|
| 1 | 20 | 9 | 18 |
| 2 | 26 | 10 | 11 |
| 3 | 15 | 15 | 8 |
| 4 | 13 | 11 | 14 |
| 5 | 29 | 11 | 29 |
| 6 | 0 | 1 | 15 |
| 7 | 19 | 14 | 12 |
| 8 | 23 | 11 | 18 |
| 9 | 40 | 13 | 22 |
| 10 | 21 | 7 | 26 |

Read as cohort averages, this says Qwen3.6 gives up about 15% of the baseline's recall,
and that is how I originally reported it.  The rows say something less tidy.  It matches or
beats the baseline on four accounts, and on account 6 — where the baseline had extracted
nothing at all — it went from 0 to 15.  On account 9 it dropped from 40 to 22.

Account 6 is also a caveat against my own numbers, because a baseline of zero observations
drags down Qwen3.5's average and makes the comparison look slightly better for Qwen3.6
than it deserves.

### What the arithmetic says about recall

Having since learned how much the retrieval corpus varies run to run, I went back and
paired the differences by account rather than comparing the two averages:

| Comparison | Mean difference | 95% confidence interval |
|---|---|---|
| Qwen3.6 − Qwen3.5 | −3.3 observations | −10.1 to +3.5 |
| Nemotron − Qwen3.5 | −10.4 observations | −16.8 to −4.0 |

The per-account differences between the two Qwen models run from −18 to +15, so the
interval comfortably contains zero.  **The 15% recall gap I reported isn't a finding — it
is within the noise of the measurement**, and separating a difference that small from the
run-to-run variation would need something like 65 accounts rather than 10.  Nemotron's
recall deficit is a different matter: roughly 10 fewer observations per account, with the
interval nowhere near zero.

Two other things fall out of looking at it this way.  Averaging over accounts doesn't
rescue the comparison, because each model's runs happened on different days, so every
account in an arm shares that arm's retrieval draw.  Averaging removes variation between
accounts, not a shift common to all of them — more accounts would have made the estimate
more precise without making it any less biased.

And the obvious fix, replaying one frozen corpus through all three models, isn't available
for these runs.  The pipeline stores a SHA256 of the research text rather than the text, so
the inputs are gone.  A hash can confirm a document you already have; it can't give you one
back.  The lesson I've taken from that is to capture the research text before a comparison
rather than after, which is cheap to do and would have made all of this a settled question
instead of an open one.

## The orchestrator set the batch size, not the GPU

The Qwen3.6 batch died at account nine with a JavaScript heap allocation failure.  That
wasn't the model or the GPU: the serial driver in n8n holds every sub-workflow's output in
the parent execution, and nine accounts of large payloads exhausted the heap.  The lighter
Nemotron payloads never triggered it, which is why it only showed up on the last of the
three runs.  The practical fix is running large cohorts in chunks of about six, or raising
the Node heap limit.

I also saw roughly one run in ten fail transiently somewhere in the pipeline across these
batches, so any cohort run needs to budget for a re-run rather than assume a clean sweep.

## What I'd hold at arm's length

Beyond the pipeline-versus-A/B caveat at the top, and the recall correction above: n is
ten accounts in a single industry, so the means are directional, and because the arms ran
on different days the retrieval draw is confounded with the model rather than randomised
against it.  The Qwen3.5 baselines were collected over about ten days and
a few may use slightly earlier prompt versions.  And the judge's reasoning rubric was
tuned on Qwen output in the first place, which could plausibly disadvantage Nemotron on
style rather than substance — though the gap between its self-rating and the judge's score
suggests something real underneath.  For the same reason as the recall numbers, the judged
scores aren't entirely insulated from the draw either — richer evidence makes reasoning
easier, and Nemotron's own reasoning scores track how much it had to work with, from 0.20
on the account with a single observation to 0.84 on the richest.

Memory was also tight throughout.  Qwen3.6 in BF16 is around 90 GB and the judge another
26 GB, on a 128 GB machine, which is close enough to the edge that one run I marked as
stuck may have been contention at evaluation time rather than anything about the model.

## Where it ended up

Qwen3.6 became the model for the inference passes and still is.  Nemotron I'd keep in
reserve for a latency-bound job or as a fast, faithful extractor feeding a stronger model
for the synthesis step, which is a mixed-model setup I haven't tried.

The more useful conclusion was about where the remaining problem was.  Even on the best
model, about 22 metrics per account were still coming back as absent, and no amount of
model upgrade was going to find evidence that hadn't been retrieved.  That pointed at the
research step rather than the inference step, which is what led to
[swapping out the research engine](self-hosted-vs-paid-research.md) — and to a result I
was not expecting.

---

*Accounts are numbered arbitrarily within this write-up and the numbering does not
correspond to any other one.  Model names, counts and scores are as measured.*
