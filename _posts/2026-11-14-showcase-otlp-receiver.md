---
layout: post
title:  "Collector Showcase: OTLP Receiver"
author: alex
categories: [ opentelemetry, collector ]
image: assets/images/showcase-1.png
featured: true
hidden: false

---

## What's it do?

The OTLP (OpenTelemetry Protocol) receiver can be configured to receive telemetry from clients or other data sources. It supports both HTTP and gRPC.

## Do I need it?

If you're using anything that's instrumented with OpenTelemetry, you'll definitely want it. There is an increasing number of applications (Docker, Claude SDK, ...) that support OTLP out of the box.

## What's a minimum config look like?

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:
```

## Where do I learn more?

- [https://explorer.opentelemetry.io/collector/components/core/otlpreceiver](https://explorer.opentelemetry.io/collector/components/core/otlpreceiver)
- [https://opentelemetry.io/docs/collector/configuration/#receivers](https://opentelemetry.io/docs/collector/configuration/#receivers)
- [https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver)
