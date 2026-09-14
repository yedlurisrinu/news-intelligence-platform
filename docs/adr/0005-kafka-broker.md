# ADR-0005: Local Kafka for development, MSK for AWS

- **Status:** Accepted
- **Date:** 14-Sep-2026

## Context

The news pipeline is event-driven. A fetch scheduler produces articles and
a consumer ingests them. Both services were built against Confluent Cloud
with SASL_SSL authentication.

Two things made that worth revisiting. Development against a hosted broker
means no offline work and a dependency on a third-party account for local
testing. And the program's cloud environment is AWS, where the managed
Kafka service is MSK.

Apache Kafka 4.0, released March 2025, removed ZooKeeper entirely. KRaft
is now the only metadata mode, which makes a single-node local broker one
container rather than two.

## Options considered

### A — Keep Confluent Cloud for both development and deployment
No broker to operate. Costs: local development requires network access and
a live account; the deployment target diverges from the rest of the
program's AWS footprint; and cost control sits outside the AWS budget that
governs everything else.

### B — Local broker for development, Confluent Cloud for deployment
Offline development, but two managed-service relationships and a cloud
deployment outside AWS.

### C — Local broker for development, MSK for AWS
One cloud provider, one budget, one set of IAM primitives.

## Decision

**Option C.** A single-node Apache Kafka broker in KRaft mode runs locally
in `docker_services`; MSK is the AWS target.

The broker image is `apache/kafka` rather than a vendor distribution.
Configuration maps directly to Kafka's own documentation with no
vendor-specific translation layer, which matters when the purpose is to
learn the platform.

Kafbat UI provides message and topic inspection. Provectus, the original
maintainer of the project it forked from, paused active development; the
fork is maintained by contributors from the original project.

The client library remains `confluent-kafka-python`. It wraps librdkafka,
the reference C implementation of the Kafka protocol, and speaks the open
protocol against any conforming broker. Confluent's authorship of the
library does not bind the broker choice.

Kafka lives in `docker_services` rather than the agent platform kit. It is
shared infrastructure consumed by separate service repositories; the kit's
compose holds only what the kit itself requires. This supersedes the
project conventions document, which listed Kafka in the kit's stack.

## Consequences

- Development works offline and costs nothing.
- Client configuration must become conditional. Local is PLAINTEXT; MSK
  uses TLS with IAM or SASL authentication. The security protocol,
  mechanism and credentials cannot remain hardcoded. Deferred to the
  threads that rewrite each service.
- A single node cannot exercise replication behaviour. Leader election,
  in-sync replica shrink, and `acks=all` semantics under broker failure
  are not observable locally. Producer behaviour tuned against a
  single-replica topic may differ on MSK, where topics span availability
  zones.
- Internal topic replication factors are overridden to 1. The defaults of
  3 cannot be satisfied by one broker, and the failure is delayed and
  misleading: the broker starts and appears healthy, then fails when a
  consumer group first attempts to commit an offset, because
  `__consumer_offsets` was never created.
- Two listeners are required. Containers on the shared network reach the
  broker at one address and host processes at another, and a listener can
  advertise only one address.
- Trace context must be propagated through Kafka headers for a single
  trace to span producer and consumer. Without it the pipeline produces
  disconnected traces per service. Not yet implemented.

## Verification

Topic creation, production and consumption confirmed through the broker's
own CLI inside the container, and independently through the UI, which
connects over the internal listener.

The host listener was verified separately with an admin client from the
host. The broker metadata returned `localhost:29092` — the address correct
for a host client rather than the container address — confirming that each
listener advertises the address appropriate to its audience.

That second check is the one worth keeping. A client bootstraps, receives
cluster metadata naming an address, and reconnects to whatever it is told.
A misconfigured advertised address produces a successful initial
connection followed by failures against an address the client was never
configured with.