---
layout: distill
title: "Full control with synthetic histopathology"
description: "A synthetic pathology world we control end to end, and the strongest generality signal the method produced."
toc:
  - name: "a world we control"
  - name: "training our own embedding model"
  - name: "the result"
  - name: "what this does and does not show"
---

[‹ FOMO-Shift overview](/projects/fomo-shift/)

Joeri's thesis ends at this point, and I took up the work from here.

The TCGA-2k result was encouraging, but it couldn't tell us how generally applicable our method was. I wanted to map out when adaptation works, and how reliably. It is difficult to take this up in a testing ground based on real-world data. You cannot obtain more data on demand, you cannot vary the shift or the task cleanly, and you constantly have to be careful to avoid data leakage issues. Therefore, I moved to a synthetic playground. There I could control everything: the data, the shift, the task, and the embedding model. I could also generate as much data as I wanted, with no leakage to worry about.

## a world we control

This section is about the data itself, the world I generate. The embedding model comes in the next section. The data is a synthetic histopathology dataset that I build from scratch. The images are cartoons inspired by real tissue: simple shapes standing in for cells, stroma, and background, arranged by rules I write. Because I generate the data, I set the class structure. I also set the shift, by changing the generation settings between the reference and deployment sets. That is a synthetic stand-in for a change of stain or scanner.

On this data I define four binary endpoints: has_stromal_reaction, has_lymphoid_aggregate, is_regressing, and shows_secretory_activity. For simplicity, each sample can only be positive for _one_ of the endpoints. I intentionally engineered these tasks to have different difficulty levels, with some meant to be inherently ambiguous. That way, the average downstream performance across the tasks ranges from poor to near-perfect, with different levels of performance drop to potentially correct for.

<div class="row justify-content-center">
    <div class="col-12">
        {% include figure.html path="assets/img/fomo-shift/synthhisto-reference-vs-deployment.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">Tiles from the synthetic world. Each is a cartoon of tissue: two cell types scattered over stroma, with the class structure set by generation rules. The reference and deployment sets share that structure but differ in acquisition appearance (hue, saturation, gamma, border thickness, and noise), a synthetic stand-in for a change of stain or scanner. These are representative tiles from each set, not the same "tissue" rendered twice.</div>

This data sits between two extremes. It is more realistic than the Gaussians of the first stage, which had no image content at all. On the other hand, it is far more controlled than real histopathology. I generate it in a structured, predictable way, so it has much less variability than real tissue. Real histopathology is a continuum, with no clean boundary between one tissue type and the next. Here the class structure is forced in strongly, so the clusters are much cleaner by construction. That is a feature for this experiment, but it limits what it can tell us. I further discuss this limitation at the end of this section.

## training our own embedding model

The other piece of control is the embedding model. Instead of a pretrained foundation model, I train my own on the synthetic data, with self-supervised learning. I use two methods, SimCLR and I-JEPA, so the conclusions do not hang on a single pre-training recipe.

The real-data stages could not give me this much control. Now the embedding model, the shift, and the task are all things I set. If adaptation works here, I can say how consistently it works across these choices, instead of relying on a single number from a single combination.

## the result

Adaptation recovered the shift in every configuration I tried. It held across both embedding models and across several classification tasks defined on the synthetic images.

This was the strongest generality signal the method had produced up to this point. On real data I had one hopeful result. Here it worked across a whole grid of choices I controlled, and in some cases it worked spectacularly well. That raised my confidence that the approach was robust across a range of circumstances, and that it might transfer to harder settings.

<div class="row justify-content-center">
    <div class="col-12">
        {% include figure.html path="assets/img/fomo-shift/synthhisto-recovery-headline.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">Downstream recovery in the synthetic world. One embedding model, four tasks defined on the synthetic tiles. Each task shows the AUC of a classifier trained on the reference set. Blue is the reference score. Teal is the shifted deployment set with no adaptation. Green is the deployment set after the label-free adapter. Each point is one reference/deployment pair, and the light lines join the same pair before and after adaptation. Adaptation raises every task back toward the reference. In the case with the largest cross-hospital drop, shows_secretory_activity, performance climbs from about 0.59 to about 0.92. The recovery is robust across all tasks and across all pairs.</div>

## what this does and does not show

This stage has clear limitations, and these limitations turn out to be important for what comes next.

The first limitation is the setting itself. Everything here is single-embedding, image-level classification on an input distribution I designed. One image gives one embedding, and the classifier acts on it directly. A real whole-slide problem is different in structure: a slide is thousands of tiles, each becomes an embedding, and a prediction comes from aggregating them, with the label attached to the whole slide rather than to any one tile. [Section 4](/projects/fomo-shift/reality-check/) takes that on.

The bigger limitation is the clean clusters. Separating and re-aligning the classes was easy here because I designed the classes to be separable. Real tissue is not built that way. The structure is not forced, the clusters are not clean, and, as I found out later, they do not always group themselves the way this experiment quietly assumes.

<div class="row justify-content-center">
    <div class="col-12">
        {% include figure.html path="assets/img/fomo-shift/synthhisto-embeddings-after.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">UMAP of reference/deployment/adapted tile embeddings, for one run. Marker shape is the source: a circle for reference, a cross for deployment, a square for adapted. Color is the endpoint label. The deployment embeddings form a cluster well-separated from the reference embeddings. The adapted deployment embeddings have moved into the reference cluster and keep their label structure.</div>

---

**‹ Previous:** [Proof of concept](/projects/fomo-shift/proof-of-concept/) · **Up:** [FOMO-Shift](/projects/fomo-shift/) · **Next:** [Reality check ›](/projects/fomo-shift/reality-check/)
