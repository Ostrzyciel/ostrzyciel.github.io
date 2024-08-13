---
layout: page
permalink: /publications/
title: Publications
description: Publications in reverse chronological order.
nav: true
nav_order: 2
---

**See also my:**

- <b class="ai ai-google-scholar"></b> [Google Scholar profile](https://scholar.google.com/citations?user=MPj9pngAAAAJ)
- <b class="ai ai-researchgate"></b> [ResearchGate profile](https://www.researchgate.net/profile/Piotr_Sowinski2/)
- <b class="ai ai-orcid"></b> [ORCID profile](https://orcid.org/0000-0002-2543-9461)

<!-- _pages/publications.md -->

<!-- Bibsearch Feature -->

{% include bib_search.liquid %}

<div class="publications">

<h1>Preprints</h1>

<!-- all params listed here: https://github.com/inukshuk/jekyll-scholar/blob/main/lib/jekyll/scholar/defaults.rb -->

{% bibliography -f preprints --group_by none %}

<h1>Published &amp; in press</h1>

{% bibliography -f papers %}

</div>
