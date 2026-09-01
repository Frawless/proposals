# Kafka Exporter reimplementation

This proposal suggests to re-implement Kafka Exporter tool under Strimzi organization in Java.

## Current Situation

Strimzi currently downloads binaries from released version of [kafka_exporter](https://github.com/danielqsj/kafka_exporter) tool on GitHub.
The binaries are baked into Strimzi images and shipped as an optional tool for extending Kafka consumer metrics.

## Motivation

The project is widely used by Kafka community, however, it is not well maintained.
Latest release is from 2025-02-17 which makes it almost year and half old.
This opens space for potential security issues as there are CVEs that are not fixed.

This situation is also not much helpful for new features that we would like to add into the exporter.
Currently, there is 215 open issues and 58 open pull requests from the community with not much attention from the code owner.

## Proposal

To mitigate security problems and allow us to better maintain the tool and provide new features, we should re-implement Kafka Exporter tool.

We will create new repository under Strimzi organization - `strimzi/kafka-exporter` - that will contain the implementation code.

### Implementation

Because Strimzi mostly consists of Java projects, the proposal is to implement the tool in Java.
The implementation will use the minimal required dependencies and will contain all the existing features of the latest version of Kafka Exporter.

All the metrics that the tool will export will follow the same naming as the existing Kafka Exporter, but this can change in the future based on community feedback and needs.

To keep the minimal dependency tree we will use the following:
- `kafka-clients` for `AdminClient`
- `prometheus-metrics-core` and `prometheus-metrics-exposition-formats` for Prometheus endpoint
- JDK built-in http server for `/metrics` and `/healthz/ready` endpoints

### CI/CD

The tool will follow Strimzi standards and will adopt the same CI/CD workflow we use for other projects.
As an output of the build and release process, we will produce a tarball with a fat-jar that can be used in Strimzi images or in standalone distributions connected to Kafka.

### Versioning

The tool will follow Strimzi versioning `<major>.<minor>.<micro>` as other projects do.
The first version will be `0.1.0`, even though the tool already covers all existing Kafka Exporter functionality.
We will keep the tool in the 0.x line for one or two releases to gather feedback and confirm parity with the Go implementation in production before releasing the `1.0.0`.

### Documentation

The `strimzi/kafka-exporter` repository will include a README covering all configuration options, the full list of exported metrics, and instructions for running the tool standalone.
The Strimzi documentation will be updated to reflect the switch from the upstream Go binary to the new Java implementation.

### Testing

The new repository will include unit tests covering the metrics collection and registry logic, and integration tests running against a real Kafka instance.
Existing system tests in `strimzi-kafka-operator` will ensure that new implementation of Kafka Exporter continues to work end-to-end after the swap.
No additional e2e scenarios will be needed.

### Security

The new implementation will keep the same TLS and SASL/OAuth handling that Kafka Exporter has today, so the Kafka Exporter container keeps the same permissions and credential sources it already has, mounted certificates and Secrets provided by the Strimzi operator.
No credentials will be logged or persisted by the tool itself.
Moving to a Java implementation removes the current need to trust and verify pre-built third-party Go binaries and their checksums, replacing them with well known JVM dependencies that already go through Strimzi's existing CVE scanning and patching process.
The AdminClient will only need the same read-level Kafka permissions the current tool requires today, describing topics and consumer groups, so no privilege changes are introduced.

Any security improvements will be added once we will create repo with initial implementation.

## Affected Projects

This proposal affects only `strimzi-kafka-operator` repo.
Other sub-projects are not affected.

In the operator repository, we will need to change the make targets to copy the new artifacts from the `strimzi/kafka-exporter` repository into the Kafka image.
The existing `kafka_exporter_run.sh` script will be enhanced to launch the fat-jar instead of the Go binary with all necessary configs.
The possibilities to configure Kafka Exporter through Kafka CR will remain the same as today.

## Backwards Compatibility

This proposal is fully compatible with previous versions as new implementation will export all the same metrics as the original implementation.
The tool will also follow the same environment variables and parameters that are used for configuration of current Kafka Exporter.
Breaking removals of configuration options will only be possible in upcoming releases.

## Rejected Alternatives

### Fork Kafka Exporter to Strimzi org

We could simply fork the original Kafka Exporter and just fix CVEs.
However, this would require onboarding it to our build mechanisms and removing things we don't want, such as Helm charts.
These changes are straightforward, but we would still need to maintain Go code, which is not our team's expertise.

### Re-write Kafka Exporter in Go

Kafka Exporter is a small tool and Go is great for such tools, however, we do not have much experience across the maintainers team, and it might be hard to find an owner for it with proper experience.

### Re-write Kafka Exporter in Quarkus

Quarkus provides an easy way to write small tools that expose Prometheus metrics.
However, it also brings additional dependencies that can be easily avoided with pure Java.
The main advantage of Quarkus for a tool like this would be native image builds, but since Strimzi images uses the JVM, there is no benefit in native images now.