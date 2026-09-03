---
layout: distill
title: "Reality check"
description: "Real whole-slide data and a modern foundation model, where the method stopped helping."
toc:
  - name: "the step up"
  - name: "why this was UNI-2, not Phikon"
  - name: "what might explain it"
  - name: "approaches that did not help"
  - name: "one avenue left"
---

[‹ FOMO-Shift overview](/projects/fomo-shift/)

After the synthetic result I was optimistic. The obvious next step was real whole-slide data with a modern embedding model, and I expected the adapter to recover part of the cross-center drop there, as it had in every stage before. Unfortunately, it did not. This stage is where I could no longer make the approach work, and where I spent a long time trying to understand why. It is also the setting the method was built for. It worked in the simpler, controlled stages, but the realistic whole-slide case is where it was supposed to pay off, and this is where it did not generalize.

## the step up

I moved to real-world whole-slide images. The embedding model was UNI-2, a state-of-the-art pathology foundation model. Where TCGA-2k had used a curated set of about 200 tiles per center, I now used every tile from each slide, with no selection. The training setup aggregates those tiles into a slide-level prediction with attention-based multiple-instance learning (ABMIL), which is the standard way to classify whole slides.

I was hopeful, but the initial results were disappointing. I tried several classification tasks: tissue subtyping (breast versus colon, BRCA versus COAD), three DCIS endpoints (ER status, HER2 status, and grade), and colorectal microsatellite-instability (MSI) status. I could not make the adapter recover the cross-center drop reliably across them. Often it did worse than nothing: aligning the tile embeddings frequently degraded performance, because a transform that deforms the embedding geometry tends to destroy useful information, which is the expected outcome when there is no genuine shift to correct.

One target broke this pattern. On DCIS grade, unlike the others, there seemed to be a genuine cross-center drop that adaptation was able to partially recover. That was enough to pursue it on its own, which is what [Section 5](/projects/fomo-shift/dcis-grading/) describes.

It is possible that ABMIL's extra layer of complexity made the problem harder. To change just one thing at a time, I stepped back to tile-level classification without ABMIL, the same tile-level setup that had worked on TCGA-2k. Joeri saw gains there in his thesis, and I reproduced them in my own pipeline, so those results hold. Now on uncurated data, with far more tiles per slide, most of the cross-center drop had itself largely disappeared, so there was little left for the adapter to recover.

## why this was UNI-2, not Phikon

The choice of embedding model was deliberate. UNI-2 is a modern model, and I want the method to help on a model people actually use. If it only worked on an older, weaker embedding model, it would be solving an artificial problem, one created by using outdated machinery. That would not be an interesting result.

To check this was not a UNI-2 quirk, I went back to Phikon on the non-curated tiles. The drop was smaller than in the curated TCGA-2k setup, and adaptation did not recover it consistently. The curation and the model had probably both contributed to the thesis result.

<!-- **TODO (phrasing):** work out how to make the "gradual insight" point here. Shift was never a solved-then-reopened problem, and my understanding of it grew in pieces over the project. Needs a phrasing that says this without over-explaining. Joren to rework. we should think about this carefully since people might ask "why didn't you bother checking this before you started the projecT?" which is a fair question and i have to think about a proper answer -->

## what might explain it

I could not pin the failure on a single cause, so what follows is merely suggestive. The gap between the hopeful TCGA-2k result and this more realistic setting has, I think, three main sources.

**The curation.** The curated TCGA-2k tiles were cleaner and easier to match than a full slide's worth. Widening to all tiles took that advantage away, and the matching procedure had less structure to work with.

**The size of the shift.** On UNI-2 the cross-center gap is already small. For the tasks above, there was often little left to recover before the adapter even started.

**The diagnostics.** I ran a cluster diagnostic on 20 center pairs. In all 20, the UNI-2 embeddings clustered by biology, by tissue content, rather than by center. The measured cross-center drop was small, and it looked compositional. It came from differences in what tiles a center happened to contain, such as artefacts or fat, rather than from a center-wide nuisance shift the adapter could undo.

Put together, the earlier success leaned on a favorable setup, and in a realistic one the little shift that remained was mostly not the kind the adapter could remove.

<div class="row justify-content-center">
    <div class="col-12">
        {% include figure.html path="assets/img/fomo-shift/uni2-clusters-biology-vs-center.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">UNI-2 tile embeddings from one representative center pair (ILSBio and MSKCC), pooled and projected with PCA (top) and UMAP (bottom), equal numbers per center drawn in a single shuffled pass. Left: colored by acquisition center. Right: colored by tissue class. In PCA the two centers are thoroughly interleaved while the classes separate cleanly. UMAP resolves finer groups, but every region carries both centers and class stays the dominant axis. The embedding organizes mainly by biology, not by center, so there is little center-wide direction for the adapter to remove. This figure shows just one center pair, but a similar pattern held for all 20.</div>

<div class="row justify-content-center">
    <div class="col-12">
        {% include figure.html path="assets/img/fomo-shift/cross-center-drop-forest-logreg.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">Cross-center AUC drop for each of the 20 center pairs, tile-level logistic regression on UNI-2 features. Each drop is the within-center reference AUC (cross-validated) minus the deployment AUC, with a slide-level bootstrap confidence interval. Red marks a robust drop, where the deployment interval lies entirely below the reference. About half the pairs show one, but no drop exceeds 0.10 AUC, and the largest involve the small IGC center with the widest intervals. The shift that remains is small, so there is little for adaptation to recover.</div>

## approaches that did not help

I invested a lot of time in this stage, working through many possibilities. Everything had to stay label-free, since in deployment there are no labels on the new data. The common thread was to give the adapter more structure to respect and less freedom to misbehave.

**LoRA.** I constrained the adapter to the identity plus a low-rank correction, with the rank controlling how much it could change. A low rank is a simplicity bias: fewer degrees of freedom, a gentler correction, and less room to overfit the distribution match.

**Cluster-conditional MMD.** Rather than match the whole Y cloud to the whole X cloud at once, I first clustered the embeddings by their content, without labels, then trained the adapter to minimize the MMD within each cluster instead of globally. The hope was that the adapter would remove center effects while keeping the distinct tissue types apart, instead of aligning the clouds by folding classes together.

**Bandwidth.** The MMD kernel has a scale, and if it sits far from the size of the residual shift, the MMD barely responds. Joeri's automatic bandwidth selection ([Section 2](/projects/fomo-shift/proof-of-concept/)) was built to handle exactly this, and I relied on it. I also tried fixed and multi-bandwidth choices, to make sure the objective stayed sensitive to whatever center structure was left.

None of it changed the outcome. When LoRA did nothing on the real data, I went back and ran it in the synthetic setting as a sanity check. There it behaved exactly as expected: fine at modest ranks, and it broke down when I dropped the rank too far. The tool worked. On the real data the picture was messier. Sometimes there was a little performance that could have been recovered, but no approach consistently delivered such a recovery.

## one avenue left

At least one avenue remains that I have not yet tried. Throughout our ABMIL experiments, the adapter matched tile embeddings. One option would be to match at the slide level instead: each slide produces a single aggregated embedding from ABMIL's attention, which is the level where the prediction is made. That gives one embedding per slide rather than thousands of tiles, so the match would rest on far fewer samples and be harder to train, but it would act where the shift reaches the output.

---

**‹ Previous:** [Synthetic histopathology](/projects/fomo-shift/synthetic-histopathology/) · **Up:** [FOMO-Shift](/projects/fomo-shift/) · **Next:** [DCIS grading ›](/projects/fomo-shift/dcis-grading/)
