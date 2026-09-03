---
layout: distill
title: "Proof of concept"
description: "Staged tests from Joeri's thesis, from toy data up to the first real acquisition shift."
toc:
  - name: "toy data: does the machinery work"
  - name: "natural images: a real embedding model, a synthetic shift"
  - name: "the first real shift: TCGA-2k"
  - name: "choosing the bandwidth"
  - name: "the conditions behind the result"
---

[‹ FOMO-Shift overview](/projects/fomo-shift/)

Joeri's thesis is where the FOMO-Shift idea was first implemented. It runs as a staged progression, and the stages are organized by how much control we keep. We start with toy data, where we set everything by hand and there is no embedding model at all. We move to natural images, where the embedding model is real but the shift is still synthetic. We end on real digital pathology, where both the embedding model and the shift are real. Each step gives up a piece of control and moves closer to the setting we actually care about.

The point of the progression is to test the central assumption under steadily harder conditions, and to find out where the machinery breaks before the data gets complicated.

## toy data: does the machinery work

The first stage uses 2D and 3D Gaussians and their mixtures. There is no embedding model here, by design. The adapter and the MMD work directly in data space, on points we place ourselves. This stage is not about pathology or about foundation models. It asks a narrower question: given two clouds and a shift we control, can the adapter find the map that aligns them?

Joeri's experiments showed that it can. The toy data also made two failure modes concrete, both of them anticipated in [Section 1](/projects/fomo-shift/goal-and-idea/). A randomly initialized adapter would sometimes settle on a class-swapped alignment, matching the clouds while sending one class onto another. A more expressive MLP adapter would collapse the clouds together in ways that drove the MMD down and ignored the class structure. The single linear map, started at the identity, avoided both.

A third behavior showed up, and it was no surprise given how symmetric the toy distributions are. When the distributions are symmetric, the match is under-determined: several different maps drive the MMD to the same low value, and only one of them is the label-preserving map we want. MMD alone cannot tell them apart. This is simply an artifact of the symmetry we built in.

## natural images: a real embedding model, a synthetic shift

The second stage brings in a real embedding model. We embed two natural-image datasets, CIFAR-10 and Imagenette (a ten-class subset of ImageNet), with a frozen DINOv2 encoder, a self-supervised vision transformer pretrained on a large natural-image corpus. For the shift we apply corruptions from the ImageNet-C family, such as blur, noise, or a weather effect, at a chosen severity. The embedding model is now real, but the shift is still one we impose ourselves.

Adaptation helps here. The gains are modest, but they are consistent across corruptions, and they grow with the severity of the corruption. That last part is the encouraging sign: the more the shift moves the embeddings, the more the adapter can recover. It is the behavior the central assumption predicts, on real embeddings for the first time.

There is an important caveat in the nature of the shift. ImageNet-C corruptions are degradations: they make each image worse, in a controlled and roughly uniform way. Real acquisition shift is not like that. A different scanner or lab produces images that are different, not necessarily worse, and the difference is not uniform across the data. The corruptions are therefore a convenient synthetic shift, but a gentler and more structured one than the shifts we ultimately care about.

## the first real shift: TCGA-2k

In the third stage, we reach the first real acquisition shift. We use TCGA-2k, a tile-level dataset drawn from many centers in digital pathology. We embed the tiles with Phikon, a pathology foundation model. The task is to tell breast (BRCA) from colon (COAD) tissue. The shift is the center. TCGA is fairly homogeneous, mostly Aperio scanners under a shared protocol, so the center-to-center differences are subtler than a cross-hospital comparison would give. Even so, site, lab, and staining vary from center to center, which is the kind of acquisition shift the method targets. We adapt between pairs of centers and check whether the downstream classifier does better on the shifted center afterward.

Joeri found that it does indeed. Of the 20 center pairs, 18 improved after adaptation. This was what we had been hoping to see all along: a label-free adapter, trained only to match embedding distributions, recovering real cross-center performance on real tissue! At this stage, I got very optimistic about the broad applicability of the approach. I felt like we would now just proceed to implement FOMO-Shift in various contexts (integrating ABMIL aggregation, moving to domains outside pathology), show that it works, write up the results and send in to some nice journal.

<div class="row justify-content-center">
    <div class="col-12">
        {% include figure.html path="assets/img/fomo-shift/tcga2k-phikon-before-after.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>
<div class="caption">TCGA-2k, breast-versus-colon classification on Phikon embeddings. Each point is one of the 20 center pairs: the horizontal axis is the downstream accuracy on the shifted center before adaptation, the vertical axis is the accuracy after the label-free adapter is applied. Points above the diagonal improved. 18 of the 20 pairs did, for a mean gain of 0.14. Figure by Joeri (thesis, section 4.4.1).</div>

However, that was the optimistic road. There was still quite a bit left to do before we got there.

## choosing the bandwidth

MMD with an RBF (radial basis function) kernel has a free parameter, the bandwidth, which sets the scale at which the kernel notices differences. Set it too wide and every pair of points looks alike. Set it too narrow and every pair looks distinct. Either way the gradient that trains the adapter is weak.

Joeri's contribution here was to stop treating the bandwidth as a fixed hyperparameter and instead choose it to make the MMD as sensitive as possible to the current gap between the clouds, updating it as training proceeds. It made training more stable and less dependent on a lucky guess, and it becomes more important as the embeddings get higher-dimensional in later sections.

<details>
<summary>The optimal-bandwidth idea, a little more precisely (skippable)</summary>

For an RBF kernel $k(x,y) = \exp(-\lVert x-y\rVert^2 / 2\sigma^2)$, the bandwidth $\sigma$ fixes the length scale of the comparison. If $\sigma$ is far from the typical distance between the two clouds, the kernel saturates and the empirical MMD is close to zero regardless of the true discrepancy, so its gradient with respect to the adapter is tiny. The dynamic scheme selects $\sigma$ during training to keep the MMD in its sensitive range for the clouds as they currently sit, rather than fixing it once from a heuristic such as the median pairwise distance. Because the clouds move as the adapter trains, a bandwidth chosen once can drift out of range, which is why choosing it dynamically helps.

</details>

## the conditions behind the result

The TCGA-2k result holds up. However, it came out of particular conditions and the next sections change those conditions one by one.

Phikon exposes a large shift. It is an older, lightweight pathology model, convenient for these early experiments. Its cross-center gap is wide, which leaves a lot for the adapter to recover. The [PathoROB benchmark](https://www.nature.com/articles/s41467-026-73923-2) puts numbers on this: pathology foundation models differ a lot in how much medical-center signal dominates the biology, with UNI-2 among the most center-robust and older, lighter models among the least. The tiles are curated, roughly 200 clean tiles per center, so the two embedding clouds are tidier than a full slide would give. Some center pairs also have mismatched class proportions. Part of what the adapter faces is then a difference in what the centers contain, not only in how they were scanned.

None of this undoes the TCGA-2k result. It does mean the result came from favorable conditions: a large recoverable shift, clean data, and a comparatively easy task. [Section 4](/projects/fomo-shift/reality-check/) is what happens when we remove those advantages.

---

**‹ Previous:** [The goal and the idea](/projects/fomo-shift/goal-and-idea/) · **Up:** [FOMO-Shift](/projects/fomo-shift/) · **Next:** [Synthetic histopathology ›](/projects/fomo-shift/synthetic-histopathology/)
