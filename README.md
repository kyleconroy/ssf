# Standard Sensor Format

This is the definition of the Standard Sensor Format, a modern metrics format
with the following goals:

* Familiarity due to similarities to [StatsD](https://github.com/etsy/statsd) fields like name and sample rate
* Orthogonal tagging for many dimensions
* Inclusion of unit for improved ergonomics after storage
* An "event" type for aperiodic, human-readable stuff
* Optional trace fields for use with tracing sytems
* Status code for signaling health

It is so named because it's intent is to represent the samples from sensors
we use to make our systems [observable](https://en.wikipedia.org/wiki/Observability).

## Example

```json
{
  "metric": "c",
  "name": "page.views",
  "value": 1,
  "sample_rate": 0.5,
  "tags": [
    "foo:bar"
  ],
  "status": 0,
  "message": "Some sort of string message here!",
  "unit": "page",
  "trace": {
    "parent_id": 123457,
    "id": 123456
  }
}
```

As Protobuf:
```protobuf
syntax = "proto3";

message Trace {
  required double id = 1;
  optional double parent_id = 2;
}

message Sample {
  enum Metrics {
      COUNTER = 0;
      GAUGE = 1;
      HISTOGRAM = 2;
      SET = 3;
      EVENT = 4;
  }
  optional Metrics metric = 1 [default = COUNTER];
  required string name = 2;
  optional int32 status = 3 [default = 0];
  optional double value = 4;
  optional float sample_rate = 5 [default = 1.0];
  repeated string tags = 6;
  optional string unit = 7;
  optional Trace trace = 8;
}
```

# TODO
* Versioning?
* Write up goals (combining metrics, logging, events, tracing and service checks)
* Settle on a serialization format. Leaning toward MessagePack of Protobuf.
* Way of representing intermediary tag munging? (aggregators gonna agg)
* Work out the whole MTU/UDP/log-lines-are-long thing
* Settle if these should be separate objects or one big one?
* Add way for "extensions" to be added ^
* Creation of a CLI tool for emitting a metric to a port a la `nc`
* Define valid tag characters. (colons as delimiter?)
* Decide if there's any real difference in a histogram and a timer

# Why?

[StatsD](https://github.com/etsy/statsd) is great. Please use it!

StatsD is also a fractured standard. Many implementations have extended it.
Here we attempt to capture many of the extensions as well as include a few novel, new ideas.

If, like me, you are eager for a protocol that has more features, this might be for you.

# Fields

Here are the fields in the document:

## Metric (required)

The key `metric` must be one of the integer values:
* `0` for counter
* `1` for gauge
* `2` for histogram
* `3` for set
* `4` for event

## Name (required)
Required: A string name for the metric, event or whatever in storage.

Names are likely to be in the form of a dotted schema, such as those
popularized by Graphite and other tools. The inclusion of tags,
however, means that the names are considerably simpler as other information
like hostnames, device names and paths will be part of the tags.

## Value (required for most types)

The value for the measurement. It is required for:

* counter
* gauge
* histogram

## Sample Rate (optional)

A floating point representation of the probability that this metric is being sampled at. If, for example
the metric is being emitted only 50% of the time then the value for `sample_rate` shall be `0.5`.

If no `sample_rate` is supplied then server implementations must assume `1.0`.

## Message (required for set, optional otherwise)

The `message` field is a string of arbitrary length meant to provide context or human-readable
information around a metric or an event. It is also used as the input for a set metric to avoid
the complication of a type union for `value`.

A common use of this field is as a replacement for a log line.

## Tags

Tags are arbitrary strings providing independent dimensions for further differentiating a metric. Such "orthogonal"
tags are widely used and explained in other systems, so that explanation will be skipped here.

SSF does not take a stance on tags meaning anything. They are just arbitrary data. Fields that have specific meaning
to SSF are represented as properties in this spec, rather than in tags.

The field `tags`, if present, must contain an array with an arbitrary number of strings.

## Status (optional)

The key `status` must be one of the integers:
* `0` for OK
* `1` for WARNING
* `2` for CRITICAL
* `3` for UNKNOWN

If no `status` is supplied then server implementations must assume `0`.

The interpretation of these statuses is the responsibility of the user, meaning
that what you mean by `WARNING` is likely specific to your contexst.

## Unit (required for all but set and event)

The field `unit` must contain a string. The interpretation of these values is the responsibility
of the user, but they are expected to be one of:
* A unit such as a `byte` or `meter` with or without a prefix
* An arbitrary string such as `pages` that is used for an informational label

# Tracing (optional)

Tracing information is optionally stored in the `trace` key. It contains an
object with required `id` and an optional `parent_id`.

## Trace ID

The field `id` may be present and contain an integer as part of a tracing system to uniquely
identify the sample as part of a trace.

## Parent Trace ID (optional)

The field `parent_id` may be present and contain an integer as part of a tracing system to make
this sample the child of another sample.

# Examples

## Counter

```json
{
  "metric": "c",
  "name": "page.views",
  "value": 1,
  "sample_rate": 0.5,
  "tags": {
    "path": "/foo"
  },
  "unit": "page"
}
```

## Event

```json
{
  "metric": "e",
  "name": "build.deployed",
  "tags": {
    "destination": "server1.example.com",
    "deployer": "gphat"
  }
}
```

## Gauge

```json
{
  "metric": "g",
  "name": "heap.total_available",
  "value": 8,
  "tags": {
    "version": "1.8.0_66"
  },
  "unit": "GiB"
}
```

## Histogram

```json
{
  "metric": "g",
  "name": "database.key_retrieval_duration",
  "value": 8,
  "tags": {
    "schema": "people"
  },
  "unit": "ms"
}
```

## Set

```json
{
  "metric": "s",
  "name": "customer.exceptions_received",
  "message": "abc12345",
  "tags": {
    "exception_name": "NoSuchMethodException"
  },
  "unit": "exception"
}
```

# Other Suggestions

## Use verbose names

When naming a metric, think of it as an English sentence: "This metric measures…".

For example, if measuring pages, one might end up with "This metric measures pages viewed" resulting in a
metric name of `pages.viewed`. It may seem a bit burdensome to write such long metric names, but
remember that these metrics are meant to be consumed in monitoring and analytic systems and therefore are
read and interpreted more of than they are typed in to your codebase.
