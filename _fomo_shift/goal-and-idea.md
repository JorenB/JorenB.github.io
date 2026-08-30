---
layout: distill
title: "The goal and the idea"
description: ""
toc:
  - name: "the setup"
  - name: "the central assumption"
  - name: "the design"
  - name: "what the adapter is, and what it needs"
  - name: "measuring the shift with MMD"
  - name: "starting from the identity"
  - name: "measuring success"
  - name: "one problem we could not solve"
---

[‹ FOMO-Shift overview](/projects/fomo-shift/)

In this section we fill in the details of the recipe we outlined in the intro. We state the assumption the whole method depends on: that a foundation model turns a large shift in the input domain into a small one in embedding space. We describe the adaptation layer we train to exploit that: a single linear map that aligns the two embedding distributions using a sample-based distributional distance measure. We say how we measure recovery. Finally, we describe the problem that limited most of the results after it: without peeking at downstream transfer performance, we usually could not tell when to stop training the adaptation layer.

## the setup

We build a model in two pieces. A frozen foundation model turns each input into an embedding. A small downstream model, trained on the X embeddings, makes the prediction we actually want (e.g. image class, biomarker status).

Then we deploy on Y, a second dataset that is shifted from X. The content is the same kind of thing, but the way it was produced changed. In pathology that means a different scanner, stain, or site. In our non-medical tests it might mean a corruption applied to natural images, such as blur or a color shift. Because of the shift, the Y embeddings come out looking subtly different from the X embeddings the downstream model was trained on, so its performance on Y drops. That drop is what we want to mitigate.

## the central assumption

Everything rests on one assumption.

The shift between X and Y can be large in the input. A change in brightness, focus, or color moves a lot of pixels by a lot. However, the meaning is mostly preserved: a person can read through the change without much trouble. The assumption is that a frozen foundation model, which was trained to encode semantic content, does the same. It absorbs most of the input shift, so what remains in embedding space is small and simple enough that a single linear map can undo it.

If that holds, two things follow. The correction works from the embedding geometry alone, without knowing what caused the shift. This is what would make the method general. The correction can also be small, because the leftover shift is small.

If the central assumption does not hold, the method has nothing to stand on. If the foundation model tangles the shift into the content in a way no simple map can separate, no adapter of the kind we use can help. The method can therefore only work when this assumption holds for the particular model and shift at hand. Later sections are, in large part, the story of where it held and where it did not.

## the design

The design is the direct consequence of our central assumption. We keep the foundation model and the downstream model frozen, and we train one small thing: an adaptation layer that acts in embedding space. It takes a Y embedding and transforms it so that the distribution of adapted Y embeddings matches the distribution of X embeddings. The frozen downstream model then treats the adapted Y embeddings as if they had come from X.

The adapter is a single linear layer. [Section 2](/projects/fomo-shift/proof-of-concept/) shows why we do not make it deeper.

## what the adapter is, and what it needs

The introduction named three properties of the method. Here they are with the detail the introduction skipped.

**Frozen backbone.** The foundation model and the downstream model stay frozen: they keep the weights they already had, and neither is retrained for Y. The one piece we train is the adapter, a single linear map (a d-by-d matrix, with d the embedding dimension). Because the foundation model never changes, every embedding can be computed once and cached, and the adapter trains on the cached embeddings. Adaptation is therefore cheap: few parameters, no backpropagation through the foundation model, and little data.

**Label-free.** The training signal is the distance between two clouds of embeddings, X and adapted Y. No label enters it. This is what separates the method from most domain adaptation, which leans on source labels, target pseudo-labels, or both. Two other label uses sit outside the adaptation and are easy to confuse with it: step 1 trains the downstream model on X labels, and our evaluation uses Y labels to score the result. Neither touches the adapter. The difference between "the adaptation needs no labels" and "we needed Y labels to know it worked" is the exact problem at the end of this section.

**Source use.** In the domain-adaptation literature, source-free means the adaptation sees no source data at all. Ours does: the objective compares adapted Y against a sample of X embeddings, so a batch of X has to be on hand while the adapter trains. It needs the X embeddings instead of the raw X inputs (images), which helps when the source images cannot travel. Once the adapter is fixed it carries no X inside it, so deployment on fresh Y needs nothing from X. Source-free at deployment, not during training.

## measuring the shift with MMD

To bring one embedding cloud toward another, we need a number that says how far apart two distributions are, computed from samples alone. We use the Maximum Mean Discrepancy, or MMD.

