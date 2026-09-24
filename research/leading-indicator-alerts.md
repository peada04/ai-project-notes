# Finding the alert that shows up first

This one is from 2020, which makes it the oldest thing here (so far) and the only one I built
without any AI help at all.  I'm including it because it's a useful contrast with
everything else in this repo, and because I still think the underlying idea was a good one.

I should say up front that I can't explain much of it today.  I've reconstructed what
follows from the deck I presented at the time, and there are decisions in there I can no
longer tell you the reasoning behind.  What I do remember clearly is how it was built: I
didn't know Python, or pandas, or TensorFlow, or what feature engineering was.  I worked on
it in my spare time for months, mostly by trying something, getting it wrong, reading
enough to understand why, and trying again.  Two steps forward and one back, for a long
time, until there was something that worked.

## The problem

In enterprise IT operations, alerts arrive constantly and most don't matter on their own.
Event management systems group related alerts into clusters, and a cluster is what someone
eventually looks at and calls an incident.

The trouble is that the cluster forms *after* enough alerts have accumulated to make the
pattern visible, by which point the thing has usually started affecting service.  Operations
teams talk about wanting to "shift left" and catch the first sign instead of the confirmed
pattern, but that's an aspiration rather than something you can compute.

What I kept coming back to was that none of the event management and AIOps products I worked
with did anything about this.  They were all good at grouping alerts once enough had arrived,
and none of them tried to tell you which alert tended to arrive first.  It seemed like an
obvious thing to want, and I couldn't work out why it wasn't there.  That's most of why I
stuck with it — not a plan, just being unable to let go of something that looked like it
ought to exist.

## The idea

The part I'd still stand behind is the definition, because it turns that aspiration into
something a database can check.

A leading indicator is an alert signature that **consistently** occurs before the cluster
that contains it — its first event time at or before the cluster's creation time.

The word doing the work is "consistently".  An alert firing early once is coincidence; with
thousands of alerts, plenty will land early by luck.  An alert signature that lands early
across hundreds of separate incidents is telling you something about how the environment
actually behaves.  That makes it a frequency you can count, and once you can count it you
can score it.

## What got built

Everything came out of the event database these environments already have.  I don't recall
whether that was a considered constraint or just the only data I had access to, but it was
the right shape either way.

So the features are SQL over existing events, snapshots and clusters.  The results below use
three of them: how many clusters a signature led, what proportion of its own occurrences
those represent, and a composite score combining the two.

The model is a multi-layer perceptron in Keras, classifying a signature as leading indicator
or not.  Two hidden layers of **four and six neurons**, ReLU on both, sigmoid on the output,
no normalisation — all based on my notes from the time, certainly not my recollection.  It is
a very small network.  I trained it on a balanced subset of 300 signatures so it saw both
classes in comparable numbers, then evaluated over the full set at its real, very lopsided
balance.

## What it found

| | Alert instances | Signatures | Clusters | Leading indicators found |
|---|---|---|---|---|
| Set 1 | 35,981 | 6,154 | 6,735 | 40 identified, 1 of them wrong |
| Set 2 — unseen | 4,719 | 2,017 | 1,508 | 21 of the 22 present |
| Set 3 | 150,000 | 10,556 | 10,000 | not scored — a volume check |

Those are counts rather than scores, deliberately.  I recorded accuracy at the time — 99.85%
on the second set — and it's the least informative number available, because leading
indicators are rare.  Set 2 held 22 of them among 2,017 signatures, about one in a hundred,
so a model that labelled every signature "not a leading indicator" would score about 99% and
be useless.

Set 2 is the one that matters: data the model had never seen, 21 of 22 leading indicators
found, one thing flagged that shouldn't have been.  Set 1 is evaluation over the pool the
training sample was drawn from, so it says the pipeline works rather than that the model
generalises.  Set 3 was 150,000 alert instances and 1.7 million snapshots from a production
system, run to see whether the approach held up at that volume.  It did, and that's all it
was asked to show.

The lead time was around ten minutes on average.  In one dataset, 90% of the time an alert
fired it did so ten minutes before the cluster containing it.  Individual signatures varied
widely — some led by five minutes, some by seconds.

## What it was for

Ten minutes isn't long, but it's the difference between automation having a chance and not:

- create the incident cluster as soon as a high-confidence leading indicator arrives, instead
  of waiting for the pattern to build
- attach automated remediation to the leading indicator itself, so some incidents get
  resolved before they escalate
- report on which alerts most often precede wider incidents, which is worth knowing even if
  you automate none of it

## How it ended

It stayed a prototype.  By the end I had something that worked on three different datasets
and a list of next steps — validating the features with people who knew more than I did,
comparing against unsupervised approaches, testing whether a model trained on one customer's
data transferred to another's, and working out whether a simple scoring formula could get
close without the model at all.

I think it could have become a real capability in the product, and the one customer I talked
to about it was definitely interested in helping me test.  None of that ended up happening,
partly due to a startup's shifting priorities.

## Looking back

The contrast with the projects elsewhere in this repo is the reason I've included it.  That
work took months before I had any results, and most of those months were focused on learning
things well enough to write the next twenty lines.  The same person building the same thing
now would zip into plan mode and have a prototype in hours instead of months.  How things
have changed!

What hasn't changed is where the value sat.  Almost all of it was in the definition and in
building features out of data that already existed.  The model was four neurons and six
neurons — small enough that it can't have been doing anything clever, and it didn't need to.

I've done a version of this at every place I've worked: find something that looks like it
should exist, that nobody is working on, and pick at it in the evenings and weekends until it
does something.  They're side projects in the real sense — not assigned, not on a roadmap —
and they've usually ended up being useful to the employer in one way or another.  Regardless
of that, to me, they've just been extremely rewarding and fun.
