---
title: Native Histograms
sort_rank: 5
---

// Final completion check by going through:
// - The design docs.
// - Grafana docs.
// - open issues

# Native Histograms

Native histograms were introduced as an experimental feature in November 2022.
They are a concept that touches almost every part of the Prometheus stack. The
first version of the Prometheus server supporting native histograms was
v2.40.0. The support had to be enabled via a feature flag
`--enable-feature=native-histograms`. (TODO: This is still the case with the
current release v2.49. Update this section with the stable release, once it has
happened.)

Due to the pervasive nature of the native-histogram related changes, the
documentation of those changes and explanation of the underlying concepts are
widely distributed over various channels (like the documentation of affected
Prometheus components, doc comments in source code, sometimes the source code
itself, design docs, conference talks, …). This document intends to gather all
these pieces of information and present them all in a unified context. This
document prefers to link existing detailed documentation rather than restating
it, but it contains enough information to be comprehensible without referring
to other sources.

While formal specifications are supposed to happen in their respective context
(e.g. OpenMetrics changes will be specified in the general OpenMetrics
specification), some parts of this document take the shape of a specification.
In those parts, the key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL
NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” are used as
described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

The key word TODO may refer to either of the following:
- Incomplete parts of this document.
- Incomplete implementation of intended native-histogram related changes.

## Introduction

The core idea of native histograms is to treat histograms as first class
citizens in the Prometheus data model. This approach unlocks the features
described in the following, which is the reason why they are called _native
histograms_.

Previously, all Prometheus sample values had been 64-bit floating point values
(short _float64_ or just _float_). These floats can directly represent _gauges_
or _counters_. The Prometheus metric types _summary_ and _histogram_, as they
exist in exposition formats, are broken down into float components upon
ingestion: A _sum_ and a _count_ component for both types, a number of
_quantile_ samples for a summary and a number of _bucket_ samples for a
histogram.

With native histograms, a new structured sample type is introduced. A single
sample represents the previously known _sum_ and _count_ plus a dynamic set of
buckets.

Native histograms have the following key properties:
1. A sparse bucket representation, allowing (near) zero cost for empty buckets.
2. Coverage of the full float64 range of values.
3. No configuration of bucket boundaries during instrumentation.
4. Dynamic resolution picked according to simple configuration parameters.
5. A sophisticated exponential bucketing schema, ensuring mergeability between
   all histograms.
6. An efficient data representation for both exposition and storage.

Compared to the previously existing “classic” histograms, native histograms
allow a higher bucket resolution across arbitrary ranges of observed values at
a lower storage and query cost with very little to no configuration required.
Even partitioning histograms by labels is now much more affordable.

Because the sparse representation (property 1 in the list above) is so crucial
for many of the other benefits of native histograms that_sparse histograms_ was
a common name for _native histograms_ early during the design process. However,
other key properties like the exponential bucketing schema or the dynamic
nature of the buckets are also very important, but not caught at all in the
term _sparse histograms_.

### Design docs

These are the design docs that guided the development of native histograms.
Some details are obsolete now, but they describe rather well the underlying
concepts and how they evolved.

