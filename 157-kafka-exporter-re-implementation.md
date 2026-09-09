# Kafka Exporter reimplementation

This proposal suggests to re-implement Kafka Exporter tool under Strimzi organization in Java.

## Current Situation

Strimzi currently downloads binaries from released version of [kafka_exporter](https://github.com/danielqsj/kafka_exporter) tool on GitHub.
The binaries are baked into Strimzi images and shipped as an optional tool for extending Kafka consumer metrics.

## Motivation

The project is widely used by Kafka community, however, it is not well maintained.
The latest release, v1.10.0, was published on 2026-09-08 after a period of minimal activity, but the project still has 216 open issues and 56 open pull requests with limited attention from the code owner.
This opens space for potential security issues as CVEs may not be fixed promptly.

This situation is also not helpful for new features that we would like to add into the exporter.

## Proposal

To mitigate security problems and allow us to better maintain the tool and provide new features, we should re-implement Kafka Exporter tool.

We will create new repository under Strimzi organization - `strimzi/insights-reporter` - that will contain the implementation code.

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
The implementation and integration of TLS configuration into Strimzi Kafka Operator will require a new proposal as it will need API changes for the tool.

### CI/CD

The tool will follow Strimzi standards and will adopt the same CI/CD workflow we use for other projects.
As an output of the build and release process, we will produce a zip file that we use across Strimzi org and that can be used in Strimzi images or in standalone distributions connected to Kafka.

### Versioning

The tool will follow Strimzi versioning `<major>.<minor>.<micro>` as other projects do.
The first version will be `0.1.0`, even though the tool already covers all Strimzi-required functionality.

### Strimzi Kafka Operator changes

#### CRD and API Changes

Insights Reporter will be introduced as a new, parallel section in the Kafka CR — `spec.insightsReporter` — alongside the existing `spec.kafkaExporter` section.
This allows users to opt in to the new Java implementation without any changes to their existing `kafkaExporter` configuration.

The `spec.kafkaExporter` section will be set as **deprecated** from the moment `spec.insightsReporter` is introduced.
The upstream `kafka_exporter` binary will be updated to v1.10.0 and continue to be supported and bundled in Strimzi images until new API version or until the tool will work without significant required changes on our side.
The `spec.kafkaExporter` section and the Go binary will be removed in a future API version once the deprecation period ends.

The new `spec.insightsReporter` section follows the same conventions as other Strimzi components such as CruiseControl.
The component is enabled by including the section in the CR and disabled by omitting it — there is no separate `enabled` field, consistent with `spec.kafkaExporter` and `spec.cruiseControl`.
The section will include the following fields from the start:

- `image` — container image override.
- `groupRegex` — consumer group include regex (default: `.*`).
- `groupExcludeRegex` — consumer group exclude regex.
- `topicRegex` — topic include regex (default: `.*`).
- `topicExcludeRegex` — topic exclude regex.
- `showAllOffsets` — whether to report offsets for all partitions the group has ever committed to (default: `true`).
- `logging` — standard Strimzi `Logging` type (Log4j 2), replacing the plain string `logging` field from `kafkaExporter`.
- `jvmOptions` — standard Strimzi `JvmOptions` type for JVM heap, GC options, and flags.
- `resources` — CPU and memory resource requirements.
- `livenessProbe` — liveness probe configuration.
- `readinessProbe` — readiness probe configuration.
- `template` — pod and container template overrides.

Fields that exist in `kafkaExporter` but are Go-specific, such as `enableSaramaLogging`, will not be carried over to `spec.insightsReporter`.

When both `spec.kafkaExporter` and `spec.insightsReporter` are set, the operator will emit a warning and use `spec.insightsReporter`, ignoring `spec.kafkaExporter`.

### Documentation

The `strimzi/insights-reporter` repository will include a README covering all configuration options, the full list of exported metrics, and instructions for running the tool standalone.
The Strimzi documentation will be updated to reflect the new component and the deprecation of `spec.kafkaExporter`.

### Testing

The new repository will include unit tests covering the metrics collection and registry logic, and integration tests running against a real Kafka instance using `strimzi-test-container`.
Existing system tests in `strimzi-kafka-operator` will be extended to cover Insights Reporter and will continue to cover the existing Kafka Exporter for the duration of the deprecation period.

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

Users who continue to use `spec.kafkaExporter` are not affected.
The Go binary is kept at the latest upstream release (v1.10.0) and will continue to function until the deprecation period ends.
Users who migrate to `spec.insightsReporter` will get the same metric names as today, so existing dashboards and alerting rules continue to work without modification.

## Rejected Alternatives

### Feature gate transition

An earlier version of this proposal used a feature gate to switch between the Go binary and the Java implementation within the existing `spec.kafkaExporter` API.
This was rejected because it couples two unrelated implementations under the same API, complicates the operator logic for the duration of the gate lifecycle, and prevents a clean API that reflects the capabilities of the Java implementation.
Introducing a new `spec.insightsReporter` section alongside a deprecated `spec.kafkaExporter` is a cleaner separation.

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
