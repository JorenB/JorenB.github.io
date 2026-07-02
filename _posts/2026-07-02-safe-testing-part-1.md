---
layout: distill
title: e-values in practice — part 1
date: 2026-07-02
description: A practitioner's introduction to e-values and anytime-valid testing, walking through a simple coffee taste-test example and comparing it step by step to the classical approach.
hidden: false
sitemap: true
toc:
  - name: Setting the stage
  - name: The scenario
  - name: The traditional approach
  - name: Anytime-valid testing
  - name: Simulated trajectories
  - name: The whole population — the cloud
  - name: Power is a curve, not a number
  - name: Is it safe?
  - name: When to keep going
  - name: How the test design changes
---

## Setting the stage

The past few years have seen increasing media coverage on safe testing, based on e-values. The framework promises multiple advantages over classical hypothesis testing approaches based on p-values. A major selling point is continuous monitoring of the data as observations trickle in, allowing us to conclude experiments early if the results are sufficiently convincing. Furthermore, e-values quantify the _evidence against the null hypothesis_, whereas p-values mostly serve to make a binary decision on the statistical (non-)significance of a result.

That said, I have never seen a scientific paper report e-values instead of p-values yet. This is not surprising. Some researchers might be interested in setting up their analysis from an e-value perspective, but it is likely that peer reviewers would subsequently have difficulty interpreting the findings. Nearly all scientists are trained in the p-value paradigm, and e-values are the new kid on the block.

However, most work on e-values so far is highly mathematical. When I initially got interested in the framework, I had a hard time finding accessible resources on how to actually compute an e-value _in practice_. This post is an attempt to bridge that gap: I walk through a concrete example, covering first the classical approach, then its e-value counterpart.

## The scenario

We have two coffee brands: "premium" and "budget". Our goal is to figure out whether people prefer the premium brand over the budget brand. To this end, we set up an experiment where we have participants do a blind taste of both brands. After tasting, each participant indicates which of the two samples they preferred (abstaining is not allowed). Our null hypothesis $H_0$ is that there is no preference: each participant is equally likely to pick one sample as the other. Under this null hypothesis, the probability that a participant picks the premium brand as their preference equals $p=0.5$. Let's say we only care about deviations in one direction, with the premium being preferred. Furthermore, we set the significance level to $\alpha = 0.05$, meaning that we have at most a 5% chance of rejecting our null hypothesis — *given that this null hypothesis is, in fact, true* (this is the Type-I error rate). If our data force us to reject the null, we'd claim that people indeed prefer premium coffee over the budget brand.

However, it might be the case that the overall preference is tiny, and that only 51% of people prefer premium over budget. We are not interested in such a result, and in an experiment like this one we'd typically specify a *minimally relevant effect size*. Furthermore, detecting tiny effect sizes would require a huge number of participants, making our study hopelessly expensive. Let's say we only consider the effect meaningful if at least 70% of participants prefer premium. This becomes our alternative hypothesis $H_1: p = 0.7$.

## The traditional approach

We use the classical Neyman-Pearson hypothesis testing framework. The null hypothesis $H_0$ states that the preference $p = 0.5$, the alternative $H_1$ says that $p = 0.7$ — right at the boundary of our minimally relevant effect size. As before, we cap the Type-I error rate at $\alpha = 0.05$. We fix the desired power at 80%. So: we want to have at least an 80% chance of rejecting $H_0$, given that $H_1$ is true. If the effect is stronger in practice ($p > 0.7$), the experiment's power increases.

A power calculation reveals that we require $n=37$ participants to reject $H_0$ at the $\alpha = 0.05$ level with at least 80% power.

{% details "Power calculation details" %}
The probability of observing $k$ or more premium preference picks in $n$ participants is $P(\mathrm{Bin}(n, p) \geq k)$. For our Type-I error guarantee, we should ensure that this probability does not exceed 5% under $H_0$ $(p=0.5)$. Under $H_1$ $(p=0.7)$, we want this probability to be at least 80%. The smallest number of participants for which this is possible is $n=37$. The rejection threshold is then $k=24$: the probability of 24 or more participants picking premium is $0.049 < 0.05$ under $H_0$, and $0.81 > 0.80$ under $H_1$.
{% enddetails %}

