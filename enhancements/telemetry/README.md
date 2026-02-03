# Telemetry System

The telemetry system is responsible for defining, collecting, processing, and
exporting telemetry data (metrics, logs, and traces) from all services on the
Datum Cloud platform. This document provides a high-level overview of the system
and the functionality that's available.

## Overview

![System context diagram for the telemetry system](./system-context.png)

The telemetry system is used by all services on Datum Cloud to define the
telemetry data they make available to consumers. The telemetry system is
responsible for collecting the telemetry data and exporting it to third-party
telemetry data platforms that support ingesting data over the OpenTelemetry
protocol.

> [!NOTE]
>
> In the future Datum Cloud plans to offer a hosted telemetry dashboard (e.g.
> Grafana) that allows users to view and analyze telemetry data exposed by Datum
> Cloud services. Datum Cloud will provide pre-built Grafana dashboards for all
> services as well as allow users to built their own dashboards.

## Capabilities

### Exporting Telemetry

Telemetry exporters can be configured to select telemetry data exposed by
services on Datum Cloud and export to the platform of your choosing. We support
exporting to any telemetry platform that supports ingesting data over the
OpenTelemetry protocol.

Review the [telemetry exporter enhancement](./exporters/) for more details on
the design of our telemetry exporter functionality.

### Policy-Driven Metrics Platform

The policy-driven metrics platform introduces a stable, declarative contract for
metric definition and production that separates *what* metrics are from *how*
they are produced. Service providers can define portable **MetricDefinitions**
that specify metric identity, description, instrument type, unit, and label
schema—independent of the underlying telemetry stack. **MetricPolicies** then
declare how to generate metric values from either control-plane resource fields
or data-plane telemetry.

This approach enables:

**For Service Providers:**

- Portable metric definitions that work across different telemetry backends
  (OTLP, Prometheus-compatible, etc.)
- Declarative policies to transform resource state and data-plane telemetry into
  standardized metrics
- Consistent metric naming, units, and labels across all platform resources
- Future-proof architecture that allows telemetry implementation changes without
  API modifications

**For Service Consumers:**

- Reliable metric discovery with well-defined schemas and stable contracts
- Consistent metric identity and labeling across all platform services
- Multi-tenant, authorization-aware metric exposure aligned with Milo IAM
- Foundation for standardized alerting conditions and metering for billing/cost
  attribution

The platform translates these high-level definitions and policies into
backend-specific configurations (OTel Collector pipelines, Prometheus rules,
etc.), eliminating the need for hand-crafted, backend-specific telemetry
configurations.

Review the [policy-driven metrics platform enhancement](./metrics/) for detailed
technical specifications.
