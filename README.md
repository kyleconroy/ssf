# Standard Sensor Format

This is the definition of the Standard Sensor Format, a modern metrics format
with the following goals:

* Familiarity due to similarity to [StatsD](https://github.com/etsy/statsd) fields like name and sample rate
* Orthogonal tagging for many dimensions
* Inclusion of unit for improved ergonomics after storage
* An "event" type for aperiodic, human-readable stuff
* Optional trace fields for use with tracing sytems
* Status code for signaling health

It is so named because it's intent is to represent the samples from sensors
we use to make our systems [observable](https://en.wikipedia.org/wiki/Observability).

## Example

```protobuf
syntax = "proto2";

package main;

message SSFTag {
  required string name = 1;
  optional string value = 2;
}

message SSFTrace {
  required int64 trace_id = 1;
  required int64 id = 2;
  required int64 duration = 3;
  optional int64 parent_id = 4;
}

message SSFSample {
  enum Metric {
      COUNTER = 0;
      GAUGE = 1;
      HISTOGRAM = 2;
      SET = 3;
      STATUS = 4;
      EVENT = 5;
      TRACE = 6;
  }
  enum Status {
      OK = 0;
      WARNING = 1;
      CRITICAL = 2;
      UNKNOWN = 3;
  }
  optional Metric metric = 1 [default = COUNTER];
  required string name = 2;
  required int64 timestamp = 2;
  optional string message = 3;
  optional Status status = 4 [default = OK];
  optional double value = 5;
  optional float sample_rate = 6 [default = 1.0];
  repeated SSFTag tags = 7;
  optional string unit = 8;
  optional SSFTrace trace = 9;
}
```

# TODO
* Versioning?
* Write up goals (combining metrics, tracing and service checks)
* Way of representing intermediary tag munging? (aggregators gonna agg)
* Work out the whole MTU/UDP/log-lines-are-long thing
* Duration in trace needs a unit (ns?)
* Add mechanism for "extensions"
* Creation of a CLI tool for emitting a metric to a port a la `nc`
* Influx fields
* OpenTracing baggage
* Should `status` be log levelesque instead of status checky?

# Why?

[StatsD](https://github.com/etsy/statsd) is great. Please use it!

StatsD is also a fractured standard. Many implementations have extended it.
Here we attempt to capture many of the extensions as well as include a few novel, new ideas.

If, like us, you are eager for a protocol that has more features, this might be for you.

# Examples

SSF can represent the following:

* A metric sample a la StatsD by using `name`, `timestamp`, `value` and optional `sample_rate`, `tags` and `unit`.
* A trace span using `name`, `timestamp`, `trace_id`, `id`, `duration` and optional `parent_id`, `status` and `tags`.
* A status check via `name`, `timestamp` and `status` with optional `value` and `tags`.

# Attributes

Here are the attributes in an SSF sample:

## Metric (required)

The key `metric` must be one of the integer values:
* `0` for counter
* `1` for gauge
* `2` for histogram
* `3` for set
* `4` for status check
* `5` for an event
* `6` for a trace

Note there is no "timer" type, because SSF treats timers as histograms. The downstream
storage can decide how to interpret them based on the unit.

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

The sample rate may be used more than traditional metrics. Tracing systems may use this as a form of weighting
to determine if trace should be recorded.

## Message (required for set, optional otherwise)

The `message` field is a string of arbitrary length meant to provide context or human-readable
information around a metric or an event. It is also used as the input for a set metric to avoid
the complication of a type union for `value`.

A common use of this field is as a replacement for a log line.

## Tags

Tags are an arbitrary number of string pairs providing independent dimensions for further differentiating
a metric. Such "orthogonal" tags are widely used and explained in other systems, so that explanation will
be skipped here.

SSF does not take a stance on tags meaning anything. They are just arbitrary data. Fields that have specific meaning
to SSF are represented as properties in this spec, rather than in tags.

SSF also allows keys to be either a key or a key and value pair. If your backend does not support key value pairs,
feel free to ignore.

## Status (optional)

The key `status` must be one of the integers:
* `0` for OK
* `1` for WARNING
* `2` for CRITICAL
* `3` for UNKNOWN

If no `status` is supplied then server implementations must assume `0`.

The interpretation of these statuses is the responsibility of the user, meaning
that what you mean by `WARNING` is likely specific to your context.

## Unit (required for all but set and event)

The field `unit` must contain a string. The interpretation of these values is the responsibility
of the user, but they are expected to be one of:
* A unit such as a `byte` or `meter` with or without a prefix
* An arbitrary string such as `pages` that is used for an informational label

# Tracing (optional)

Tracing information is optionally stored in the `trace` key.

## Trace ID

The field `trace_id` represents the id of the overall trace that this span is a part of.

## Span Id

The field `id` may be present and contain an `int64` as part of a tracing system to uniquely
identify the sample as part of a trace.

## Parent Trace ID (optional)

The field `parent_id` may be present and contain an `int64` as part of a tracing system to make
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