That's all there is to it. We now put up posters and recruit participants. Once all 37 are in, we count the number who preferred premium coffee. If 24 or more prefer premium, we reject $H_0$ and claim that the expensive brand is indeed significantly preferred over budget. If we count fewer than 24 premium picks, the experiment is inconclusive. (Note: this is not the same as having shown that there is "no preference". We simply do not have sufficient evidence to rule it out.)

In fact, we do not always need to wait for the full 37. If we reach 24 premium votes before having seen 37 participants, we know that the verdict is sealed and we reject $H_0$. Similarly, once 14 participants have preferred _budget_ over premium, we can never reach the rejection threshold and we consider the experiment inconclusive. This *curtailment* introduces some flexibility in our setup, but in other aspects we are still tied to a rigid set of rules. For example, if we find that 23 out of 37 participants prefer premium, that's highly suggestive of a true effect — however, we have no way of continuing our experiment past this point. The p-value for this outcome is 0.09, and we'd get into fights with statisticians all over the world if we'd call our result "*almost* significant".

## Anytime-valid testing

The classical approach is inherently a *fixed-n design*. The anytime-valid (AV) testing framework (based on e-values) allows us to take a different approach in which we can *continuously* monitor the experiment's results. For each new observation, we simply update our running measure of evidence, which might grow enough to trigger a null rejection ("experiment successful"). No need to specify a sample size upfront; we keep collecting data until we either 1) find a null rejection or 2) exhaust our budget or patience. Furthermore, the anytime-valid test provides us with a **measure of evidence** that we have collected so far (against the null), which can inform us about whether or not to continue the experiment for much longer.

That's the sales pitch. But how does it work?

The anytime-valid test builds on an **e-value**. For our simple scenario, a possible e-value is the **Bayes factor** of the alternative hypothesis versus the null. The Bayes factor measures how much more likely it was to observe some data under $H_1$ than under $H_0$. In our experiment, our observations are the number of premium preferences $k$ out of $n$ participants observed *so far*. The Bayes factor — our e-value — for this experiment has the following expression:

$$
E = 1.4^k \cdot 0.6^{n-k}
$$

{% details "Derivation" %}
If the probability for a premium pick is $p$, the probability of observing $k$ premium picks in $n$ participants equals $P(\mathrm{Bin}(n, p) = k) = \binom{n}{k} \cdot p^k \cdot (1-p)^{n-k}$. Plugging this into the definition of the Bayes factor, the binomial coefficients cancel:

$$
\begin{aligned}
BF_{10} &= \frac{\binom{n}{k} \cdot p_1^k \cdot (1-p_1)^{n-k}}{\binom{n}{k} \cdot p_0^k \cdot (1-p_0)^{n-k}} \\[6pt]
        &= \left(\frac{p_1}{p_0}\right)^k \cdot \left(\frac{1-p_1}{1-p_0}\right)^{n-k},
\end{aligned}
$$

where $p_0, p_1$ correspond to the premium pick probabilities under $H_0, H_1$, respectively. Since $p_0 = 0.5$ and $p_1 = 0.7$ we find $BF_{10} =: E$ as written above.
{% enddetails %}

We update the e-value each time a new participant records their brand preference. An $E \leq 1$ means the data so far favor $H_0$; an $E > 1$ means they favor $H_1$. The higher $E$ is, the more evidence we have against the null.

The rejection criterion for this anytime-valid test is simple: at the significance level $\alpha$ (which we set to $0.05$), we reject the null hypothesis if our e-value grows higher than $1/\alpha$ (20 in our case) *at any point in time*.

Now let's see how this would look.

## Simulated trajectories

We simulate three taste test experiments under each scenario, each running up to $n=50$ participants. Under a real preference at exactly our design rate ($p = 0.7$, left), the evidence climbs, sooner or later crossing the $E=20$ threshold where we can declare significance. If there is no preference ($p = 0.5$, right), the e-value generally drifts near 1. At most 5% of these no-preference trajectories will ever cross the threshold: this is the Type-I error control offered by the anytime-valid testing framework.

