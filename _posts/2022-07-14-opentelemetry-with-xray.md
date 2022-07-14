---
layout: post
title: OpenTelemetry tracing with AWS X-Ray
---

NServiceBus version 8 will come with OpenTelemetry instrumentation to capture message processing traces and metrics and export those to any OpenTelemetry compatible tooling. Some tools like Jaeger or Honeycomb are easy to setup and configure. Others require a bit more effort and AWS X-Ray definitely is one of those. In this post I'm documenting how to configure a .NET application to send NServiceBus (and any other) OpenTelemetry data to X-Ray in a development environment. My development environment is using Windows and requires Docker, so your experience on a different OS might differ (but I expect it to be simpler).

Unlike other OpenTelemetry exporters, X-Ray integration happens via a dedicated collector from the AWS Distro for OpenTelemetry (ADOT) project. The .NET application will export OpenTelemetry data via the OpenTelemetry protocol to the ADOT collector which in turn will export the collected data to AWS X-Ray.

TODO: mermaid diagram or similar

Let's get started:

1. Azure CLI needs to be setup locally and configured with your account.
1. Configure IAM permissions for X-Ray
1. Install OpenTelemetry.Exporter.OpenTelemetryProtocol
1. Install install aws xray otel package
1. configure endpoints: .AddXRayTraceId() and .AddOtlpExporter()
1. Prepare the ADOT Collector configuration file
1. Start the locally running ADOT Collector using docker.
1. Run the application
1. verify log output on ADOT collector

If everything ran correctly, you should be able to see your captured traces in the AWS X-Ray dashboard:

TODO: Screenshot
