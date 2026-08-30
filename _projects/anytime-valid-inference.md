---
layout: page
title: Anytime-valid inference
description: A side interest, with a focus on anytime-valid log-rank tests for clinical trials.
img:
importance: 2
category: work
---

Anytime-valid inference is a set of statistical methods that stay valid no matter how often you look at the data. They let you monitor an experiment as observations arrive and stop at any point, without the inflated false-positive rate that repeated significance testing incurs. The central object is the e-value, a measure of evidence against a null hypothesis that can be updated at every observation.

This is a side interest of mine, alongside the imaging work. My current focus is on anytime-valid log-rank tests: sequential versions of the log-rank test that let a trial monitor time-to-event data as it accrues and stop early when the evidence is already clear. The aim is to make these methods practical for real clinical trials.

Two related items are on this site:

- **[e-values in practice — part 1]({% post_url 2026-07-02-safe-testing-part-1 %})** — a pedagogical post. It explains e-values through a deliberately simplified scenario, a coffee taste-test, as an introduction for readers new to the idea, and sets the classical approach beside its e-value counterpart.
- **[SAVI 2026 — anytime-valid log-rank testing for randomised trials](/savi-2026/)** — a conference poster, with supplementary materials, presented at SAVI 2026 on the log-rank work above.