MMD compares two distributions through a kernel. With a suitable kernel it is sensitive to differences beyond the mean and the variance, so it can catch shifts that a coarser measure would miss. It needs only samples from each side, with no density estimation. The adapter trains by pushing the MMD between adapted Y and X down toward zero.

The natural comparison is CORAL, which aligns the second-order statistics, the covariances, of the two clouds. That is cheaper and sometimes enough, but it is blind to any difference the covariance does not capture. MMD is the more sensitive instrument, at the cost of a kernel and its bandwidth, which becomes its own problem in [Section 2](/projects/fomo-shift/proof-of-concept/).

<details>
<summary>The MMD, more precisely (skippable)</summary>

For distributions $P$ and $Q$ and a kernel $k$, the squared MMD is

$$\mathrm{MMD}^2(P, Q) = \mathbb{E}_{x,x'\sim P}[k(x,x')] + \mathbb{E}_{y,y'\sim Q}[k(y,y')] - 2\,\mathbb{E}_{x\sim P, y\sim Q}[k(x,y)].$$

With a characteristic kernel, such as the RBF kernel $k(x,y) = \exp(-\lVert x-y\rVert^2 / 2\sigma^2)$, this is zero if and only if $P = Q$. We estimate each expectation by averaging the kernel over a mini-batch of embeddings, so the whole quantity is available from samples. The bandwidth $\sigma$ sets the scale at which differences are measured, and the procedure turns out to be sensitive to an appropriate choice of $\sigma$, so [Section 2](/projects/fomo-shift/proof-of-concept/) returns to it.

</details>

## starting from the identity

The adapter starts as the identity map, plus a small perturbation. This follows from the central assumption again, now in the initialization. If the leftover shift in embedding space is small, then the map that undoes it is close to the identity, so the identity is where training should begin.

Starting there also avoids a failure that MMD invites. MMD only asks that the two clouds overlap. A map that swaps two classes, sending Y's class A onto X's class B and the reverse, can drive the MMD down just as well as the correct map, while destroying information the downstream model relies on. Beginning near the identity keeps the adapter in the basin of the correct alignment instead of a swapped one.

The adapter is a single linear layer for a related reason. A more expressive adapter, such as an MLP, can warp the Y cloud onto the X cloud in nonlinear ways that minimize MMD while ignoring the class structure. [Section 2](/projects/fomo-shift/proof-of-concept/) shows this collapse directly. A linear map is far more limited, so it has far fewer ways to match the clouds while wrecking what the downstream model needs. Still, it is not immune to collapse, which is one more reason we start from the identity.

## measuring success

We report downstream performance in three places: on X, on Y before adaptation, and on Y after adaptation. X is the reference, the score when there is no shift to fight. Y before adaptation is what the shift left us with. We take the gap, X minus Y-before, as the shift's cost, and Y after adaptation says how much of that cost the adapter recovered. This treats X as the level to aim for, which is a simplification: Y need not be exactly as hard as X, so the gap is an estimate of what is recoverable rather than an exact budget.

It is important to consider these three numbers together. A gain on Y means little if the gap was small to begin with. This is precisely what we found with modern foundation models. [Section 4](/projects/fomo-shift/reality-check/) is where that catches up with us.

## one problem we could not solve

The evaluation above uses Y labels. The adaptation itself does not, but measuring whether it worked does. In a real deployment we would not have Y labels, and without them we cannot tell how well the adapter did.

As the adapter trains, the MMD keeps dropping. Downstream performance on Y does not necessarily track it. It may rise, level off, and in some runs decline again while the MMD is still improving, as the adapter starts matching the clouds in ways that no longer respect the task. To stop at the best point, we would need to see the Y performance, which needs Y labels. Without them there is no label-free signal that says "stop here".

We never identified a reliable signal to do this. It limited most of the later results, though not all: on DCIS grade ([Section 5](/projects/fomo-shift/dcis-grading/)) the recovery curve happened to plateau, so the final adapter was as good as any earlier one and no labels were needed to stop. Where the curve does not behave that way, we have no label-free way to pick the best adapter state. [Section 6](/projects/fomo-shift/where-it-leaves-us/) comes back to it as one of the central open problems.

<!-- Figure (todo): method schematic. Frozen foundation model, adapter on Y, frozen downstream model, with the MMD comparison drawn against a sample of X embeddings. Candidate: Joeri thesis Fig 1.1, or redrawn. -->

---

**Up:** [FOMO-Shift](/projects/fomo-shift/) · **Next:** [Proof of concept ›](/projects/fomo-shift/proof-of-concept/)
