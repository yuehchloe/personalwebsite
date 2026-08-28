---
title: "SignNow: Real-Time Audio to ASL Translation"
date: 2026-01-15
summary: "A three-stage speech-to-ASL pipeline that renders output as filmed human video. Won 1st place in Health Track of UMich Ross Tech Innovation Jam."
tags: ["NLP", "Whisper", "FastAPI", "Python", "Accessibility"]
cover:
    image: "cover.png"
    alt: "SignNow"
    relative: true
---

**Role:** ASL translation layer. User research, video corpus, gloss mapping and grammar rules, prototype flows.

**Team:** 5 people

**Timeline:** Sep 2025 to Jan 2026

**Award:** 1st place in Health Track, UMich Ross Tech Innovation Jam ($1,500)

**Code:** [github.com/SammiWang0516/SignNowPrototype](https://github.com/SammiWang0516/SignNowPrototype)

---

## Problem

Deaf patients in emergency care wait for interpreters during the window when decisions get made. Emergency calls involving language barriers require
[33 to 43 percent longer dispatch times](https://pubmed.ncbi.nlm.nih.gov/23952940/).
About [11 million Deaf and hard-of-hearing Americans](https://help.nationaldeafcenter.org/article/51-how-many-deaf-people-live-in-the-united-states)
use a healthcare system that routes around them by default.

The existing workarounds each fail somewhere. Written notes assume English fluency, which does not hold when ASL is the first language. Video remote
interpreting needs setup time and a stable connection, neither of which is reliable in an emergency room. Nothing on a phone went from spoken audio to ASL without a third party in the loop.

Speech recognition and text generation were not the hard part. Both are solved by models you can call. What no off-the-shelf component supplies is the middle:
which signs the system can express, how English maps onto them, and what counts as a correct answer when the two grammars diverge. That layer was my work.

## Solution

### What the research decided

I ran interviews with Deaf community leaders, and sent a survey to Deaf organizations around the United States. More then 70% participants regarded animated avatars as untrustworthy or unable to reach the same accuracy as human interpreters due to the lack of facial expressions. (Facial expressions are an integral part of sign language because of their grammatical functions.) Therefore, while pose estimation onto an animated figure is the cheaper path, it is unlikely to be adopted by the end user.

Choosing human video set three constraints on the downstream:

- The system can only express signs that exist as clips, so vocabulary becomes a design decision.
- Matching has to handle multi-word signs. For example, a filmed clip for BLOOD PRESSURE is one sign and not two.
- Unmapped words need a defined behavior, due to the nature of a closed dictionary.

### English to ASL Gloss Translation and Video Corpus

ASL gloss (written ASL) has different grammar than English. The word order and sentence structure all differ, so substituting word for word produces something a signer
cannot read. Getting the translation layer right meant defining what correct output looks like, then encoding it.

The rules I wrote cover copula removal, negation placement, W-question reordering, and semantic normalization. These run in the offline path and also define the target the prompted model is asked to produce.

Mapping gloss to video uses longest-phrase-first matching so multi-word signs resolve as single units, falling back to single-word lookup. Words outside the
map fall back to fingerspelling, letter by letter, which is what a human signer does with a name or an unfamiliar term.

I filmed the corpus, about 130 signs scoped to clinical encounters. The vocabulary came from two sources: frequency in medical transcripts, and common
daily words identified by working through likely scenarios. Clips are hosted separately and referenced by URL from the mapping file.

### The Pipeline

Three stages sit behind a FastAPI service, each exposed as its own endpoint so it could be tested in isolation.

**Audio to English.** Whisper base, running locally rather than through an API.
Latency was a factor, but the main reason was privacy: this audio carries personal medical information that should not leave the device to reach a third-party server.

**English to gloss.** The primary path sends the transcript to Perplexity's `sonar` model under a system prompt tuned for clinical language. The fallback is
a local PyTorch encoder-decoder transformer that runs when the API is unreachable or no key is set. The fallback exists because the network is least reliable exactly where the tool matters most; a translation service for emergency rooms that fails without connectivity has failed at its own use case.

**Gloss to video.** Matched clips play in sequence in the mobile web client.

### Prototype

<video class="rounded-video" controls muted loop playsinline width="360">
  <source src="two-way.mp4" type="video/mp4">
</video>

## Impact

Won the Health Track at the UMich Ross Tech Innovation Jam.

Usability testing with three Deaf and hard-of-hearing participants. All completed the core flow without instruction.

Pitch projections (hypothesis-driven estimates): 20% communication improvement, 15 minutes saved per visit, $2B annual value.

## Limitations

- Fingerspelling keeps output correct past the edge of the dictionary, but it is slower to watch than a sign. A transcript heavy in unmapped terms degrades in
speed rather than accuracy. Expanding the map is the fix, and it costs filming time.
- The language selector offers ASL, BSL, and PSL. Only ASL is implemented.
- Filming every sign does not scale to open-domain speech. The alternative is generative signing, which reopens the trust problem the research surfaced. That tension is unresolved.
- Has a latency of 5-10 seconds.
