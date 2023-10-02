---
layout: post
title:  "Bye bye logging exporter, hello debug exporter!"
author: alex
categories: [ opentelemetry, observability, collector ]
image: assets/images/logs.png
featured: true
hidden: false

---

Last week, a new version of the OpenTelemetry Collector was [released](https://github.com/open-telemetry/opentelemetry-collector-releases/releases/tag/v0.86.0) which includes a new and game-changing exporter, the `debug` exporter. Now, to be honest, it's neither new nor game-changing. But it sounds more exciting if I put it in those terms so I'll just go with it. The `debug` exporter is essentially a rename of what was known as the `logging` exporter. This rename has been on the community's TODO list for some time and I wanted to share some more context around why it happened. More importantly, if you're using the logging exporter today, I'll list out what you need to do, and answer some very important questions.

## What's with the rename?

The decision to rename the logging exporter was not made lightly. This was debated many times in the community's weekly calls and discussed over months in a GitHub [issue](https://github.com/open-telemetry/opentelemetry-collector/issues/7769). The logging exporter was originally created when there was no support for logs in OpenTelemetry. But as logging became a signal, and the OpenTelemetry Collector added support for it, the confusion created by the `logging` exporter grew. Do we use the logging exporter only for logs? How do I export the logging exporter's logs over OTLP? There's a logging pipeline and a logging exporter, how does that work? Is there an exporter I can use for debugging the collector?

Additionally, there was never a commitment from the community around the consistency of the output from the logging exporter. As the component was still marked as being under development, there were no guarantees that end users could rely on the output beyond the case of debugging issues with the collector. So it was decided to rename the exporter to represent its original intent, which was to provide debug output for the collector. Feel free to review the following GitHub issues referencing the rename if you'd like additional context

- https://github.com/open-telemetry/opentelemetry-collector/issues/3524
- https://github.com/open-telemetry/opentelemetry-collector/issues/7769

## OMG THE LOGGING EXPORTER IS DEPRECATED NOW WHAT??!?

So the first thing to keep in mind, is that although it is now deprecated, the logging exporter will continue to be shipped with the Collector for some time. The initial thoughts from the community was to keep it around for at least one year. This may be extended as the community receives feedback from end users. The second thing, is that switching from the logging exporter to the debug exporter requires a small amount of effort.

### Updating the configuration

The following shows configuration for the logging exporter:

```yaml
exporters:
  logging:
    verbosity: detailed
```

After the rename, the configuration now looks like this:

```yaml
exporters:
  debug:
    verbosity: detailed
```

### Collector builder manifest

Some additional things to consider when making the change, is that if you're using the [OpenTelemetry Collector Builder](https://opentelemetry.io/docs/collector/custom-collector/), you'll need to update your manifest.yml file to include the new exporter. You may want to change the following line:

```yaml
exporters:
  - gomod: go.opentelemetry.io/collector/exporter/loggingexporter v0.86.0
```

To the following:

```yaml
exporters:
  - gomod: go.opentelemetry.io/collector/exporter/debugexporter v0.86.0
```

Note that if you'd like to include both to give yourself a change to ship a new Collector before changing your configuration over, that's totally ok as well.

### What about `loglevel`?

If you've used the `logging` exporter for some time, you may still be using the [deprecated](https://github.com/open-telemetry/opentelemetry-collector/pull/6334) `loglevel` configuration setting. The debug exporter intentionally does not support this option, to reduce the amount of tech debt in the new exporter. If you were using the following configuration:

```yaml
exporters:
  logging:
    loglevel: info
```

This would have to be migrated to use the `verbosity` option when switching to the debug exporter:

```yaml
exporters:
  debug:
    verbosity: normal
```

### Will this change again in the future?

I certainly hope not. I don't think anyone wants to go through another rename for this exporter again in the future.

Note that along with the releases in the collector distributions last week, the [helm charts](https://github.com/open-telemetry/opentelemetry-helm-charts) have also been updated to support the debug exporter. Hope this helped you get a better understanding of why the change happened and what changes (if any) need to be made to adapt to the change.

------

Photo by [Joel & Jasmin Førestbird](https://unsplash.com/@theforestbirds?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/Kfy_FwhfPlc?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)