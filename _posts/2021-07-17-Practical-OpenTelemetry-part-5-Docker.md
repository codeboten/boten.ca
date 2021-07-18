---
layout: post
title:  "Practical OpenTelemetry part 5: Test drive Python, Go, Java with a single command"
author: alex
categories: [ opentelemetry, java, python, go, collector, observability, tracing ]
image: assets/images/antoine-petitteville-hHntcuiLbOg-unsplash.jpg
featured: true
hidden: false

---

In part 5 of my Practical OpenTelemetry series, I've decided to take a bit of time to make launching the applications from the previous parts easier. This article will go over:

* containerizing each service written so far
* creating a docker-compose file for all the previous services
* launching the services and seeing the results

The nice thing about this article, is that if all you want is to use a single `docker compose` command to launch an application and see OpenTelemetry in Go, Python and Java in action, you now can.

### Requirements

* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [Docker Compose](https://docs.docker.com/compose/install/)

If you're only interested in seeing the results, run the following to clone the repository and launch the example using compose.

```
git clone https://github.com/codeboten/practical-otel.git && cd practical-otel
docker compose up
```

If the previous docker compose command failed, you may be using an older version of compose, in which case, the command may be `docker-compose`. Open http://localhost:9411 or http://localhost:16686 and you should be able to see traces right away! Let's figure out how all the in this example works. First, I dockerized the applications from previous examples.

## Containerizing services

Containerizing in context of this article refers to taking the application code and packaging it as a [Docker container](https://www.docker.com/resources/what-container). In order to containerize the applications, I created separate `Dockerfile` configuration for each part of this series. 

### Part 1

```dockerfile
FROM python:slim

RUN pip install \
  flask \
  requests

RUN pip install \
  opentelemetry-api \
  opentelemetry-sdk \
  opentelemetry-exporter-jaeger

RUN pip install \
  opentelemetry-instrumentation-flask \
  opentelemetry-instrumentation-requests

WORKDIR /app
ADD grocery_store.py ./
ADD shopper.py ./

CMD ["python", "grocery_store.py"]
```

### Part 2

```dockerfile
#
# Build stage
#
FROM golang:1.16
WORKDIR /go/src/github.com/codeboten/practical-otel/part2
COPY go.* .
COPY inventory.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o inventory .

#
# Package stage
#
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/codeboten/practical-otel/part2/inventory .
CMD ["./inventory"]
```

### Part 3

```dockerfile
#
# Build stage
#
FROM maven:3.6.0-jdk-11-slim AS build
COPY Checkout/src /home/app/src
COPY Checkout/pom.xml /home/app
RUN curl -L https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v1.3.1/opentelemetry-javaagent-all.jar -o /home/app/opentelemetry-javaagent-all.jar
RUN mvn -f /home/app/pom.xml clean package

#
# Package stage
#
FROM openjdk:11-jre-slim
COPY --from=build /home/app/target/Checkout-0.0.1-SNAPSHOT.jar /usr/local/lib/Checkout.jar
COPY --from=build /home/app/opentelemetry-javaagent-all.jar /usr/local/lib/
EXPOSE 8080
ENTRYPOINT ["java", \
     "-javaagent:/usr/local/lib/opentelemetry-javaagent-all.jar", \
     "-Dserver.host=0.0.0.0", \
     "-Dserver.port=8083", \
     "-Dspring.redis.host=redis", \
     "-Dotel.resource.attributes=service.name=checkout", \
     "-Dotel.metrics.exporter=none", \
     "-jar", "/usr/local/lib/Checkout.jar" \
]
```

Once we have Dockerfiles, we can build each application individually to create a container. We won't in this example, as we'll use compose to build all the applications simultaneously shortly.

### Part 4

The configuration for the collector from [Part 4]((https://words.boten.ca/practical-otel-part-4-collector/)) needs to be updated slightly to ensure that the collector can send data to the backends. If you remember, in Part 4, we exported the traffic to localhost, but because we're now running the network using a Docker environment, the destination for the backends will need updating. Below you'll find the new collector configuration needed, it's the same as it was in Part 4, with new hosts configured for `jaeger` and `zipkin` exporters:

```yaml
receivers:
  jaeger:
    protocols:
      thrift_compact:
      grpc:
exporters:
  jaeger:
    endpoint: jaeger:14250
    insecure: true
  zipkin:
    endpoint: http://zipkin:9411/api/v2/spans
    format: proto
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [jaeger]
      exporters: [zipkin, jaeger, logging]
```

### Compose

To tie it all together, the following Docker compose configuration file configures the `shopper`, `store`, `inventory` and `checkout` applications. It uses [OpenTelemetry environment variables](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/sdk-environment-variables.md#jaeger-exporter) defined in the specification to set the host and port to connect and send data to the `collector`. Additionally, this compose file also configures the collector, zipkin, jaeger and redis instances needed for all the examples to work.

```yaml
version: "3.9"
services:
  shopper:
    build: part1/
    command: "/app/shopper.py"
    depends_on:
      - collector
      - store
    environment:
      OTEL_EXPORTER_JAEGER_AGENT_HOST: collector
      STORE_URL: http://store:5000
  store:
    build: part1/
    depends_on:
      - collector
      - inventory
    environment:
      OTEL_EXPORTER_JAEGER_AGENT_HOST: collector
      INVENTORY_SERVICE_URL: http://inventory:8080/inventory
      CHECKOUT_SERVICE_URL: http://checkout:8083/orders
  inventory:
    build: part2/
    depends_on:
      - collector
      - checkout
    environment:
      OTEL_EXPORTER_JAEGER_AGENT_HOST: collector
      OTEL_EXPORTER_JAEGER_AGENT_PORT: 6831
  checkout:
    build: part3/
    depends_on:
      - collector
      - redis
    environment:
      OTEL_EXPORTER_JAEGER_ENDPOINT: http://collector:14250
      OTEL_TRACES_EXPORTER: jaeger
  collector:
    image: otel/opentelemetry-collector:0.27.0
    volumes:
      - ${PWD}/part4/collector-compose-example.yaml:/etc/collector.yaml
    command:
      - "--config=/etc/collector.yaml"
    depends_on:
      - jaeger
      - zipkin
  # tracing backends
  jaeger:
    image: jaegertracing/all-in-one
    ports:
      - "0.0.0.0:16686:16686"
  zipkin:
    image: openzipkin/zipkin
    ports:
      - "0.0.0.0:9411:9411"
  # data store
  redis:
    image: redis

```

### Gotchas

One thing that's worth highlighting is that the jaeger port varies depending on which application is instrumented. This is because currently, different languages in OpenTelemetry have implemented different protocols for Jaeger, and depending on which protocol is supported, a different port is used.

Time to get the services running! Run docker compose from the root of the repository

```
docker-compose up
```

This command takes a few minutes the first time it is run as Docker is downloading images and building the application containers. Once all the services are up and running, visit http://localhost:9411 or http://localhost:16686 and search for traces! If you're interested in the previous chapters of this series, check them out here:

- [Practical OpenTelemetry part 1: Python](https://words.boten.ca/Practical-OpenTelemetry-part-1-Python/)
- [Practical OpenTelemetry part 2: Go](https://words.boten.ca/practical-otel-part-2-Go/)
- [Practical OpenTelemetry part 3: Java](https://words.boten.ca/Practical-OpenTelemetry-part-3-Java/)
- [Practical OpenTelemetry part 4: Collector](https://words.boten.ca/practical-otel-part-4-collector/)

As always, feel free to let me know if this has been helpful in your journey to using OpenTelemetry!

------

Photo by [Antoine Petitteville](https://unsplash.com/@ant0ine?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/container?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)