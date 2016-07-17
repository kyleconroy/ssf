This is the definition of a modern metrics format with the following attributes:
* Standard [statsd](https://github.com/etsy/statsd) fields like name and sample rate
* JSON formatted for ease of generation and consumption
* Inclusion of units for improved ergonomics after storage
* Orthogonal tagging for many dimensions
* An "event" type for aperiodic, human-readable stuff
* Optional trace fields for use with tracing sytems
* Status code for signaling health

## Example

```json
{
  "metric": "c",
  "name": "page.views",
  "value": 1,
  "sample_rate": 0.5,
  "tags": {
    "foo": "bar"
  },
  "status": 0,
  "message": "Some sort of string message here!",
  "unit": "page",
  "parent_trace_id": 123456,
  "trace_id": 123457
}
```

# Why?

[StatsD](https://github.com/etsy/statsd) is great. Please use it!

But if, like me, you are eager for a protocol that has more features, this might be for you.

It's also, however, a bit of a fractured standard. Many implementations have extended it.
Here we attempt to capture many of the extensions as well as include a few novel, new ideas.

## Why JSON?

Because it's easy. StatsD's simplicity is large factor in it's success. To aid in correctness
and easy of parsing, JSON seems a reasonable standard!

Compare the simplest of StatsD messages:

```
page.views:1|c
```

with:
```
{"metric:counter","name":"page.views","value":1,"unit":"page"}
```

The JSON format is still very approachable typed out by hand or emitted using string-interpolation
in every programming language I can think of. It is also more verbose, which is good and bad. Bad
because it's more to type and more data on the wire, but good because it's more obvious to future
readers, provides a unit and allows for more features!

## Why not $OTHER_FORMAT

I have regularly used tricks like [nc](https://en.wikipedia.org/wiki/Netcat) and raw UDP sockets
as a way to emit simple StatsD datagrams and requiring a binary format was a non-starter for me,
it would make such methods harder. We want instrumentation to be as easy as possible!

# Fields

Here are the fields in the document:

## Metric (required)

The key `metric` must be one of the strings:
* `c` for counter
* `e` for event
* `g` for gauge
* `h` for histogram
* `s` for set

TODO: Include timer separate from histogram?

## Name (required)
Required: A string name for the metric, event or whatever in storage.

## Value (required for mist types)

The value for the measurement. It is required for:

* counter
* gauge
* histogram

## Sample Rate (optional)

A floating point representation of the probability that this metric is being sampled at. If, for example
the metric is being emitted only 50% of the time then the value for `sampe_rate` shall be `0.5`.

If no `sample_rate` is supplied then server implementations must assume `1.0`.

## Message (required for set, optional otherwise)

The `message` field is a string of arbitrary length meant to provide context or human-readable
information around a metric or an event. It is also used as the input for a set metric to avoid
the complication of a type union for `value`.

## Tags

Tags are name-value pairs providing independent dimensions for further differentiating a metric. Such "orthogonal"
tags are widely used and explained in other systems, so that explanation will be skipped here.

The field `tags`, if present, must contain an object with an arbitrary number of fields. Each field and it's value
must be a string.

## Status (optional)

The key `status` must be one of the integers:
* `0` for OK
* `1` for WARNING
* `2` for CRITICAL
* `3` for UNKNOWN

If no `status` is supplied then server implementations must assume `0`.

The interpretation of these statuses is the responsibility of the user.

## Unit (required for all but set and event)

The field `unit` must contain a string. The interpretation of these values is the responsibility
of the user, but they are expected to be one of:
* A unit such as a `byte` or `meter` with or without a prefix
* An arbitrary string such as `pages` that is used for an informational label

# Trace ID (optional)

The field `trace_id` may be present and contain an integer as part of a tracing system to uniquely
identify the sample as part of a trace.

# Parent Trace ID (optional)

The field `parent_trace_id` may be present and contain an integer as part of a tracing system to make
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
