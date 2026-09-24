# Four text-to-speech models before one was good enough to ship

Each AIP Pulse report goes out with a short podcast called Signal & Close — three
speakers talking through what changed at a company over the last month.  The script is
written by a local model and read by a local text-to-speech model, and the whole thing
runs on hardware in my home lab.

The rule I set myself was that the podcast doesn't ship unless it's actually worth
listening to.  A report nobody reads is a waste of the reader's time; an audio briefing
that sounds wrong is worse, because it undermines the report it arrives with.  So the
podcast stayed switched off for a while, and I went through four models before one cleared
the bar.

None of them failed for the reason I expected.

## What the four were, and what stopped each one

| Model | Ran on | Why it didn't ship |
|---|---|---|
| XTTS v2 | CPU container | Licence — non-commercial only |
| Kokoro-82M | CPU container | Shipped, but flat and thin across three speakers |
| Dia-1.6B | GPU | Artifacts and repetition over a full episode |
| Chatterbox 0.5B | GPU | Nothing — this is the one in production |

### The one that was disqualified before quality came into it

XTTS v2 sounded decent, not great.  It's released under a licence that doesn't permit commercial use,
and this ends up attached to a report that goes to people at work.  So it was shut down automatically ona reading of the licence rather than a listening test, which is not where I expected the first elimination to come from.  I'd been thinking about this as an audio quality problem
and it opened as a licensing problem.

I keep the licence column in the table for that reason.  For anything self-hosted it's
worth establishing before you get attached to how something sounds.

### The one that shipped and wasn't good enough

Kokoro-82M is Apache-licensed, small enough to run on CPU, and genuinely decent.  It was
the first version anyone heard, but more as an example and proof that it could be done in my environment.  

Across three speakers having a conversation, it was flat and un-natural.  Each voice was clear and none
of them was doing anything — no variation in emphasis, everyone reading at the same pace
regardless of what they were saying.  Fine for one narrator reading a summary, thin for
something framed as a discussion between colleagues.  That's a hard thing to argue about
because there's no number attached to it, and I sat with it for a while before accepting
that "clear but flat" wasn't going to be good enough for something I was asking people to
spend five minutes on.

It's still deployed, but not being used.  It wasn't a bad model, just not good enough for production.  

### The one that taught me how to test

Dia-1.6B was the interesting failure, and it's the reason the rest of this went better.

In a short sample it was clearly the best thing I'd heard.  Expressive, varied, the sort of
read that sounded like a person rather than a system.  It was so promising, but over a full episode it fell apart.  Across twelve turns it produced audible artifacts and
started repeating itself, and the specific problem was that it didn't reliably stop — it
would reach the end of a line and keep generating.  Since the quality was good when it worked, I had my coding agent tune various parameters over a lot of different runs, and the agent thought there were improvements, but listening showed I never could get a clean episode.  It probably works great in a different environment, just not mine. 

The useful part is what it did to the acceptance test.  Every TTS demo you'll ever see is a
short clip, which is exactly the length at which this class of failure is invisible.  So
the gate became: generate the *same complete episode* through the incumbent and the
candidate, actually listen to all twelve turns of both, with a fixed seed so the comparison is
reproducible.  That's slower and more tedious than comparing samples, and it's the only
version of the test that would have caught what Dia did.

The check is written down in the deployment notes as "the check Dia failed", which is
probably the most useful thing that model contributed.

### The one that shipped

Chatterbox 0.5B cleared the full-episode test — twelve turns, no artifacts, stopping cleanly
every time — and it's noticeably more expressive across multiple speakers than the model it
replaced.  It's been the production voice since July, and I've gotten great feedback on the podcast since implementing it.

The three speakers are the model's own built-in voices, which is worth saying because people
assume otherwise when a briefing sounds like a conversation.  Nobody's voice has been
cloned.

## How it was rolled out

Worth recording because it's the part I'd repeat.  The new model was added alongside the old
one rather than replacing it — a separate service on a separate port, with the existing
configuration untouched, so nothing changed for anyone until a single field was repointed.

Two consequences.  Rollback is that one field, with nothing to rebuild or redeploy.  And the
switch was scoped to the podcast's own configuration — a single entry that covers all three of
its speakers — so the two other configurations, which use the same service for single-narrator
audio elsewhere, kept pointing at the old model and were never at risk.  The evaluation ran
against real output on the real path, and the blast radius was one setting.

## What I took from it

The thing I'd tell anyone choosing a model for this: **test at the length you intend to
ship at.** Some candidates might be fine in a short sample but fall apart at full length. 

And the second thing, which is more about the rule than the models: it is genuinely
uncomfortable to hold something back when it mostly works, because "mostly works" is
visible progress and holding it back looks like nothing happening.  The podcast was off for
weeks while this played out.  I think that was right.  The version that shipped is one I'm
happy to have land in someone's inbox, and the two earlier versions weren't, and nobody
would have told me so — they'd just have stopped listening.
