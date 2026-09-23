# Swapping out the research engine: what the tests actually showed

One of the bigger decisions in building AIP and AIP Pulse was where the research
actually comes from.  Both solutions start the same way — given a company, go find out
what's happened recently — and for the first several months that step was a paid
deep-research API.  Eventually I replaced it with a self-hosted setup, and the testing
that led to that decision surprised me enough that it's worth writing down.

I should say up front what "self-hosted" does and doesn't mean here.  What moved in
house was the model doing the reading and the writing, running on a DGX Spark.  Web
search did not.  That's still Brave's API behind a SearxNG front end, on a paid plan,
because self-hosting a model is a weekend project and self-hosting a web index is not.
So the cost comparison below is about marginal cost per report, not free versus paid.

## What I expected

I assumed the self-hosted option would be a compromise.  Cheaper, obviously, and
probably good enough, but worse on the thing I cared about most, which is how much
grounded, specific fact ends up in the report.  The paid deep-research endpoint returns
a lot more text, and more text seemed like it ought to mean more findings.

## The first test

The AIP pipeline labels every metric it derives as measured, estimated, or
absent_inferred, depending on whether it found a real number in a real source.  That
gives a fairly direct way to ask whether better research actually helped.  I ran ten
financial services accounts through both engines, holding the inference model, the
extraction prompt, and the evaluation judge constant, so the only thing that changed was
where the research came from.

| Per account, n=10 | Self-hosted | Paid deep research |
|---|---|---|
| Research text | 9,409 chars | 38,091 chars |
| Observations extracted | 17.3 | 14.1 |
| Measured metrics | 4.9 | 4.1 |
| Absent / inferred metrics | 2.0 | 3.6 |
| Judge: faithfulness | 0.846 | 0.873 |
| Judge: reasoning quality | 0.872 | 0.917 |

Four times as much research text produced slightly fewer grounded metrics and noticeably
more unsupported inference.  That wasn't the result I was expecting, and it took me a
while to believe it.

It's also worth saying that after the first four accounts I thought the opposite.  The
paid engine looked like a clear win on recall at that point, but nearly all of that came
from one large bank where it happened to find a very rich seam of numbers.  Across the
full ten, the self-hosted engine came out ahead on six, and its wins were the big gaps
(twelve measured metrics to two on one account, eight to two on another) while most of
the paid engine's wins were narrow.  If I'd stopped at four accounts I'd have made the
opposite call and been quite confident about it.

The reason, once I looked at the actual output, is fairly mundane.  The paid engine is
built to produce something a person wants to read, so it returns long, flowing prose.
The extraction step wants the opposite — short, specific, source-anchored statements it
can pull a figure or a date or a vendor name out of.  On one account the two engines
produced almost the same number of observations, 22 against 21, but twelve measured
metrics against two.  The specifics weren't missing from the paid output.  They were
narrated.

Where the paid engine genuinely did better was synthesis.  Its faithfulness and
reasoning scores were consistently a little higher, and if the deliverable were an essay
rather than a structured panel of metrics, it probably would have won.

## The second test

AIP Pulse has a different bar.  Every objective in the report has to carry an evidence
quote that corroborates against a source the system actually retrieved, so I ran a
matched pair of large investment banks through both engines with the same prompt and the
same verification behind it.

Both came out at six verified objectives of six, with nothing unverified.  The
difference was in the supporting detail: the self-hosted engine retrieved 161 and 97
sources against the paid engine's 17 and 14, at no per-report cost against about a
dollar an account.  The paid engine was roughly three times faster, which is a real
advantage, but for a nightly batch of one to three accounts it isn't worth much.

The obvious objection is that both of those are large, heavily covered companies where
any engine does well, so parity there doesn't prove much.  I followed up on a private
company with much thinner coverage, which was the case I expected to break the
self-hosted arm, and it returned 87 sources with all six objectives verified.

## Reliability was a factor too

Separately from any of this, the paid endpoint had been returning
`finish_reason: "length"` fairly often, and when it did, the workflow treated the call as
failed and fell back to a cheaper, non-deep model that was more inclined to answer from
training data and back-fill citations.  Looking at a sample of thirty production runs,
58% used complete deep research, 33% truncated and fell back, and another 8% fell back
for other reasons.  So roughly two reports in five were being written off the weaker
model, which hurt quality in a way that wasn't obvious from the run status.

The thing I found useful there is that the obvious fix was a dead end.  Completion
tokens were 13k to 16k on every run against a limit of 20,000, so the limit was never
what was binding and raising it did nothing at all.  There was also no threshold that
separated the good runs from the truncated ones — one run truncated on fewer tokens than
another that completed on the same prompt.  It was variance on someone else's server.

What did help was noticing that the truncation only cut the tail.  The objectives and
the news sections were all above the cut, so a truncated response was usually about 95%
complete and worth keeping rather than throwing away.  Accepting those recovered most of
the 33% and took the grounded rate to around 90%.  It was still a workaround for
somebody else's intermittent failure, though, which is part of why replacing the engine
started to look better than continuing to patch around it.

## Something I got wrong along the way

The first version of that second comparison reported six of six verified links for both
engines, and I was pleased with it.  That number came from a counter that was itself
broken: the verification code trusted whatever URL the model wrote on the line, and on
the self-hosted runs those URLs were mostly invented.  The real figure against retrieved
sources was zero of six.

The underlying grounding was fine, which is why the engine comparison survived, but for
a few days I had a green metric that was really measuring the model's honesty about its
own citations.  Two habits came out of that.  One, a counter reading zero isn't evidence
of anything until I've made it read non-zero on purpose.  Two, when a number turns out to
be wrong, correct it at the top of the document rather than quietly editing it, because
the broken number usually teaches more than the fixed one.

## A couple of bugs that fell out of the comparison

Neither has anything to do with which engine is better, but both were sitting there
waiting for the right input.

A hash that had worked for months started failing on the paid engine's output, because
it was hashing text as `bytea` and that output contained backslashes, which Postgres
parses as escapes.  Converting the text properly fixed it.  And a ten-account batch ran
out of memory at account nine, which turned out to be n8n holding every sub-workflow's
output in the parent execution rather than anything to do with the GPU.  The practical
batch size was a property of the orchestrator, not the inference host.

## Where it ended up

The self-hosted engine went into production in July, and the reports have been running
on it since.  The external model is gone from the path, the external search API isn't and
probably won't be, and the failure mode I'm left with — an occasional run that retrieves
nothing — is at least one a guard can see and retry, which the old one wasn't.

The part I'd have argued against beforehand is that for a pipeline whose job is
extraction, the research step should be optimised for the parser rather than for a
reader.  Volume of text turned out not to be a good proxy for how much grounded fact
you actually end up with, and a targeted search feeding a local model did better on that
than a deep-research API did, on a quarter of the words.

---

*Accounts are described by role rather than named.  Model names, counts and scores are
as measured.*
