---
layout: page
title: FOMO-Shift
description: Label-free, training-free adaptation of frozen foundation models to domain shift.
img:
importance: 1
category: work
toc:
  sidebar: left
---

Suppose you build a model on a dataset and it works. You trained it on data from one source, it does what you wanted, and you are ready to use it. Then you point it at data from a second source: a different hospital, a different scanner, a different lab. The data looks highly similar to you, but the model does worse. Often much worse.

This is domain shift, and it is one of the standing problems in applied machine learning. The model learned to deal with the data distribution of the first source, and the distribution of the second source is subtly (or not-so-subtly) different. Acquisition shift is one kind of domain shift: the underlying content (e.g. a tissue sample) is the same, but the way it was captured (scanned) changed. A different scanner, a different stain, a different site. That is the kind of shift this project set out to handle.

FOMO-Shift began as an idea: that a small correction in embedding space could undo domain shift while the expensive parts of the model stay frozen. We applied to NWO for a grant to test it, and it was awarded (project [NGF1609242045](https://www.nwo.nl/projecten/ngf1609242045)). The grant also funded a six-month [MSc thesis by Joeri](https://scripties.uba.uva.nl/search?id=record_57103), whose work became the early phases of this project. The project has since formally closed. In this report we write up what we tried, what worked, what did not, and where things stand now.

The method builds on foundation models: large networks, pretrained on a lot of data, that turn an input into a general-purpose embedding. FOMO-Shift keeps such a model frozen and does its correction in that embedding space.

## the idea in three steps

Call the source dataset X and the shifted dataset Y. The method has three parts.

1. Build a model on X. Use a frozen foundation model to turn each input into an embedding, then train a small downstream model on those X embeddings to do the actual task.
2. Train an adaptation model. This is a small transform that takes Y embeddings and moves them so their distribution matches the distribution of X embeddings. It trains without labels, and it does not touch the foundation model or the downstream model.
3. Deploy on Y. Run Y through the frozen foundation model, apply the adaptation model, and get the prediction from the frozen X downstream model. No further training on Y.

The bet behind this is a geometric one. The shift in the input, once passed through a frozen foundation model, should show up as a manageable structure in embedding space. If it does, a simple transform can undo it, and it can do so working from the embedding geometry alone, without knowing what caused the shift. That is what would make the method flexible. The next section, *The goal and the idea*, makes this assumption precise and explains why so much depends on it.

## the sections

The sections are meant to be read in order, each of them describing one natural stage of the project.

1. [The goal and the idea](/projects/fomo-shift/goal-and-idea/). The setup, the central assumption, and the design that follows from it.
2. [Proof of concept](/projects/fomo-shift/proof-of-concept/). The staged tests in Joeri's thesis, from toy data up to the first real acquisition shift.
3. [Full control with synthetic histopathology](/projects/fomo-shift/synthetic-histopathology/). A synthetic, cartoon-like pathology world we control end to end, and the strongest generality signal the method produced.
4. [Reality check](/projects/fomo-shift/reality-check/). Real whole-slide data and a modern foundation model, where the method stopped helping.
5. [One real shift: DCIS grading](/projects/fomo-shift/dcis-grading/). A setting with a genuine cross-cohort drop, a partial recovery, and the ceiling we hit.
6. [Where it leaves us](/projects/fomo-shift/where-it-leaves-us/). What the whole project adds up to, and what would have to change for the method to work.

## what the method assumes and needs

The method has three defining properties.

**Frozen embedding model.** The foundation model and the downstream model never retrain. Only the small adapter trains. In that sense the method is training-free for the expensive parts.

**Label-free adaptation.** The adapter uses no labels, from X or from Y. It matches distributions in embedding space, nothing more. Note that step 1 still uses X labels to train the downstream model. Label-free describes the _adaptation step_ and not the whole pipeline.

**Source use.** The adapter is a function of X and Y together. It needs a sample of X embeddings as the target to match against while it trains. Therefore, this is not source-free in the usual sense of that term. The one thing that is source-free is timing: once the adapter is trained, running it on new Y data needs nothing further from X.

[*The goal and the idea*](/projects/fomo-shift/goal-and-idea/) pins these definitions down more precisely.

## what we test it on

The method works on embeddings, so in principle the domain does not matter. Some of our tests are proof-of-concept, on toy data and natural images. The real-world tests are on whole-slide images (WSIs) from digital pathology, a practical fit: the kinds of acquisition shift there are well understood, the data is plentiful, and strong foundation models already exist to build on.

## why this report exists

We have not (yet?) reached the goals we set for the project. The general, reliable adaptation we were after did not come together — at least not so far. This report explains what we tried, what worked, and what did not. The grant was deliberately high-risk and high-reward, so a negative result was always on the table.

We are writing it up anyway, for three reasons. First, we tried _many_ different things, and some of it is a useful resource for anyone chasing similar ideas. Second, the failures are informative: they say something about where this kind of method can and cannot work. Third, it is a straightforward account of where the funding and the effort went.

The grant has formally ended. That does not mean the idea is finished. In this set of notes I'm taking stock of what we have achieved so far and where we might still try to take the idea in the future.

## who did what

The early phases, the toy studies, the natural-image tests, and the first pathology results, are Joeri's MSc thesis, and they stand on their own. The work after the thesis, the synthetic-histopathology studies, the whole-slide reality check, and the DCIS study, is mine.

<!-- Figure (todo): method schematic. Frozen foundation model, then the adapter applied to Y, then the frozen X downstream model. To be drawn later. -->