{% include figure.html path="assets/img/2026-07-02-safe-testing-part-1/fig1_trajectories.png" class="img-fluid rounded z-depth-1" zoomable=true caption="A handful of taste tests. You declare at the first crossing of $E = 20$." %}

We see that two out of three premium-preference runs (left) reach a rejection well before the $n=37$ mark. One run even rejects at only 12 participants, having seen only one budget pick by that time. Even with curtailment, the earliest time the classical test could have rejected is at 24 participants. The blue curve also rejects early, at $n=25$. The purple trajectory shows what we pay for flexibility: it eventually does reject, but it needs 8 more participants than the classical test to do so (and 11 more when comparing to the curtailed classical test).

There is a subtlety here that is worth pointing out: the decision is the **first** crossing of $E=20$ and this decision is not reversed if the curve drops below the threshold later on. The blue curve illustrates such a scenario: after crossing at $n=25$, it dips back down towards $E\approx4$ before climbing again. That dip doesn't undo the earlier decision. Once $E$ has crossed the threshold, the anytime-valid framework guarantees the rejection stands, no matter what happens to $E$ afterwards.

Now it's nice to see that the AV test *can* reject early. But is this the typical behavior? And how soon can we expect a rejection *on average*?

## The whole population — the cloud

Instead of simulating just three trajectories, we now simulate 10,000 and investigate their aggregate behavior. Below we see the resulting *median* e-value at each $n$, along with bands indicating the quartiles and the 10th–90th percentile range. We also overlay the three example trajectories from the earlier figure.

{% include figure.html path="assets/img/2026-07-02-safe-testing-part-1/fig2_cloud.png" class="img-fluid rounded z-depth-1" zoomable=true caption="Most real-preference runs cross 20, almost none of the no-preference runs do." %}

The evidence typically climbs steadily for $H_1$ trajectories, but there is a substantial spread. Under $H_0$ the trajectories largely stay low, drifting near 1 or below throughout. Since this figure shows the *current distribution* of $E$ for each $n$, we cannot extract typical values for the number of participants required for a null rejection. However, we do see that the early-rejecting orange trajectory is clearly a "lucky" outlier that we'd only rarely encounter, whereas the purple curve quite closely tracks the median for some time.

The table below shows the summary statistics of the 10,000 simulated trajectories:

| statistic | classical | classical + curtailment | anytime-valid |
|---|---|---|---|
| % reject by $n=37$ *(classical power)* | 81% | 81% | 64% |
| % reject by $n=50$ | 81% | 81% | 77% |
| median (mean) rejection $n$ *(among rejecters by $n=50$)* | 37 (37) | 33 (33) | 25 (26) |
| 25th–75th percentile *(same population)* | 37–37 | 31–35 | 17–35 |

We see in the first row that the power of the AV test is lower at the classical experiment's conclusion (at $n=37$), meaning that we have a smaller chance of finding a positive result by that time. By $n=50$ the AV test has nearly closed the gap with the classical power. However, among the 77% that do reject by $n=50$, the median rejection $n$ is 25 — well below the classical test's $n=37$. Furthermore, the classical test has no way to continue past $n=37$, so its power is frozen at 81% the moment the 37th participant is in. The AV test has no such ceiling. Its power can keep climbing beyond that point through continuation of the experiment. This naturally raises the question: how does this power evolve over time?

## Power is a curve, not a number

The power of the anytime-valid test is the cumulative probability that we have observed a rejection by participant $n$. Therefore, it does not have a fixed value (like the classical power), but rather grows with each new participant. Below we visualize this growth as a function of $n$, based on 2 million simulated trajectories where participants indeed had a $p=0.7$ preference for premium coffee. The graph shows what fraction of trajectories have crossed the $E=20$ threshold (for significance level $\alpha = 0.05$) at or before each value of $n$.

{% include figure.html path="assets/img/2026-07-02-safe-testing-part-1/fig3_power_over_time.png" class="img-fluid rounded z-depth-1" zoomable=true caption="The AV test's power grows continuously, eventually surpassing that of the classical test." %}

