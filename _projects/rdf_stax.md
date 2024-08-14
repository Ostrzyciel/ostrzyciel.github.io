---
layout: page
title: RDF Stream Taxonomy
description: RDF-STaX organizes RDF stream types and provides a common vocabulary for describing them.
img: assets/img/projects/rdf_stax.png
importance: 1
category: Knowledge graphs
related_publications: true
---

**[RDF-STaX](https://w3id.org/stax/)** started from a realization that many people talk about using "RDF streams", but almost everyone means something different by this term. Is this stream a sequence of RDF triples, graphs, or datasets? Are there timestamps in this stream? Does time even matter for your use case? RDF-STaX aims to systematize this with a taxonomy of RDF stream types.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/rdf_stax_taxonomy.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    RDF Stream Taxonomy, as proposed in the RDF-STaX paper {% cite sowinski2024rdf %}.
</div>

This taxonomy is represented as an RDF vocabulary ([https://w3id.org/stax/ontology](https://w3id.org/stax/ontology)), making it possible to easily reuse it in various applications. It can be used (for example) for annotating published datasets and RDF streams, or for embedding the stream type information in the stream itself, so that the stream processing systems can understand it (interoperability!).

- **Find out more on the RDF-STaX website: [https://w3id.org/stax/](https://w3id.org/stax/)**
- GitHub: [https://github.com/RDF-STaX](https://github.com/RDF-STaX)
- Journal paper: {% cite sowinski2024rdf %}
