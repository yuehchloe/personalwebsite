---
title: "SignNow: Real-Time Audio to ASL Translation"
date: 2026-01-15
summary: "A three-stage speech-to-ASL pipeline that renders output as filmed human video. 1st place, Health Track, UMich Ross Tech Innovation Jam."
tags: ["NLP", "Whisper", "FastAPI", "Python", "Accessibility"]
---

**Role:** ASL translation layer. User research, video corpus, gloss mapping and
grammar rules, prototype flows.
**Team:** 5 people
**Timeline:** Sep 2025 to Jan 2026
**Award:** 1st place, Health Track, UMich Ross Tech Innovation Jam ($1,500)
**Code:** [github.com/SammiWang0516/SignNowPrototype](https://github.com/SammiWang0516/SignNowPrototype)

---

## The problem

Deaf patients in emergency care wait for interpreters during the window when
decisions get made. Emergency calls involving language barriers require
[33 to 43 percent longer dispatch times](https://pubmed.ncbi.nlm.nih.gov/23952940/).
About [11 million Deaf and hard-of-hearing Americans](https://help.nationaldeafcenter.org/article/51-how-many-deaf-people-live-in-the-united-states)
use a healthcare system that routes around them by default.

The existing workarounds each fail somewhere. Written notes assume English
fluency, which does not hold when ASL is the first language. Video remote
interpreting needs setup time and a stable connection, neither of which is
reliable in an urgent room. Nothing on a phone went from spoken audio to ASL
without a third party in the loop.

Speech recognition and text generation were not the hard part. Both are solved
by models you can call. What no off-the-shelf component supplies is the middle:
which signs the system can express, how English maps onto them, and what counts
as a correct answer when the two grammars diverge. That layer was my work.

## The solution

### What the research decided

I ran interviews with Deaf community leaders at ThinkSelf, alongside sessions
with U-M Health providers. The finding that shaped the build was not about
speed. Participants rejected AI-animated signing avatars as untrustworthy rather
than as low quality. In a room where the message is a diagnosis or a consent
question, a rendering that reads as a robot proxy fails even when it is
linguistically correct.

Pose estimation onto an animated figure is the cheaper path. It generalizes to
any gloss token and needs no filming. Choosing human video instead closed off
that generality and set three constraints on everything downstream:

- The system can only express signs that exist as clips, so vocabulary becomes a
  design decision rather than a model property.
- Matching has to handle multi-word signs, since a filmed clip for BLOOD
  PRESSURE is one sign and not two.
- Unmapped words need a defined behavior, because a closed dictionary will be
  hit.

The rest of the translation layer is the answer to those three.

### The corpus

I filmed the corpus, about 130 signs scoped to clinical encounters. The
vocabulary came from two sources: frequency in medical transcripts, and common
daily words identified by working through likely scenarios. Clips are hosted
separately and referenced by URL from the mapping file.

### Gloss and grammar

ASL is not signed English. Word order, dropped copulas, and topic-comment
structure all differ, so substituting word for word produces something a signer
cannot read. Getting the middle stage right meant defining what correct output
looks like, then encoding it.

The rules I wrote cover copula removal, negation placement, W-question
reordering, and semantic normalization, applied on top of 97 phrase patterns for
common clinic, airport, and school exchanges. These run in the offline path and
also define the target the prompted model is asked to produce.

Mapping gloss to video uses longest-phrase-first matching so multi-word signs
resolve as single units, falling back to single-word lookup. Words outside the
map fall back to fingerspelling, letter by letter, which is what a human signer
does with a name or an unfamiliar term.

### The rest of the pipeline

Three stages sit behind a FastAPI service, each exposed as its own endpoint so
it could be tested in isolation.

**Audio to English.** Whisper base, running locally rather than through an API.
Latency was a factor, but the main reason was privacy: this audio carries
personal medical information that should not leave the device to reach a
third-party server.

**English to gloss.** The primary path sends the transcript to Perplexity's
`sonar` model under a system prompt tuned for clinical language. The fallback is
a local PyTorch encoder-decoder transformer that runs when the API is
unreachable or no key is set. The fallback exists because the network is least
reliable exactly where the tool matters most; a translation service for
emergency rooms that fails without connectivity has failed at its own use case.

**Gloss to video.** Matched clips play in sequence in the mobile web client.

### Prototype flows

Visual design was Caitlin Weingarden's. I built the prototype flows in Figma:
the states the app can be in and the transitions between them. That is the same
problem as the mapping logic in a different tool, since both are questions about
what the system does when input does not fit the expected path.

## Impact

Won the Health Track at the UMich Ross Tech Innovation Jam.

Usability testing with three Deaf and hard-of-hearing participants. All
completed the core flow without instruction.

Pitch projections, not measured results and not independently sourced: 20%
communication improvement, 15 minutes saved per visit, $2B annual value. These
were hypothesis-driven estimates built for the pitch, not measured data.

## Limitations

Fingerspelling keeps output correct past the edge of the dictionary, but it is
slower to watch than a sign. A transcript heavy in unmapped terms degrades in
speed rather than accuracy. Expanding the map is the fix, and it costs filming
time.

The language selector offers ASL, BSL, and PSL. Only ASL is implemented.

Filming every sign does not scale to open-domain speech. The alternative is
generative signing, which reopens the trust problem the research surfaced. That
tension is unresolved.

Has a 5-10 second latency.
