---
layout: distill
title: "One real shift: DCIS grading"
description: "A setting with a genuine cross-cohort drop, a partial recovery, and the ceiling we hit."
toc:
  - name: "the setup"
  - name: "finally, a drop to recover"
  - name: "what adaptation recovered"
  - name: "the ceiling"
  - name: "the wall"
  - name: "scope"
---

[‹ FOMO-Shift overview](/projects/fomo-shift/)

In [Section 4](/projects/fomo-shift/reality-check/) DCIS grade stood out as the one endpoint where a shift still looked recoverable. This section is about that endpoint. I wanted a setting where a substantial cross-cohort shift was present. On modern embeddings most of that shift had disappeared, and a method that corrects shift cannot prove itself where there is nothing to correct. DCIS grading gave me a setting where the shift was still there.

## the setup

The task is predicting DCIS grade from whole-slide images across two cohorts: an internal Dutch DCIS cohort as the reference, and the UK Sloane cohort as the deployment set. They come from different institutions, and they were even scanned on different hardware: the internal cohort on Aperio scanners (.svs format) and Sloane on Hamamatsu (.ndpi). The gap between them is therefore a cross-institution and cross-scanner shift, the kind the method targets.

Grade is not a tile-level property. In subtyping a single tile often already shows whether the tissue is breast or colon, but a pathologist grades DCIS by examining the whole lesion, weighing the nuclear features across it and keying on the worst-looking regions. A single tile (at the magnification levels appropriate for FM feature extraction) rarely carries enough of that to pin the grade down. The label is therefore attached to the whole slide, just like the tasks in the previous section, so the tiles have to be aggregated into one vector per slide. What changes here is _how_ I aggregate: instead of the attention-based ABMIL, I use mean-pooling on the tile embeddings (essentially the MeanMIL approach), the simplest natural way to pool.

Two things made mean-pooling an appropriate choice here. First, the cross-cohort drop was more pronounced under mean-pooling than under ABMIL, which gave the adapter a larger gap to work on. Second, it does not push the tile embeddings through a learned attention layer, so the pooled vector stays in the UNI-2 embedding space the adapter operates in, and the effect of the adaptation stays clean to interpret.

The UNI-2 tile embeddings of a slide are mean-pooled into one vector, a logistic regression is trained on the reference cohort, and its ROC AUC is measured on the deployment cohort. The adapter is the usual one, trained with global MMD as a plain linear map (no LoRA initially), and applied to the tile embeddings before pooling.

One caveat on the data. Neither cohort is public, so this is not the setting I would prefer for a publication. I used it because the shift is clearly present, which is exactly what the earlier settings lacked.

## finally, a drop to recover

Across cohorts, DCIS grade shows a large drop. A classifier trained on the internal cohort scores about 0.77 AUC within that cohort and about 0.65 on Sloane, a drop of roughly 0.12. For the first time since TCGA-2k, there was an obvious gap for the method to work on. ER status drops similarly, from about 0.82 to about 0.72, while HER2 barely moves, from about 0.75 to about 0.73. 

## what adaptation recovered

Global tile-MMD recovered about half of the grade drop, lifting Sloane from about 0.65 to about 0.71 against a within-cohort ceiling near 0.77. This is not a single lucky fit. The recovery is +0.064 AUC by paired bootstrap (95% CI [+0.029, +0.100], positive on every draw) and +0.059 ± 0.016 by held-out nested cross-validation, so it holds up across folds. The spread across adapter seeds is negligible. It is the largest recovery I saw on uncurated, modern-embedding data.

Unfortunately, it was also specific to grade. ER recovery came out at −0.007 and HER2 at +0.015, both indistinguishable from zero. Whatever the adapter was doing for grade, it did not transfer to the other endpoints on the same slides.

<!-- Figure (todo): the grade recovery headline, AUC on Sloane before and after adaptation, against the internal Dutch cohort reference (entry_19 grade_recovery_headline). -->

## the ceiling

About half of the grade drop, roughly +0.06 AUC, was where the recovery stopped. I tried to improve beyond it in the obvious directions. The default adapter is a full linear map. Lower-capacity LoRA versions, bandwidth tuning, and cluster-conditional matching all left the number where it was. The LoRA sweep in particular showed that capacity was not the limit: low rank underfits, and higher ranks just match the full linear map. The recovery plateaued near half of the grade drop every time.

<!-- Figure (todo): the ceiling, recovery staying near half as capacity, bandwidth, and cluster-conditioning are varied (entry_19 grade_ceiling_levers). -->

## the wall

The result runs into the same label-free stopping problem from [Section 1](/projects/fomo-shift/goal-and-idea/), though here it caused less trouble than I expected. As the adapter trains, the MMD only ever decreases. It keeps decreasing well past the point where downstream performance stops improving, so it gives no signal for when to stop. What helped here is the shape of the grade-recovery curve: it climbs to about half the drop and then holds flat, so the last adapter I train is as good as any earlier one, and I did not need labels to pick a stopping point.

## scope

This is one endpoint, grade, out of the many different ones I tried across multiple datasets. It is scored with a linear model on mean-pooled embeddings, across two cohorts of one institution each. There are ABMIL results too, but whether the recovery holds up under a stronger aggregation like that needs a closer look. Some of the ceiling might come from the plain mean-pooling rather than the embeddings themselves. It is the method's best showing on realistic data, but it should be considered along with all these limitations. That said, I still consider this a minor success.

---

**‹ Previous:** [Reality check](/projects/fomo-shift/reality-check/) · **Up:** [FOMO-Shift](/projects/fomo-shift/) · **Next:** [Where it leaves us ›](/projects/fomo-shift/where-it-leaves-us/)
