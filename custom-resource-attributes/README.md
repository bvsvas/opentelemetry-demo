# Configuring Custom Resource Attributes in OpenTelemetry

> **Note:** The configurations, attributes, divisions, value streams, and organizational data provided in this directory are purely for illustration purposes as an example.

This directory contains examples demonstrating how to inject, derive, and manipulate custom resource attributes using OpenTelemetry instrumented applications and the OpenTelemetry Collector.

## Contents

- `java-app-team-a.yaml` & `java-app-team-b.yaml`: Java deployments configured with standard OpenTelemetry environment variables (`OTEL_RESOURCE_ATTRIBUTES`).
- `nodejs-app-team-a.yaml` & `nodejs-app-team-b.yaml`: Node.js deployments similarly configured.
- `otel-collector.yaml`: OpenTelemetry Collector configuration demonstrating the `transform` processor with OpenTelemetry Telemetry Language (OTTL).

## Prerequisites

All manifests in this directory target the `otel-demo` namespace, which must exist before applying them:

```bash
kubectl create namespace otel-demo
```

## Configuration Guide

### 1. Injection at the Edge (Application Level)

Injecting static metadata at startup—using the `OTEL_RESOURCE_ATTRIBUTES` environment variable—is the simplest and most effective way to add custom resource attributes. This "shift-left" approach ensures your metadata travels natively with all traces, metrics, and logs directly from the source, eliminating the need for complex code changes or heavy processing at the Collector.

> **Important:** According to the OpenTelemetry Resource SDK specification, a [Resource](https://opentelemetry.io/docs/concepts/resources/) is an **immutable representation** of the entity producing the telemetry. The key-value pairs you define in `OTEL_RESOURCE_ATTRIBUTES` must be static/constant at the application startup level. They cannot be dynamically changed per-request or per-transaction. If you have data that changes per request (like a transaction ID or user email), those must be captured as **Span Attributes** or **Datapoint Attributes** within your code, not as Resource attributes.

**Example for Kubernetes (`java-app-team-a.yaml`):**
```yaml
env:
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: "broadcom_organization=Broadcom,broadcom_division=IMS,broadcom_github_repo=sample-repo-alpha,broadcom_valuestream=ITOM,broadcom_user_id=demo-user-01,service.name=java-app-team-a"
```

**Example for a Standalone Java Application:**
If you are running a traditional standalone Java application with the OpenTelemetry Java Agent, you can inject these attributes directly via the command line using `-Dotel.resource.attributes` or by exporting the environment variable before starting the application:

```bash
# Method 1: Exporting as an environment variable
export OTEL_RESOURCE_ATTRIBUTES="broadcom_organization=Broadcom,broadcom_division=IMS,broadcom_github_repo=sample-repo-alpha,broadcom_valuestream=ITOM,broadcom_user_id=demo-user-01,service.name=java-app-team-a"
java -javaagent:opentelemetry-javaagent.jar -jar myapp.jar

# Method 2: Passing as a JVM argument
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.resource.attributes="broadcom_organization=Broadcom,broadcom_division=IMS,broadcom_github_repo=sample-repo-alpha,broadcom_valuestream=ITOM,broadcom_user_id=demo-user-01,service.name=java-app-team-a" \
     -jar myapp.jar
```

### 2. Derivation in the Collector

> **Note:** Derivation in the Collector is entirely optional. This configuration is provided as an illustrative example of how you can leverage the Collector's processing capabilities to dynamically compute and attach additional enriched attributes based on the foundational metadata already supplied by the application.

The `otel-collector.yaml` uses the **Transform Processor** (with OTTL) to look at the incoming resource attributes and derive new organizational attributes downstream without requiring application changes:

- If `broadcom_division` is `IMS`, it assigns a new resource attribute `broadcom_cost_center` = `CC-IMS-100`.
- If `broadcom_valuestream` is `ITOM`, it assigns a new resource attribute `broadcom_support_tier` = `Tier-1`.

**Example OTTL Statement (Resource Context):**
```yaml
trace_statements:
  - context: resource
    statements:
      - set(attributes["broadcom_cost_center"], "CC-IMS-100") where attributes["broadcom_division"] == "IMS"
      - set(attributes["broadcom_support_tier"], "Tier-1") where attributes["broadcom_valuestream"] == "ITOM"
```

### 3. Contextual Mapping to Datapoints/Spans

Also within the Transform Processor, we evaluate the newly derived resource attributes to mutate the actual datapoints and spans. This is an advanced technique useful for dynamic routing, sampling, or prioritizing telemetry.

- For datapoints and spans where the derived `resource.attributes["broadcom_support_tier"] == "Tier-1"`, it injects a datapoint/span attribute `routing_priority = critical`.

**Example OTTL Statement (Span/Datapoint Context):**
```yaml
  - context: span
    statements:
      - set(attributes["routing_priority"], "critical") where resource.attributes["broadcom_support_tier"] == "Tier-1"
```

## OpenTelemetry Best Practices for Resource Attributes

1. **Shift Left (Inject at the Source)**: Inject static resource attributes at the application deployment level. This ensures contextual completeness across metrics, logs, and traces.
2. **Use the Transform Processor for Conditional Logic**: The `transform` processor with OTTL is the modern standard for *conditional or dynamic* telemetry mutation (e.g., `where` clauses, deriving one attribute from another). The `metricstransform` processor is deprecated in favor of `transform`. However, the simpler `attributes` and `resource` processors remain the recommended, non-deprecated choice for static, unconditional attribute changes — as used for the `dxotel.cluster` insert in `otel-collector.yaml`.
3. **Resource vs. Datapoint Attributes**: 
   - *Resource attributes* describe the entity producing the telemetry (e.g., container, pod, team, organization).
   - *Datapoint/Span attributes* describe the specific event or measurement (e.g., HTTP status code, routing priority). 
4. **Limit Cardinality on Metrics**: While trace span attributes can have high cardinality (e.g., user IDs, request IDs), attaching high-cardinality data as attributes to *Metrics* can cause Time Series Database (TSDB) storage costs to explode. Use resource attributes freely for metadata, but be cautious when mapping them to metric datapoint attributes.
5. **Namespace Custom Attributes**: Always use a strict custom namespace (e.g., `broadcom_organization` instead of just `organization`) to prevent future collisions with standard OpenTelemetry Semantic Conventions.

## References

For more detailed information on OpenTelemetry Resource Attributes and configuration, please refer to the official documentation and specifications:

- [OpenTelemetry Concepts: Resources](https://opentelemetry.io/docs/concepts/resources/)
- [OpenTelemetry SDK Environment Variables](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)
- [OpenTelemetry Specification: Resource SDK](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/resource/sdk.md)