At the classical experiment's $n=37$ conclusion, the AV test is still behind. However, it catches up and overtakes at $n=55$, exceeding classical's 81% power without having committed to that sample size upfront. In fact, the power eventually reaches 100%, although we'd need an infinite number of participants (and an infinite amount of coffee) to do so.

## Is it safe?

We have shown how the anytime-valid test behaves when the effect we're looking for is *real*: the simulations underlying the power-over-time curve were performed at $H_1$ where $p=0.7$. However, what if the effect simply does not exist? In that case, we want to limit the probability that we accidentally reject $H_0$ while it is in fact *true*. We briefly discussed this Type-I error control in [the scenario](#the-scenario), and the anytime-valid framework promises that it provides such guarantees. It is straightforward to verify this promise by simulating trajectories under $H_0$ (where $p=0.5$) and generating the corresponding power-over-time curve (again for significance level $\alpha = 0.05$).

{% include figure.html path="assets/img/2026-07-02-safe-testing-part-1/fig4_type1_safety.png" class="img-fluid rounded z-depth-1" zoomable=true caption="Under no preference, the false-alarm rate stays under 5%, however long you keep watching." %}

Some of the simulated trajectories indeed cross the significance threshold, but in the long run this fraction does not exceed the pre-specified significance level. That is precisely the anytime-validity that we want our test to satisfy: Type-I error control under continuous monitoring of the data.

## When to keep going

So far we have seen that the AV test can detect a real preference with a time-dependent power, and that its false-alarm rate stays controlled if there is no preference at all. The whole point is that we do not know *a priori* whether the preference exists or not (in reality, $p$ need not be exactly $0.5$ or $0.7$ either): that's why we do our experiment. This is also where the interpretation of $E$ as a measure of evidence becomes useful. If $E$ drops below 1 at some point, the data so far favor $H_0$ and we'd have little reason to continue. On the other hand, if $E$ is larger than 1 but still below the significance threshold, this indicates that $H_1$ explains our data better and we may want to add a few more observations to our list.

Recall the borderline situation from [the traditional approach](#the-traditional-approach), where we were just one premium pick short of significance. This could have motivated us to start a new experiment, but we wouldn't have been able to reuse the results from the original 37 participants since this would violate the test's error guarantees. The anytime-valid test carries no such restriction, and the e-value quantifies the running evidence of all participants tested so far. Continuing the experiment therefore means picking up where we left off. We are also never forced to commit to a fixed sample size. Instead, we can use the running evidence to make an informed choice about whether or not — and for how long — to continue the experiment. A more detailed discussion on such continuation rules is left for a potential future post.

## How the test design changes

Now that we've gone through the full e-value analysis of our coffee tasting experiment, let's zoom out and summarize the differences between the anytime-valid test and the classical approach.

| classical fixed-sample | e-value |
| --- | --- |
| fix $N$, look once, accept/reject | monitor, stop when convinced, report graded evidence |
| peeking is forbidden | peek at any time |
| one study, one verdict | evidence accumulates and can be combined |

Each row traces back to something we've seen. Instead of fixing the sample size upfront, we monitor data continuously and let the evidence build up. Peeking is forbidden in the classical framework, but in the AV context we update the e-value each time a new participant shows up and check whether $E$ has crossed the threshold. The fixed-sample design gives a single yes/no verdict, whereas the e-value approach allows us to grow the body of evidence. We can even combine evidence from multiple independent studies, simply by multiplying all their e-values together. Two independent taste tests, or ten, can be combined into one valid verdict without any extra machinery.

This concludes our basic description of the anytime-valid testing framework, applied to a simple example scenario. There are two separate topics I'd like to address in future posts. One is the use of different stopping rules for deciding when to continue or wrap up the experiment, as already hinted at in the previous section. The other is something we largely glossed over so far: throughout the analysis we have pretended that our guess for the effect size under $H_1$ was exactly correct. In practice, this does not occur, and we will always have to deal with a discrepancy between the *expected* and the *real* effect size. In another future post I'd like to discuss in more detail how the e-value analysis behaves under this "misspecification" of $H_1$.

---

<small>
*Disclaimer: this post was largely written by me, with substantial copy-editing help from Claude Sonnet. I take full responsibility for the contents.*
</small>
