---
layout: post
title: "Building the Collector, can't someone else do it?"
author: alex
categories: [opentelemetry, collector, observability, builder]
image: assets/images/builder.jpg
featured: true
hidden: false
---

The [OpenTelemetry Collector Builder](https://opentelemetry.io/docs/collector/custom-collector/) is a tool that allows users to compile their own custom instance of the [Collector](https://words.boten.ca/cloud-native-observability-excerpt/). This is useful in a few different scenarios. Maybe the existing distributions contain more components than you need. Choosing specific components reduces the complexity and size of the resulting Collector. In my experience, the fewer components you're including in binaries shipped around production, the happier the security teams tend to be.

Another reason you may need to use the Builder is that you're producing custom Collector components. With the Builder, adding a component requires as little as a single line in the manifest file, and you can add a component hosted anywhere. Whatever your reason is, getting familiar with the Builder is a good practice if you're in the business of deploying the Collector in your environment.

## Using the Builder

One thing I've been wanting to make easier still is the process of using the Builder. Today, if you're looking at building a local instance of the collector, installing the Builder and using it can be done in a few steps:

1. Install the Builder
2. Create a manifest.yaml file
3. Run the builder with a --config manifest.yaml flag
4. Profit!

Of course, building binaries on a local machine, then shipping it off to a production environment isn't often a recommended practice. The better way to go about it is to put your configuration in a repository, and use a build pipeline to do all the work. This will ensure repeatability (usually...) of the build, along with keeping a record of the changes to your build. Unfortunately, having built a few different custom Collectors has caused me to copy and paste a lot of unnecessary code around. Could a custom action do it instead? It sure can!

## Collector Builder GitHub Action

In order to reduce the amount of required boiler plate code, I wrote a custom [GitHub action](https://github.com/codeboten/collector-builder-action/tree/main) runs a Docker container with the Builder already installed. A user of this action specifies the path of the manifest file to use as configuration via the `manifest-file` input. The action will then produce a custom Collector binary! The following shows an example GitHub action configuration that clones the current repository, uses the collector-builder-action, and uploads the resulting Collector binary:

```yaml
on: [push]

jobs:
  custom-collector-action:
    runs-on: ubuntu-latest
    name: A job to build a custom OpenTelemetry Collector
    steps:
      - uses: actions/checkout@v3

      - name: Custom Collector Step
        uses: codeboten/collector-builder-action@v1

      - uses: actions/upload-artifact@v3
        with:
          name: otelcolaction
          path: _build/otelcolaction

```

In my [test repository](https://github.com/codeboten/custom-collector/tree/main), the manifest looks like any other Collector manifest file:

```yaml
dist:
  module: github.com/open-telemetry/opentelemetry-demo/src/otelcollector
  name: otelcol-demo
  description: OpenTelemetry Collector for OpenTelemetry Demo
  version: 0.81.0
  output_path: ./_build
  otelcol_version: 0.81.0

receivers:
  - gomod: go.opentelemetry.io/collector/receiver/otlpreceiver v0.81.0
exporters:
  - gomod: go.opentelemetry.io/collector/exporter/loggingexporter v0.81.0
  - gomod: go.opentelemetry.io/collector/exporter/otlpexporter v0.81.0
  - gomod: go.opentelemetry.io/collector/exporter/otlphttpexporter v0.81.0
processors:
  - gomod: go.opentelemetry.io/collector/processor/batchprocessor v0.81.0
connectors:
  - gomod: github.com/open-telemetry/opentelemetry-collector-contrib/connector/spanmetricsconnector v0.81.0

```

And that's all you need in your repository. Once the GitHub action is run, the Collector Builder Action should produce an artifact as it did in my case.

![github-action-screenshot](/assets/images/builder-1.png)

So what next? Well, the initial release of this action is full of sharp edges. There's no support for multiple architectures. It would be great if this action could produce a Docker image. And I'm pretty sure there are many more things I'd like to improve. If this is useful to you, please let me know via whatever social platform. You can also open an issue or a pull request! And who knows, maybe this will eventually be donated to the OpenTelemetry project. In the meantime, enjoy building your Collector, now with even fewer steps!

----

Cover Photo by [Marcel Strauß](https://unsplash.com/@martzzl?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/rxGN4Z6wN5s?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)