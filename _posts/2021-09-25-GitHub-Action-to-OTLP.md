---
layout: post
title:  "OpenTelemetry + GitHub Actions: Observability for your builds"
author: alex
categories: [ opentelemetry, cicd, otlp, collector, observability, tracing ]
image: assets/images/diego-ph-fIq0tET6llw-unsplash.jpg
featured: true
hidden: false
---

Build pipelines. Many people use them. They always start out with so much promise; quick builds help achieve fast feedback loops for developers. Deploying to production can happen automatically with all the checks to ensure validated code goes out. A solid Continuous Integration and Continuous Delivery (CICD) practice helps organizations reduce the friction and frustration that can come from deploying increasingly complex software as the list of dependencies grows.

One of the things I learned time and time again working in the software industry, is that few organizations spend the time to make their build pipelines as resilient as the code they ship to customers. And fair enough, CICD is rarely the core competency of an organization so when time is limited, it doesn't make sense to focus on things that don't provide value to your users. That being said, neglected builds leads to increasingly slower build times as the codebase becomes larger. Sluggish builds mean developers spend more time waiting around for their code to be merged or deployed. Additionally, neglected builds also tend to eventually suffer from intermittent failures. These failures can be caused by many reasons:

* brittle unit tests
* integration tests relying on external services
* load tests struggling on shared nodes
* variable dependencies

All these factors lead to slower feedback loops and less frequent deployments, causing stress and unhappiness in the maintainers of the software. What if we could use the same observability tools that we use in production to monitor tests and builds to help identify patterns in the builds?

## Using OpenTelemetry with your builds

OpenTelemetry provides the tooling necessary to instrument and collect telemetry about your softrware. A few months ago, a [plugin for Jenkins](https://plugins.jenkins.io/opentelemetry/) was released to collect monitoring data from Jenkins and make it possible to emit it to an OpenTelemetry backend. Most of the projects I work on these days are built using GitHub Actions so taking inspiration from that Jenkins plugin, I wrote a GitHub Action. 

![action-diagram](/assets/images/github-to-otlp-01.png)

This action:

1. pulls timing information about GitHub workflows 
2. produces a distributed trace representing the workflow
3. emits the tracing data to a backend that supports the OpenTelemetry protocol

This lets me configure the action to send data to the same observability backend I use for monitoring and operating the rest of the software I work with. This means I can use the same tool to see patterns in the builds or create alerts if certain thresholds are crossed. The flexibility of using OTLP frees me from having to look at multiple tools when something is failing. 

## Requirements

* A GitHub action you want to see tracing information for
* A backend that supports OTLP over gRPC or an OpenTelemetry Collector configured to receive external traffic

Add the following code to your action yaml:

```
      - name: Send data to OTLP backend
        uses: codeboten/github-action-to-otlp@v1
        with:
          endpoint: "endpoint:443"
          headers: "key=value"
```

And trigger your job, that's all the code required! You can see an [example job](https://github.com/codeboten/github-action-to-otlp/blob/main/.github/workflows/test.yaml) sending data to my Lightstep account which requires a header for an access token to be specified. The data emitted looks like something this:

![trace-view](/assets/images/github-to-otlp-02.png)

Today, the action doesn't support private repositories as there's no authentication code in the action. If this is something you're interested in, I would love to hear from you via a GitHub issue or a pull request to the [repository](https://github.com/codeboten/github-action-to-otlp). I'd also like to improve the action by augmenting the telemetry to include metrics and logs about the builds as well, but one thing at a time.

------

Photo by [Diego PH](https://unsplash.com/@jdiegoph?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/automation?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)