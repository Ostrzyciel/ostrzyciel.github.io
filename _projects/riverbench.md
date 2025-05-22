---
layout: page
title: RiverBench
description: RiverBench is an open RDF streaming benchmark suite that can be used for a wide range of benchmarking tasks
img: assets/img/projects/riverbench.png
importance: 2
category: Knowledge graph streams
related_publications: true
---

I created **[RiverBench](https://w3id.org/riverbench/)** to provide a common, high-quality base for benchmarking various RDF systems. It aims to solve some of the frequent woes of benchmarks, such as poorly described and buggy datasets, missing license information, messy distribution formats, no systematic way to report results, and lacking documentation.

RiverBench is fully open and community-driven – **you** can submit your own datasets, benchmark tasks, results, and more. It heavily relies on CI automation to make sure all datasets follow the same high-quality standards: rich RDF metadata, clear licensing, multiple distribution formats, detailed documentation, and more.

RiverBench uses the **[RDF Stream Taxonomy (RDF-STaX)]({% link _projects/riverbench.md %})** to describe and validate the stream types of benchmark datasets. The datasets are distributed both in W3C-standard formats (N-Triples, Turtle), and in **[Jelly]({% link _projects/jelly.md %})**, a high-performance binary RDF format.

- **Find out more on the RiverBench website: [https://w3id.org/riverbench/](https://w3id.org/riverbench/)**
- GitHub: [https://github.com/RiverBench](https://github.com/RiverBench)
- Preprint: {% cite sowinski2023riverbench %}
