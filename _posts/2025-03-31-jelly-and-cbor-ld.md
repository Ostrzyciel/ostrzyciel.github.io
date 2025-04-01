---
layout: post
title: "Jelly and CBOR-LD: two flavors of binary RDF"
date: 2025-04-01 00:00:00 +0000
description: "Playing with two binary RDF formats, Jelly and CBOR-LD, through experiments on IoT data and Verifiable Credentials."
tags: rdf jelly cbor-ld iot compression
giscus_comments: true
thumbnail: assets/img/blog/2025/jelly_and_cbor_ld.png
related_publications: false
---

I've recently learned about CBOR-LD (see [presentation](https://docs.google.com/presentation/d/1ksh-gUdjJJwDpdleasvs9aRXEmeRvqhkVWqeitx5ZAE/)), a binary version of [JSON-LD](https://json-ld.org/) with advanced compression mechanisms to make the serialized representation as small as possible. From what I understand, the main use case for CBOR-LD are QR codes and NFC tags, where the reducing size of the data is crucial.

This got me really interested, because I maintain [Jelly](https://w3id.org/jelly), also a binary RDF format. Jelly, however, focuses on very high read-write performance, and while it is pretty compact, I would not expect it to be comparable to CBOR-LD in terms of size.

## How does CBOR-LD work?

Quite honestly, I have a limited understanding of that, but here's what I've figured out so far. In JSON-LD there is the notion of a "context", which is basically a dictionary that maps JSON keys to more complex structures.

Here is an example from the [JSON-LD 1.1 spec](https://www.w3.org/TR/json-ld11/#example-7-in-line-context-definition). We can take an input document like this:

```json
[{
  "http://schema.org/name": [{"@value": "Manu Sporny"}],
  "http://schema.org/url": [{ "@id": "http://manu.sporny.org/" }],
  "http://schema.org/image": [{ "@id": "http://manu.sporny.org/images/manu.png" }]
}]
```

...add a context like this:

```json
{
  "@context": {
    "name": "http://schema.org/name",
    "image": {
      "@id": "http://schema.org/image",
      "@type": "@id"
    },
    "homepage": {
      "@id": "http://schema.org/url",
      "@type": "@id"
    }
  }
}
```

...and we get a compacted document like this:

```json
{
  "@context": ... ,
  "name": "Manu Sporny",
  "homepage": "http://manu.sporny.org/",
  "image": "http://manu.sporny.org/images/manu.png"
}
```

I've replaced the context with `...` here, because there are actually two ways you can use a context in JSON-LD: you can either inline it (put the whole `@context` map there) or you can refer to it by a URL. The latter is a clever trick, because you are removing a bunch of information from the document and putting it somewhere else. The producer of the data and the consumer must agree on the context, but once they do, the data can be very compact. This behavior is in fact unique to JSON-LD among standardized RDF formats. The others (like Turtle) never put data outside of the document.

CBOR-LD is first of all a translation of JSON-LD to binary, which already saves a few bytes. But it also takes this context mechanism to the extreme with what it calls ***semantic compression***. The context dictionary is then indexed by integers (not strings), and you can also put more pattern types in the dictionary than with JSON-LD. CBOR-LD also uses value encodings of literals (so, instead of a string "1234" you would have an actual integer), which is another way to save space, while losing a bit of information (RDF purists might not like that).

My explanation is probably not very accurate, so I recommend you check out the [presentation slides](https://docs.google.com/presentation/d/1ksh-gUdjJJwDpdleasvs9aRXEmeRvqhkVWqeitx5ZAE/)  I mentioned earlier or the [draft spec](https://json-ld.github.io/cbor-ld-spec/) for more details.

## How does Jelly work?

That's really a story for a few other blogposts! 😆 But, in short, [Jelly](https://w3id.org/jelly) is much closer to more traditional RDF formats (like [Turtle](https://www.w3.org/TR/turtle/)) than CBOR-LD. Internally, it looks a lot like an N-Triples/N-Quads file (just a long list of triples), but the triples are interspersed with dictionary entries. 

For example, if we are going to use the IRI `http://xmlns.com/foaf/0.1/knows`, we put the prefix `http://xmlns.com/foaf/0.1/` in the prefix lookup under, say, ID 4, and then we put the string `knows` in the name lookup under ID 2. The IRI then looks like this: `RdfIri(4, 2)`. The lookups are updated as we go along through the stream, evicting old entries if needed. They are sent via the same byte stream as the triples/quads, so you get the entire type in one file – just like in Turtle or N-Triples.

There are also a few other techniques used, including compressing repeating terms and some tricks with differential compression of lookup references... You can find the [full spec on Jelly's website](https://jelly-rdf.github.io/dev/specification/serialization/), if you are interested. The nice part about it is that it's very fast, works automatically with any RDF data (no need to think about contexts), can handle data of any size (with bounded memory!), and is sufficiently compact for most use cases.

## Experiment 1 – IoT data

Like I mentioned, the two formats have very different use cases. Being shameless and curious, I first tried them with something that Jelly was built for – streaming IoT data. I took the [`assist-iot-weather-graphs`](https://w3id.org/riverbench/datasets/assist-iot-weather-graphs/1.0.3) dataset from RiverBench. The dataset is a series of weather observations, with each measurement being a named RDF graph with a bunch of triples describing the sensors, wind speed, temperatures, and so on (you can find samples on [its description page](https://w3id.org/riverbench/datasets/assist-iot-weather-graphs/1.0.3) in RiverBench).

The code for the experiments is **[here](https://github.com/Ostrzyciel/jelly-cbor-ld/blob/072f69920978f4e98c4ab3311c8135f0b27cd366/src/main/java/eu/ostrzyciel/experiments/jelly_cbor_ld/Main.java)**. It uses the [new shiny integration of Jelly-JVM with the Titanium RDF API](https://w3id.org/jelly/2.9.x/user/titanium/). The new [Titanium API](https://github.com/filip26/titanium-rdf-api) is intended to be a minimalistic bridge between different RDF libraries and I think it does a great job at that – linking Jelly-JVM to Titanium JSON-LD was a breeze.

I first needed to encode the data in JSON-LD, and then compact it. For that I needed a context. Now – I'm bad at JSON-LD, so I used the context creation algorithm that Apache Jena uses, which is to simply use all explicitly defined prefixes and build a context like this:

```json
{
  "@context": {
    "aiot": "https://assist-iot.eu/ontologies/aiot#",
    "aiot_p2": "https://assist-iot.eu/ontologies/aiot_p2#",
    "geosparql": "http://www.opengis.net/ont/geosparql#",
    "om": "http://www.ontology-of-units-of-measure.org/resource/om-2/",
    "pilot": "https://assist-iot.eu/pilot2_rdf/",
    "prov": "http://www.w3.org/ns/prov#",
    "rdfs": "http://www.w3.org/2000/01/rdf-schema#",
    "sosa": "http://www.w3.org/ns/sosa/",
    "ssn": "http://www.w3.org/ns/ssn/",
    "xsd": "http://www.w3.org/2001/XMLSchema#"
  }
}
```

Note that this is likely not even close to being optimal, but it is simple to build and analogous to Turtle. Also note that Jelly does not need you to manually specify the prefixes, it figures them out automatically from the raw data.

I then put the context under a local URL (in a file on disk), so that the context is just an URL reference, and compressed it with CBOR-LD.

Here are the results, in bytes:

{: .table .table-sm}
| Stream element | Jelly | JSON-LD | JSON-LD (compacted) | CBOR-LD |
| -- | --: | --: | --: | --: |
| 0 | 3929 | 13321 | 6735 | 5273 |
| 1 | 2259 | 13321 | 6735 | 5273 |
| 2 | 2261 | 13321 | 6735 | 5273 |
| 3 | 2260 | 13320 | 6734 | 5272 |

One thing I didn't mention is that Jelly can work with RDF streams, so it is able to encode multiple RDF datasets (in this case, 1 dataset = 1 weather observation) as a single stream. This is why the first frame in Jelly's stream is larger – it contains the prefixes and other lookup entries. The subsequent frames are much smaller, because they only contain the dictionary references and literals.

CBOR-LD seems to be much worse here, but like I mentioned, I did a very bad job at preparing the JSON-LD context. If I used something really fine-tuned to the kind of data I'm working with, the results would be much better (probably better than Jelly!). If someone wants to try their hand at this, I would be very interested in updating the blogpost with them. I think it should be possible to get it down below 1000 bytes.

## Experiment 2 – Verifiable Credentials

Now, this is what CBOR-LD was really built for. [Verifiable Credentials (VCs)](https://w3c-ccg.github.io/vc-barcodes/) contain information about, for example, your driver's license or vaccination status. They are meant to be stored in QR/barcodes or NFC tags, so the smaller the better.

The [`ld-cli` tool](https://github.com/filip26/ld-cli?tab=readme-ov-file#custom-cbor-ld-dictionaries) has a great example of something like this – only 145 bytes for the whole thing! I wanted to see how Jelly would do with it.

The code for this experiment is in [the same file as before](https://github.com/Ostrzyciel/jelly-cbor-ld/blob/072f69920978f4e98c4ab3311c8135f0b27cd366/src/main/java/eu/ostrzyciel/experiments/jelly_cbor_ld/Main.java).

{: .table .table-sm}
Stream element | Jelly | JSON-LD (compacted) | CBOR-LD
-- | --: | --: | --:
0 | 1337 | 1098 | 145
1 | 331 | 1098 | 145
2 | 331 | 1098 | 145

Remember, CBOR-LD is putting essentially the entire dictionary (IRIs and stuff) in the context, which is not part of the message. In Jelly, I ran the same VC through the encoder a few times, to get a similar effect as with the IoT data. The first frame is larger, because it contains the prefixes and dictionary entries, but the subsequent frames are much smaller.

So – if you treated the first Jelly frame as a "context" of sorts that is pre-shared between the producer and the consumer, you would get a very similar mechanism as in CBOR-LD. In that case I'd say that the comparison between the `331` bytes of Jelly to the `145` bytes of CBOR-LD is a fair one.

I also tried applying binary compression (gzip and zstd) to the outputs ([see the code](https://github.com/Ostrzyciel/jelly-cbor-ld/blob/072f69920978f4e98c4ab3311c8135f0b27cd366/src/main/java/eu/ostrzyciel/experiments/jelly_cbor_ld/Compression.java)):

{: .table .table-sm}
Binary compression | Jelly (el. 1) | JSON-LD (compacted) | CBOR-LD 
-- | --: | --: | --:
None | 331 | 1098 | 145
gzip (level 6) | 274 | 549 | 168
zstd (level 19) | 264 | 552 | 157

Setting higher compression levels did not make the files smaller. We can see that you can shave off a few bytes from the Jelly file, but it's still far from being as compact as CBOR-LD. On the other hand, the CBOR file gets a bit larger, because the data is already very tightly compressed.

## Concluding thoughts

Jelly and CBOR-LD are both binary RDF formats, but with very different goals, use cases, and strengths.

CBOR-LD can be really, really compact, if you put in the work of defining a good context. Jelly does everything automatically for you ("one size fits all"), but is not as compact. There are [things that could be improved](https://github.com/Jelly-RDF/jelly-protobuf/issues/14) in Jelly to make it a bit smaller (and I'm working on it!), but it often does require sacrificing some interoperability or speed.

On the very far extreme end, you could define a binary format that has a context that's a pattern of triples with a couple of blanks to fill in (like a template). The binary representation would be just the serialized field contents, in sequence. Literally zero overhead. Flexibility would be zero as well, but it would be the most compact format possible. At this point I start to wonder if it wouldn't make sense to just use a dedicated binary format for the specific use case, and then link it to an [RML template](https://rml.io/) or something like that. 😆 Quite honestly, I think that this kind of approach may make quite a lot sense for narrowly-defined messages – from what I understand this is similar to what the [SmartEdge EU project](https://www.smart-edge.eu/) is working on.

The point is that there is always a trade-off between compactness, speed, and flexibility. Jelly is optimized for speed and flexibility, while CBOR-LD is optimized for compactness. The choice of format depends on the use case. I think CBOR-LD is an excellent idea overall, but one with relatively narrow use cases. There is also the issue of how to keep track of the registered contexts, and the related security implications – this was brought up at the last W3C JSON-LD CG meeting. I will keep working on making Jelly a bit more compact, but there is only so much that you can do without sacrificing the other properties.

Caveat: I did not compare speed here, at all. While Jelly is [insanely fast](https://w3id.org/jelly/dev/performance/), especially for IoT data, I would expect CBOR-LD to be way slower, because it's another layer of complexity on an already complex format (JSON-LD). I guess that's a topic for future blogpost.

## Shout-outs

Many thanks to [filip26](https://github.com/filip26/) for his help in explaining to me how CBOR-LD works and how to use it. He made the amazing [Titanium RDF API](https://github.com/filip26/titanium-rdf-api) I used to convert between Jelly and CBOR-LD, the [iridium-cbor-ld library](https://github.com/filip26/iridium-cbor-ld), and the [`ld-cli` tool](https://github.com/filip26/ld-cli), which was super helpful.

Also, shout out to the [W3C JSON-LD Community Group](https://www.w3.org/groups/cg/json-ld/), through which I got to know CBOR-LD better!
