- Feature Name: socket-proxy-ocs-cloudevents
- Start Date: 2021-05-19
- RFC PR: [mbta/technology-docs#0004](https://github.com/mbta/technology-docs/pull/4)
- Asana task: [Secure connection between socket_proxy and rtr](https://app.asana.com/0/881264583703207/1200290609745412/f)
- Status: Accepted

# Summary
[summary]: #summary

When migrating how SocketProxy sends events, standardize on using the CloudEvent format.

# Motivation
[motivation]: #motivation

Standardizing the format of the events:

- Provides additional metadata
- Makes it easier for future applications to consume these events
- Provides a framework for other CTD applications
- Avoids bikeshedding of the format

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
OCS (Occupancy Control System) is responsible for sending messages indicating where the trains are, and their schedule. RTR uses these messages to generate VehiclePositions files (where the trains are) and TripUpdates files (when we predict the trains will be at their stops).

OCS sends an ad-hoc, CSV-like format. Parsing this data requires separate documention, and it doesn’t include valuable information such as the date the event was processed.

[CloudEvents](https://cloudevents.io) is "A specification for describing event data in a common way”. It provides some structure around events so that events being generated and processed by different applications can collaborate. Like using JSON:API for the V3 API, and GTFS-RT for transit data, using CloudEvents standardizes some basic structure to allow collaboration across teams.

SocketProxy (or a new application running on that server) will take the OCS messages, parse them into a CloudEvent, and push them to the cloud. The specific implementation of the push is out of scope and will be a separate RFC.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

An example TMOV message, as received from the OCS on 2020-02-11:

`9529,TMOV,02:00:40,G,10230,42.336251428162655,-71.14902308252195,1,831,25,  3885,gpsci,3885,0.00,254.19,3,42.336251428162655,-71.14902308252195`

The same TMOV message, as a CloudEvent representing the raw message (encoded in JSON):

```json
{
  "specversion": "1.0",
  "type": "com.mbta.ocs.raw_message",
  "source": "opstech3.mbta.com/socketproxy",
  "id": "63fdc3dcbb02acab21f74b3a9f88b898",
  "partitionkey": "TMOV-G",
  "time": "2020-02-11T02:00:40-05:00",
  "data": {
    "received_time": "2021-02-11T02:00:41.030-05:00",
    "raw": "9529,TMOV,02:00:40,G,10230,42.336251428162655,-71.14902308252195,1,831,25,  3885,gpsci,3885,0.00,254.19,3,42.336251428162655,-71.14902308252195"
  }
}
```

The fields outside the “data” key are defined by the CloudEvents spec (or an extension):

- `specversion` is the version of the CloudEvents spec that this event follows. If the spec evolves in the future, old events can still be parsed using this tag.
- `type` is the type of event that this represents. By giving it a namespace under “com.mbta.ocs” we can distinguish these events from other events (from OCS and other teams), even if they’re collected in one space. It also provides a schema for consumers of the messages.
- `source` represents where the event came from: for these events, they came from SocketProxy 1on Opstech3. While the value must be a URI-Fragment, it does not need to be a valid website.
- `id` is unique to each event from a source, and can be used to detect duplicates.
- `time` is when the event happened (or if not available, when the event was received). It includes a timezone to indicate a specific time across DST transitions.
- `partitionkey` is an optional extension, and helpful if consumers of the events want to balance the data across multiple shards/topics while keeping a relevant ordering.

Each message `type` will have a different structure for `data`. The specific format for each message will be determined by the developer(s) creating the message, but the structure and types of each message should be documented and stable.

If a message schema changes in the future, it receives a new message type. For example, if the raw message format changes, new messages would have a type of `com.mbta.ocs.raw_message.20210519` (with the date of the change) or `com.mbta.ocs.raw_message.v2`.

CloudEvents represent the platonic ideal of an event, and are agnostic towards how they are represented over the wire. All events can be represented losslessly as JSON blobs, but there are also protocol mappings:

- HTTP has message headers, leaving the HTTP body for `data`. It would be used in webhooks, for example.
- Kafka also has message headers which can capture the information outside of `data`, leaving the body of the Kafka message to be the `data`. It also has some native support for Avro, another possible encoding of a CloudEvent.

Standardizing on CloudEvents allows us to defer those decisions, and possibly make different decisions about encodings/protocols for different applications if required.

CloudEvent documentation:

- Primer: https://github.com/cloudevents/spec/blob/v1.0.1/primer.md
- Core specification: https://github.com/cloudevents/spec/blob/v1.0.1/spec.md
- JSON format: https://github.com/cloudevents/spec/blob/v1.0.1/json-format.md
- HTTP protocol binding: https://github.com/cloudevents/spec/blob/v1.0.1/http-protocol-binding.md
- Kafka protocol binding: https://github.com/cloudevents/spec/blob/v1.0.1/kafka-protocol-binding.md
- Avro format: https://github.com/cloudevents/spec/blob/v1.0.1/avro-format.md

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

## It increases the amount of data we send for each OCS message
For example, the full log of OCS messages for 2020-02-11 is 62M.

As JSON CloudEvents with a 32 hex-digit ID, it is 163M (2.63x).

## OPMI uses the CSV format...

...so we’ll need some way to continue writing those data to S3, either via SocketProxy or a new client of these messages.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Use only the raw OCS data, without the CloudEvent wrapper

This loses useful metadata, such as the source of the data and the time the message was received.

## Use an ad-hoc format to represent the additional metadata

Each team will re-invent their own message format, resulting in complications for systems such as LAMP which will want to process data from multiple systems.

# Prior art
[prior-art]: #prior-art

JSON:API is the standard we use for our V3 API. While we can’t use that specific standard here, it’s an example of us adopting a standard for our own applications.

Our current event standardization is that each application uploads a GTFS-RT Enhanced JSON file to a known location in S3. While we haven’t formally documented this specification, the general format of GTFS-RT (as a Protobuf) is standardized.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

This does not address whether the messages should be sent to AWS Kinesis Streams, Kafka, RabbitMQ, or some other event streaming sink: that will be resolved by a future RFC.

This does not address whether the messages use the Avro, JSON, Protobuf, or other format for the encoding of events: that will also be resolved by a future RFC.

This does not specify the specific format of the `source` field. The example uses a combination of the hostname and the application, which means the requirements of the [specification](https://github.com/cloudevents/spec/blob/v1.0.1/spec.md#source-1). Another possibility would be to indicate the team responsible for the application, rather than the host it is running on: `trc.ctd.mbta.com/socketproxy`

This does not currently address good values for the `id` field. However, these values are opaque to clients and can be changed arbitrarily. The example uses a short hash of the date, time, message type, and sequence, but a random UUID would also meet the requirements [https://github.com/cloudevents/spec/blob/v1.0.1/spec.md#id].

This does not currently address good values for the `partitionkey` field. The example uses the line and OCS message type as the partition key. This ensures that TMOVs for a given line are routed to the same shard. It’s not clear whether this provides enough data to spread load between many different partitions/shards, but this is likely event sink-specific.
- Extension documentation:  https://github.com/cloudevents/spec/blob/v1.0.1/extensions/partitioning.md#partitionkey

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, I expect that more teams will move to pushing events rather than uploading GTFS-RT files to S3. Standardizing on a generic format allows CTD (and possibly the MBTA overall) to build tools that work across a number of event producers.
