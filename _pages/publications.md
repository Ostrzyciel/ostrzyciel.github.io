---
layout: page
permalink: /publications/
title: publications
description: Publications in reverse chronological order.
nav: true
nav_order: 2
---

<!-- _pages/publications.md -->

<!-- Bibsearch Feature -->

{% include bib_search.liquid %}

<div class="publications">

<h1>preprints</h1>

<!-- all params listed here: https://github.com/inukshuk/jekyll-scholar/blob/main/lib/jekyll/scholar/defaults.rb -->

{% bibliography -f preprints --group_by none %}

<h1>published &amp; in press</h1>

{% bibliography -f papers %}

</div>