- [Sparse high-resolution histograms for
  Prometheus](https://docs.google.com/document/d/1cLNv3aufPZb3fNfaJgdaRBZsInZKKIHo9E6HinJVbpM/edit),
  the original design doc.
- [Prometheus Sparse Histograms and
  PromQL](https://docs.google.com/document/d/1ch6ru8GKg03N02jRjYriurt-CZqUVY09evPg6yKTA1s/edit),
  more an exploratory document than a proper design doc about the handling of
  native histograms in PromQL.

### Conference talks

A more approachable way of learning about native histograms is to watch
conference talks, of which a selection is presented below. As an introduction,
it might make sense to watch these talks and then return to this document to
learn about all the details and technicalities.

- [Secret History of Prometheus
  Histograms](https://fosdem.org/2020/schedule/event/histograms/) about the
  classic histograms and why Prometheus kept them for so long.
- [Prometheus Histograms – Past, Present, and
  Future](https://promcon.io/2019-munich/talks/prometheus-histograms-past-present-and-future/)
  is the inaugural talk about the new approach that lead to native histograms.
- [Better Histograms for
  Prometheus](https://www.youtube.com/watch?v=HG7uzON-IDM) explains why the
  concepts work out in practice.
- [Native Histograms in
  Prometheus](https://promcon.io/2022-munich/talks/native-histograms-in-prometheus/)
  presents and explains native histograms after the actual implementation.
- [PromQL for Native
  Histograms](https://promcon.io/2022-munich/talks/promql-for-native-histograms/)
  explains the usage of native histograms in PromQL.
- [Prometheus Native Histograms in
  Production](https://www.youtube.com/watch?v=TgINvIK9SYc) provides an analysis
  of performance and resource consumption.
- [Using OpenTelemetry’s Exponential Histograms in
  Prometheus](https://www.youtube.com/watch?v=W2_TpDcess8) covers the
  interoperability with the OpenTelemetry.

## Glossary

- A __native histogram__ is the new complex sample type representing a full
  histogram that this document is about.
- A __classic histogram__ is the older sample type representing a histogram
  with fixed buckets, formerly just called a _histogram_. It exists as such in
  the exposition formats, but is broken into a number of float samples upon
  ingestion into Prometheus.
- __Sparse histogram__ is an older, now deprecated name for _native
  histogram_. This name might still be found occasionally in older
  documentation. __Sparse buckets__ remains a meaningful term for the buckets
  of a native histogram.

## Data model

### General structure

Similar to a classic histogram, a native histogram has a _count_ of
observations and a _sum_ of observations field. In addition, it contains the
following components, which are described in detail in dedicated sections
below:

- A _schema_ to identify the method of determining the boundaries of any given
  bucket with an index _i_.
- A sparse representation of indexed buckets, mirrored for positive and
  negative observations.
- A _zero bucket_ to count observations close to zero.
- _Exemplars_.

### Flavors

Any native histogram has a specific flavor along two independent dimensions:

1. Counter vs. gauge: Usually, a histogram is “counter like”, i.e. each of its
   buckets acts as a counter of observations. However, there are also “gauge
   like” histograms where each bucket is a gauge, representing arbitrary
   distributions at a point in time. The concept was previously introduced for
   classic histograms by
   [OpenMetrics](https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md#gaugehistogram).
2. Integer vs. floating point (short: float): The obvious use case of
   histograms is to count observations, resulting in positive integer numbers
   of observations within each bucket, including the _zero bucket_, and for the
   total _count_ of observations, represented as unsigned 64-bit integers
   (short: uint64). However, there are specific use cases leading to a
   “weighted” or “scaled” histogram, where all of these values are represented
   as 64-bit floating point numbers (short: float64). Note that the _sum_ of
   observations is a float64 in either case.

// When does it happen.

// Counter nature of count vs sum with negative observations.

// down to details: Overflow, NaNs, resolution limit, schema extension.

### Schema

### Buckets

### Zero bucket

### Exemplars

### OpenTelemetry interoperability

## Exposition formats

### Classic Prometheus formats

// Only proto
// Combined classic/native, how to mark observation-less native histograms
// TODO: exemplars

### OpenMetrics

## Instrumentation libraries

// Both usage as well as implementation details, e.g. observing NaNs.
// bucket limits
// partition by labels at will, why feasible
// OTel approach: "infinite" resolution, going down

## Scrape configuration

To enable the Prometheus server to scrape native histograms, the feature flag
`--enable-feature=native-histograms` is required. This flag also changes the
content negotiation to prefer the classic protobuf-based exposition format over
the OpenMetrics text format. (TODO: This behavior will change once native
histograms are a stable feature.)

### Finetuning content negotiation

### Limit bucket count and resolution

### Scrape both classic and native histograms

## TSDB

// Mixed series.
// Staleness.
// When to cut a new chunk.

### Counter reset considerations

### Out-of-order support

## PromQL

// always float (but MAY re-convert to int for storage, remote-write)
// merging
// counter reset handling
// including for _sum_

### Recording rules

### Alerting rules

### Testing framework

## Prometheus query API

## Prometheus UI

// Note that Grafana support etc. is out of scope.

## Template expansion

## Remote write & read

## Pushgateway

## `promtool` 

This section describes `promtool` commands added or changed to support native
histograms.

## `prom2json`

## Migration considerations

// Scraping.
// Querying.
// Storing classic histograms as native histograms.

## Future extensions

### Custom bucket layouts

