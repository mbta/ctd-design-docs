- Feature Name: socket-proxy-kinesis-json
- Start Date: 2021-06-02
- RFC PR: mbta/technology-docs#0000
- Asana task: [Secure connection between socket_proxy and rtr](https://app.asana.com/0/881264583703207/1200290609745412/f)
- Status: Proposed

# Summary

Send OCS messages as JSON-formatted CloudEvents to AWS Kinesis

# Motivation

Currently, SocketProxy proxies OCS message to each RTR instance running in AWS. This method of sending Occupancy Control System (OCS) messages has several issues we are trying to address:
- insecure: the messages travel unencrypted over the public internet
- insecure: anyone from the MBTA network can forge OCS messages
- unreliable: if an RTR instance isn't running (crashed, or during a restart/redeploy) when a message comes in, that message is lost
- unreliable: if the production RTR instance isn't running, then the log of messages written to S3 (and used by OPMI) will not have those messages
- missing metadata: the messages do not include the date they were sent, or the time the messages were received by SocketProxy 

To address these issues, we'll update SocketProxy to write the OCS messages (along with additional metadata) as JSON CloudEvent messages to an AWS Kinesis stream. The partition key will be based on the type of message and the line.

# Guide-level explanation

AWS Kinesis "makes it easy to collect, process, and analyze real-time, streaming data so you can get timely insights and react quickly to new information." Unlike a message queue (SQS or RabbitMQ), messages written to AWS Kinesis can be repeatedly read by multiple clients.

Kinesis consists of named *streams*, and each stream consists of one or more *shards*. *Records* are read/written to the stream, and distributed between the shards based on a *partition key*. *Producers* write records to the stream, and are read by zero or more *consumers*. When written, each record receives a *sequence number*: this number is always increasing and uniquely identifies the record within the shard. It also orders the messages: messages within a shard have a defined order, but not messages received by two different shards.

In this case, SocketProxy will be the producer. For each message received from the OCS, it will transform that message into a CloudEvent (see [RFC 4](https://github.com/mbta/technology-docs/pull/4)), encode the event as JSON, and put a record in the "ctd-raw-ocs-messages" stream. The "ctd-raw-ocs-messages" stream will be configured with two (2) shards. In order that train movement messages are processed in the correct order, the partition key will be based on the type of OCS message and the line.

The four (4) RTR instances will be consumers of the stream. Instead of expecting messages on TCP sockets, they will read the CloudEvent JSON payload from the stream, parse out the raw message, and send it through the existing pipeline.

A [JSON Schema](https://json-schema.org/) document will be written in the `mbta/schemas` repository, with the name of the CloudEvent message.

# Reference-level explanation

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that:
- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.
The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

Currently, SocketProxy has a Receiver GenServer responsible for each incoming connection from OCS. Receiver then spawns a Forwarder GenServer to send the data to each of the destinations (currently, the four RTR instances). After this RFC, SocketProxy will have a KinesisReceiver which is responsible for writing records to a configured Kinesis stream:

```
SOCKET_PROXY_LISTEN_PORT=8000 SOCKET_PROXY_KINESIS_STREAM=ctd-ocs-raw-messages mix run --no-halt
```

The maximum size of a record (including the partition key) is 1MB, and record storage is charged in units of 25KB. Each shard supports writing 1MB or 1000 records per second For reading, each shard can support 5 read transactions per second, and up to 10,000 records per transaction, up to a total of 2MB per second. One (1) shard should be sufficient, but we will provision two (2) shards. While one shard should be sufficient, having multiple shards ensures that consumers properly handle multiple shards from the start.

By default, records are kept for 24 hours. This can be extended up to 7 days for an extended data retention stream, and 365 days for a long-term data retention stream.

This leads to a few possibilities for managing the state of the stream in RTR.

1. [discouraged] Always read from the latest message in the shard. This replicates the existing behavior when receiving messages directly from SocketProxy. However, this results in skipping messages during restarts.
2. [encouraged] The current sequence number for each shard is recorded as a part of the RTR state file in S3. When an instance restarts, it can continue from the last message it received.
3. [encouraged] Using the Kinesis Client Library (KCL) to maintain the current sequence number in DynamoDB. While KCL is a Java application, the Multi-Lang Daemon provides access to other languages include Elixir.
4. [potentially easier, more expensive] RTR stops maintaining state, and starts reading the Kinesis stream from the start of service whenever it starts. This would require increasing the default Kinesis storage above the default of 24 hours, and absorbing some additional cost to use extended data retention.

SocketProxy will write JSON-encoded CloudEvents to the Kinesis stream. JSON-encoded CloudEvents are somewhat larger than a binary encoding, but still easily fit within the limits of a Kinesis record. They are also human-readable, aiding in debugging.

To ensure that the format of the JSON messages are documented, a JSON Schema document will be added to a new repository: mbta/schemas. By documenting them in a GitHub repo, we have several advantages over a purpose-built schema registry:

- pull requests can be opened for new message types, providing documentation and discussion as new messages are added
- naming the schemas after the message types allows programmatic mapping of the message to the schema, for validation or other processing
For example, the schema for the "com.mbta.ocs.raw-message" schema could live at https://github.com/mbta/schemas/blob/HEAD/com.mbta.ocs.raw-message.schema.json

## Further Documentation

- Kinesis high-level overview: https://docs.aws.amazon.com/streams/latest/dev/key-concepts.html
- Kinesis Client Library (KCL): https://docs.aws.amazon.com/streams/latest/dev/shared-throughput-kcl-consumers.html
- Elixir Multi-Lang Daemon interface: https://github.com/AdRoll/exmld

- JSON Schema: https://jsonschema.org

# Drawbacks

Kinesis is a new technology that we're adding to our infrastructure. Even though it's mostly managed by Amazon, we'll need to develop some in-house expertise with building producers and consumers.

Running Kinesis locally is easy enough with tools like LocalStack or Kinesialite, but more involved than running a TCP proxy. This might require some re-working of how the RTR integration system works when pushing logged data into a local instance for testing.

# Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

AWS has tools to proxy messages from Kinesis into Kafka (and vis versa) so if we need to migrate in the future that is possible. If we had a lot more operational staff (or more experience with Kafka) this might point to a different decision, but I believe that Kinesis gets us the value we need with the minimum of cost and operational overhead.

## Managed Streaming for Kafka (MSK)

MSK  is the main managed alternative to Kinesis in AWS. [Kafka](https://kafka.apache.org/) is "an open-source distributed event streaming platform".

### Advantages
- more external software written to write to/consume Kafka
- vendor-neutral: Kafka can run on any cloud or on-premises
- pricing is based on cluster size and storage, so not as closely tied to the number of events
- messages have additional metadata, requiring slightly less overhead when using CloudEvents
- Being used by Cubic for Fare Transformation in Azure

### Disadvantages
- MSK cannot currently resize a cluster, so we either need to pay more up front, or pay an additional migration cost as we scale up
- requires additional REST proxy to send messages from outside the VPC


## Kafka (non-managed)

Non-managed Kafka requires significant additional operational overhead to manage (at least 6 servers per environment).

## Other potential encodings of CloudEvents

For example: Avro, Protobuf.

### Advantages
- slightly smaller encodings
- all messages must conform to a schema

### Disadvantages
- reading the data also requires importing the schema (can't do it ad-hoc)
- not human readable

# Prior art

<!-- Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:
- Can we learn something about this proposal from other projects in CTD?
- Can we learn something about this proposal from other transit agencies?
- Can we learn something about this proposal from past experiences in other jobs or projects?
This section is intended to encourage you as an author to think about the lessons from other places, and provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine.
Note that while precedent is some motivation, it does not on its own motivate an RFC. -->

Deutsche Bahn has been using Kafka in production for long enough that Confluent has written [case studies](https://www.confluent.io/blog/deutsche-bahn-kafka-and-confluent-use-case/) about it. Another agency in #transit-children also mentioned that they are using Kafka, but not for what. From my reading of the cases studies, the advantage is the approach (streaming data into one place) rather than the specific technology (Kafka).

Similarly to how we're currently standardized on GTFS-RT Enhanced for communicating transit data between applications, I hope that this pattern can be a model for future data integrations between CTD applications, as well as integrations between CTD and other MBTA departments.

# Unresolved questions

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->

- I expect that the specific implementation of state management will be resolved during development, rather than as a part of this RFC.
- Some of the specific names (the name of the CloudEvent message, the name of the Kinesis stream) may change, either based on RFC feedback or during development.

# Future possibilities

<!-- Think about what the natural extension and evolution of your proposal would be and how it would affect the project as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project in your proposal. Also consider how this all fits into the roadmap for the project.
This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.
If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.
Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information. -->

## Future TRC work

- Replace RTR's cache of messages in S3 with Kinesis Firehose
- Move all RTR instances into one ECS cluster once they don't need separate IPs

## Future work for other teams

- Split Busloc's processing into pieces mediated through Kinesis
- send Real-time Signs data through Kinesis