---
layout: post
title:  "Reflecting on an incredible year in OpenTelemetry"
author: alex
categories: [ opentelemetry, observability ]
image: assets/images/reflections-2023.png
featured: true
hidden: false

---

And just like this, we're fast approaching the end of 2023. This year has been an incredible year for OpenTelemetry. The project has reached a new milestone, and continues to see growth in both contributions and usage. I wanted to take a bit of time and highlight some things I'm really excited about that happened in the past year, starting with end-users.

## End users getting involved

Although initially the project came together via the collaboration of many vendors, the goal of the project was always to make lives better for tne end user. I loved seeing the increase in end user engagement throughout 2023. For the first time ever, the [governance committee election](https://opentelemetry.io/blog/2023/gc-election-results/) saw a representative from an end user organization be elected in Daniel Gomez Blanco. The [end user working group](https://github.com/open-telemetry/community/tree/main/working-groups/end-user) continued its fantastic work throughout the year bringing stories of migration and use-cases for OpenTelemetry to the community, as well as help folks answer their many questions. OpenTelemetry maintainers also hosted the project's first ever [Contribfest](https://opentelemetry.io/blog/2023/contribfest-na/) at Kubecon in Chicago, which saw many new contributors roll up their sleeves and contribute code.

## OpenTelemetry officially announces General Availability

In case you didn't see the OpenTelemetry project update at Kubecon in Chicago, I recommend watching the session's [recording](https://www.youtube.com/watch?v=OEGgmTNfYsU). A little over four and a half years after the initial announcement of the project, OpenTelemetry finally declared General Availability (GA) for the third signal on its original roadmap: logging. This was a huge milestone for the project and a testament to the efforts of its community members.

![OTel GA announcement](/assets/images/otel-ga.png)

With the logging signal officially GA, implementations of the specification were finally able to make progress in shipping the logging bridge API. As of December 2023, PHP, Java, and C++ implementations have already shipped support for this. You can check on the progress of your favourite language via the specification [compatibility matrix](https://github.com/open-telemetry/opentelemetry-specification/blob/main/spec-compliance-matrix.md#logs).

## Metrics land in OpenTelemetry Go

The last area that I really to highlight was the stable release of the Metrics [API](https://github.com/open-telemetry/opentelemetry-go/blob/main/CHANGELOG.md#11600390-2023-05-18) and [SDK](https://github.com/open-telemetry/opentelemetry-go/blob/main/CHANGELOG.md#11900420007-2023-09-28) in the Go implementation. This was huge for many reasons, but personally this was huge because it allows the OpenTelemetry collector to move forward with its adoption of OpenTelemetry. It is something that is especially dear to my heart as I look forward to 2024 as the year where I'll be able to emit OTLP directly from the Collector for its own telemetry! Thanks to the OTel Go community for making this happen!

## Next: 2024

There were of course many more things that evolved in 2023, the OTel Arrow work producing remarkable results, the configuration working group delivering a JSON schema, and the semantic convention stabilization efforts are just a few more of those. I look forward to the new year to see where the project will go from here. Thank you to all the contributors for making this a phenomenal project. Check out the [Humans of OTel](https://opentelemetry.io/blog/2023/humans-of-otel/) for a behind the scene look at some of the Otellinos (please send me more suggestions for other names for otel contributors) making this a great community.