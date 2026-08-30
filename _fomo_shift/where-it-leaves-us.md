---
layout: distill
title: "Where it leaves us"
description: "What the project adds up to, and what would have to change for the method to work."
toc:
  - name: "what the method needs"
  - name: "what stands in the way"
  - name: "the missing test bed"
  - name: "if the project resumes"
  - name: "where it leaves us"
---

[‹ FOMO-Shift overview](/projects/fomo-shift/)

The idea was a single label-free transform in embedding space that undoes domain shift, on top of a frozen foundation model. It worked cleanly in the controlled stages and gave one solid result on realistic data. Unfortunately, it has not matured into the general tool I was hoping to build. In this final section I will take stock of our findings and list potential directions I may still pursue in the future.

## what the method needs

What FOMO-Shift truly needs to show its worth: a real shift between the two datasets that 1) shows up in the embedding space, that 2) a simple map can undo, and that 3) actually degrades performance on downstream tasks. A shift that leaves the model's performance intact is nothing to fix. The bigger the drop it causes, the more an adapter might be able to recover. A small drop leaves little to gain, and where there is no drop at all, forcing a transform through only does damage.

The catch is that on a modern foundation model the drop from the shift is often mild. It is not totally absent, and it varies from one endpoint to the next, but for many of the tasks I tried there was little to rescue. It was possible to bring a larger drop back by picking an old, weak embedding model, but that would solve a problem I made for myself, and that result would not mean much in practice. What I would want instead is a drop that still holds on a model someone would actually deploy. The closest I came to that in pathology was in predicting DCIS grade.

I tried to move in small steps, changing one thing at a time so I could see what each change did. The toy data and the synthetic histopathology worked because I set the shift myself and the classes were clean and separable. The TCGA-2k experiments worked on an older embedding model, with a wide cross-center gap and a curated tile set. What I am currently guessing is that some of the steps I still had to take to reach a realistic setting were themselves the ones that broke the method, even taken in isolation: 1) the move to uncurated whole slides where the two distributions are messier and 2) the move to a modern model that absorbs most of the shift. What I noticed over the course of the work was that many of the drops were modest and that the recovery was hard to get working consistently across arbitrary endpoints. DCIS grade came closest to a real shift on a modern model, and even there the recovery was partial.

## what stands in the way

For many of the tasks I explored in this setup, the performance degradation under a modern foundation model turned out to be smaller than expected. These models already absorb most of the cross-site variation, so there was little degradation left to recover. With so little to work with, it was hard to tell whether a measured gain was robust and consistent across runs, or sat within the noise.

The other problem I keep running into is the label-free selection of the adaptation layer. As the adapter trains, the MMD keeps dropping, but a falling MMD does not tell me the downstream task got any better. There is no single quantity to optimize for, because whether an adapter improves things is defined relative to a particular downstream task (or several of them). On DCIS grade this was not an issue, because the recovery curve rises and then flattens out. I do not yet know whether other suitable endpoints behave that way.

## the missing test bed

There may be other domains where the shift is more commonplace, and radiology is an good candidate where we can ask this question. There are several reasons for this: radiology foundation models are still in their infancy, many tasks cannot yet be done properly at the sample level (after aggregation), and accordingly not much is known about the problem of domain shift in this context. I could not test it properly for the same reasons: current CT/MRI foundation models are not as successful as state-of-the-art pathology models. It is where I would look for a fair test.

## if the project resumes

A few things I would try next.

*Follow up on DCIS grade.* Grade was the one endpoint where the method clearly helped. I would work out what set it apart from the endpoints where it did not, in the hope that this points to a class of problems where FOMO-Shift is worth applying.

*Match at the slide level.* Every adaptation so far worked on tile embeddings, but the prediction is made on one aggregated embedding per slide. With mean-pooling this is clean, since the slide embedding is just the average of tile embeddings and stays in the foundation model's space. With a learned aggregator like ABMIL it is messier, because the matching target is then shaped by the downstream attention rather than living in the raw embedding space. Either way the match would rest on one embedding per slide instead of thousands of tiles, far fewer samples to train the adapter on.

*A label-free selector.* The stopping problem is the one that would make deployment far more practical, and it is unsolved. Until there is a way to choose the adapter without labels, the method stays awkward to use in the field.

## where it leaves us

There may still be a place where this works. It would need a domain whose shift stays large on a strong model, together with a way to stop training the adapter without labels. So far I have not found that combination. Across the settings I tried, a single label-free transform did not reliably undo the shift on a modern foundation model. The DCIS grade result says the idea is not empty. The rest of the project says it is (probably) not general. Still, I don't consider the project dead, and I may still come back to some of the directions above.

<!-- Figure (todo): optional, a single recap timeline of the phases, from toy data to DCIS grade, marking where the method helped and where it did not. -->

---

**‹ Previous:** [DCIS grading](/projects/fomo-shift/dcis-grading/) · **Up:** [FOMO-Shift](/projects/fomo-shift/)
