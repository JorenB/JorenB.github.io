---
layout: page
title: recommended
permalink: /recommended/
description: Stuff I find interesting
nav: true
sitemap: false
nav_order: 5
toc:
sidebar: left
---

<div class="recommendations">
{%- for category in site.data.recommendations.categories %}
{%- assign recs = site.data.recommendations.items | where: "category", category.name -%}
<h2 class="category">{{ category.title }}</h2>
<ul class="rec-list">
{%- for rec in recs -%}
<li>
<h3><a href="{{ rec.url }}">{{ rec.title }}</a></h3>
<p class="post-authors"> by {{ rec.authors }}</p>
<p class="post-description">{{ rec.description}}</p>
</li>
{%- endfor %}
</ul>

{% endfor %}

</div>
