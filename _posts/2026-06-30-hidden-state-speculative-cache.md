---
layout: post
title: "What if you cached the model's hidden states instead of running it again?"
date: 2026-06-30
categories: llm-inference speculative-decoding
---

This started as an experiment. I didn't expect it to work this well.

---

When a language model generates a token, the hidden state at that step — the internal vector right before the final projection — encodes everything the model has processed up to that point. Context, position, all of it. It's a precise fingerprint of the model's computational state at that exact moment.

I started wondering: what if you stored those states during inference, and the next time the model lands in a "similar" state, you just reuse what it already computed?

That's the idea. Log hidden states and output distributions during inference. At runtime, retrieve the nearest stored state. If the match is close enough, skip the decode step and use the cached output.

No draft model. No second set of weights. The corpus *is* the draft — built from the model's own past computations.

---

## Does it work?

On a working prototype with a 4B parameter model running on a consumer GPU:

When the correct output token exists somewhere in the retrieval corpus, the system finds it with **99.4% accuracy**. The hidden state turns out to be an extremely precise retrieval key — more precise than I expected.

Overall accuracy (including cases where the corpus doesn't cover the current input yet): **~76%**.

That 76% isn't a retrieval problem. It's a coverage problem. The corpus is still small. When the model encounters something outside its logged trajectories, there's nothing to retrieve. I'm currently building a larger corpus to see where the ceiling actually is.

---

Still a prototype. Still measuring. I'll post results once the larger corpus evaluation is done.

Code isn't public yet, but I'm happy to discuss — reach out.
