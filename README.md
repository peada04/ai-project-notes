# AI Project Notes

Notes on a set of side projects, most of them built in a home lab over the last year or
so.  I work in technology as a pre-sales solutions engineer rather than as a developer,
so a lot of what's here is me working something out rather than me knowing it in advance.
The write-ups exist because the interesting part usually turned out to be the measurement
that changed my mind, and that tends to get lost once the thing is working.

The thread running through most of them is that generating output with these tools is easy,
and generating output that is accurate and worth someone's time is not.  Several of these
notes are about holding something back until it was good enough, or about finding out that
something I had already shipped wasn't as good as I thought it was.

Two of the projects account for most of it:

**AIP Pulse** researches large companies — for me that means prospects and customers —
and delivers an HTML email and a short "Signal & Close" podcast to account teams every 30
days.  It uses public sources only.  It went into pseudo-production at the request of
leadership and currently delivers over 150 reports a month.

**AIP** is the more advanced version of the same idea: a hybrid-RAG solution that also
ingests sales and technical plays and correlates company research against them.  It isn't
in production.

Both run on a home lab built around Proxmox, Docker, a DGX Spark and a Synology NAS, with
n8n for orchestration, Postgres for state, vLLM serving Qwen 3.6, and Gemma 4 as an
evaluation judge.  The code for both lives in private repositories, because the data they
handle isn't mine to publish.  These notes are the part that can be shared, so companies are
described by role rather than named and anything that would identify one is left out.

## Start here

- [Keeping up with a customer is harder than it looks](research/what-these-systems-do.md) —
  what both systems are for, why handing the job to a model doesn't work on its own, and why
  there's a year of issues behind a report that looks simple.

## Models and engines

- [Swapping out the research engine](research/self-hosted-vs-paid-research.md) — a paid
  deep-research API returned four times as much text as a self-hosted engine and produced
  fewer grounded facts.  Why, and what it cost me to find out.
- [Four text-to-speech models before one was good enough to ship](research/choosing-a-tts-model.md)
  — the podcast stayed switched off through three candidates.  One was disqualified on its
  licence before quality came into it, and one sounded best in a short sample and fell apart
  over a full episode, which changed how I test.
- [Choosing an inference model](research/choosing-an-inference-model.md) — three models
  over ten accounts, where the newest was better on every judged layer and twice as fast,
  and the fastest one was ruled out for being confidently wrong about its own reasoning.

- The three-layer split *(planned)* — separating what was observed from what can be derived
  from it and what can be inferred on top, why collapsing those was the biggest source of
  wrong answers, and which parts of it a model actually grades.

## Retrieval and evidence

- Being selection-bound rather than retrieval-bound *(planned)* — 94 sources retrieved,
  25 used, 68 discarded by a single constant.  This one killed a lot of my own ideas.
- What happened when I raised that constant *(planned)* — the citation list doubled and
  the report body didn't change at all.
- Citations nothing in the report refers to *(planned)* — how a footer ends up selected
  without reference to the text above it.
- A corpus that is 70% different every run *(planned)* — and why the obvious explanation,
  that the news had simply moved on, turned out to be wrong.

## Entity resolution

- Two companies with the same name *(planned)* — source-side disambiguation, why a
  registry has to adjudicate rather than propose, and the alias my tool suggested that
  was the same company all along.

## Proofs of concept

- Extending the reports to more solution segments *(planned)* — a proof of concept that
  worked: one pipeline producing a distinct, credible report per segment rather than one
  per account, with none of a segment's vocabulary leaking into another's report, including
  in the case I picked deliberately because it should have been the hardest.  On hold
  rather than abandoned.  At full scope the retrieval cost doesn't fit the capacity I have
  unless a single retrieval can serve several segments at once, which is measured but not
  built, and going further is a bigger commitment than a proof of concept justifies on its
  own.

## Working notes

- Measuring production rather than staging *(planned)* — a rate computed across both read
  23.6% where production alone read 14.5%.
- Verifying a prompt change without re-running the pipeline *(planned)* — freeze the
  input, prove the baseline is byte-identical, then ablate one thing at a time.
- Checking that a new guard isn't vacuous *(planned)* — break the thing it guards and
  watch it fail, because a guard you've never seen fail and a guard that can't fail look
  the same.

## Earlier work

- [Finding the alert that shows up first](research/leading-indicator-alerts.md) — a 2020
  machine learning project to detect the alerts that reliably precede an IT incident, built
  in my spare time over months without knowing Python, pandas or TensorFlow when I started.
  The useful part was defining what "the first sign of a problem" meant in a way a database
  could check.

## Tools

Two pieces of the lab are general enough to be useful on their own and will be published
separately: an n8n community node for HashiCorp Vault (OIDC auth, dynamic database
credentials, PKI and transit), and a set of scripts that treat n8n workflows as code with
drift detection between staging and production.

## A note on the notes

These are written after the fact but from the original working documents, including the
numbers that turned out to be wrong.  Where I corrected something I've said so in place
rather than quietly fixing it, because the broken measurement is usually more instructive
than the fixed one.

I also go back and re-examine earlier findings when something new makes that worth doing —
a later measurement, a better way of looking at the data, or just noticing that I'd assumed
something I hadn't checked.  A write-up here isn't a verdict I'm defending.  If a conclusion
doesn't survive a second look, the note says so and says which part changed, because getting
it right eventually is the point rather than having been right the first time.
