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

We will create new repository under Strimzi organization - `strimzi/lag-reporter` - that will contain the implementation code.

### Implementation

Because Strimzi mostly consists of Java projects, the proposal is to implement the tool in Java.
The implementation will use the minimal required dependencies and will cover all functionality currently used by Strimzi.
Features not used by Strimzi are explicitly out of scope for the initial implementation, including Helm charts, Kerberos authentication, and SASL mechanisms other than OAUTHBEARER.

All the metrics that the tool will export will follow the same naming as the existing Kafka Exporter, but this can change in the future based on community feedback and needs.

To keep the minimal dependency tree we will use the following:
- `kafka-clients` for `AdminClient`
- `prometheus-metrics-core` and `prometheus-metrics-exposition-formats` for Prometheus endpoint
- JDK built-in HTTP server for the `/metrics` and health check endpoints

#### HTTP Server

The tool exposes two HTTP listeners on separate ports, following the same pattern used by the Kafka Bridge:

- **Management port** (default: `8080`) — serves `/healthz/ready` over plain HTTP only.
This port is never TLS-enabled and is used exclusively by the operator's liveness and readiness probes.
Keeping health check endpoints on a dedicated plain-HTTP port means probe behaviour is stable regardless of the TLS configuration of the metrics endpoint.
- **Metrics port** (default: `9404`) — serves `/metrics`.
In the initial release this port also uses plain HTTP.

There is community demand for having TLS enabled on metrics endpoint ([strimzi-kafka-operator#12556](https://github.com/strimzi/strimzi-kafka-operator/issues/12556)).
The implementation and integration into Strimzi Kafka Operator will require new proposal as it will need API changes for the tool.

### CI/CD

The tool will follow Strimzi standards and will adopt the same CI/CD workflow we use for other projects.
As an output of the build and release process, we will produce a zip file that we use across Strimzi org and that can be used in Strimzi images or in standalone distributions connected to Kafka.

### Versioning

The tool will follow Strimzi versioning `<major>.<minor>.<micro>` as other projects do.
The first version will be `0.1.0`, even though the tool already covers all Strimzi-required functionality.

### Strimzi Kafka Operator changes

#### Feature Gate for Strimzi Kafka Operator

The switch from the Go binary to the Java implementation will be gated behind a new feature gate — `StrimziLagReporter` — to allow a safe, opt-in transition for users.

The gate will progress through the standard Strimzi feature gate lifecycle:

1. Introduced as **alpha** (disabled by default) - users can opt in to the new Java implementation while the Go binary remains the default (introduced in 1.4.0).
2. Promoted to **beta** (enabled by default) after two releases (in 1.6.0) once sufficient feedback and production validation has been gathered - users can still opt out by explicitly disabling the gate.
3. Promoted to **GA** and the feature gate removed after four releases (in 1.8.0) - the Go binary is dropped and the Java implementation becomes the only option.

When the feature gate is disabled, the operator continues to use the existing Go binary and `kafka_exporter_run.sh` launch script unchanged.
When the feature gate is enabled, the operator will use Java binary baked in the Kafka image and `lag_reporter_run.sh` to launch the application.

#### CRD and API Changes

Moving from a Go binary to a Java implementation requires normalising some fields in `KafkaExporterSpec` that are Go-specific.

The following changes will be made to the `kafkaExporter` section of the Kafka CR:

- `logging` — currently accepts Go-style log levels (`info`, `debug`, `trace`).
  The existing field is kept and its value is mapped internally to the equivalent Java log level, so existing CRs continue to work without changes.
- `enableSaramaLogging` — this field controls logging of the Sarama Go client library, which has no equivalent in the Java implementation.
  The field will be deprecated and ignored when the Java implementation is active.
  When the feature gate is enabled and this field is set, the operator will emit a warning in its log.
- `jvmOptions` — a new field following the standard Strimzi `JvmOptions` type will be added to allow users to configure JVM heap, GC options, and other JVM flags, consistent with how other Java-based Strimzi components expose this.
  This field is only meaningful when the Java implementation is active.
  When the feature gate is disabled and this field is set, the operator will emit a warning in its log that `jvmOptions` is ignored by the Go binary.

The deprecated Go-specific fields will be removed in a future API version once the feature gate reaches GA and the Go binary is fully retired.

### Documentation

The `strimzi/lag-reporter` repository will include a README covering all configuration options, the full list of exported metrics, and instructions for running the tool standalone.
The Strimzi documentation will be updated to reflect the switch from the upstream Go binary to the new Java implementation.

### Testing

The new repository will include unit tests covering the metrics collection and registry logic, and integration tests running against a real Kafka instance using `strimzi-test-container`.
Existing system tests in `strimzi-kafka-operator` will ensure that implementation of Lag Reporter continues to work end-to-end after the swap.
No additional e2e scenarios will be needed.

### Security

The new implementation preserves the TLS/mTLS configuration used by Strimzi today. 
The tool continues to use the cluster CA certificate and client certificate/key mounted by the operator via Secrets.
No credentials will be logged or persisted by the tool itself.
Moving to a Java implementation removes the current need to trust and verify pre-built third-party Go binaries and their checksums, replacing them with well-known JVM dependencies that already go through Strimzi's existing CVE scanning and patching process.

SASL OAUTHBEARER support will be added as a follow-up.
Other SASL mechanisms are currently out of scope.

## Affected Projects

This proposal affects only `strimzi-kafka-operator` repo.
Other sub-projects are not affected.

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
The main advantage of Quarkus for a tool like this would be native image builds, but native images are not viable for Strimzi because they only support x86-64 and aarch64, while Strimzi ships images for s390x and ppc64le as well.