---
layout: post
title:  "Practical OpenTelemetry part 2: Go"
author: alex
categories: [ opentelemetry, go, observability, tracing ]
image: assets/images/kiarash-mansouri-fzoSNcxqtp8-unsplash.jpg
featured: true
hidden: false

---

For the next step of our Practical OpenTelemetry application, we'll add a service that gives our store an idea of what inventory is available. We'll write this component using Go and instrument using the [OpenTelemetry Go API and SDK](https://github.com/open-telemetry/opentelemetry-go). Please note that part 2 will skip over setting up Jaeger locally and the virtual environment for Python, which was covered in [Part 1](https://words.boten.ca/Practical-OpenTelemetry-part-1-Python/).

![architecture-diagram](/assets/images/otel-part-2-0.png)

Let's revisit the `grocery_store.py` and add a new `/whats-in-store` endpoint to retrieve the inventory. To keep the code as minimal as possible, we'll add a method that makes a call to the inventory service through the same `requests` library we used in previously:

```python
#!/usr/bin/env python3
# grocery_store.py
import os
import requests
...
@app.route("/whats-in-store")
def whats_in_store():
    inventory_service = os.environ.get(
        "INVENTORY_SERVICE_URL", "http://localhost:8080/inventory"
    )
    return requests.get(inventory_service)

```

Now we can update `shopper.py` to make two calls to our store:

```python
#!/usr/bin/env python3
# shopper.py
...
with trace.get_tracer(__name__).start_as_current_span("going to the grocery store"):
    res = requests.get("http://localhost:5000")
    print(res.text)
    res = requests.get("http://localhost:5000/whats-in-store")
    print(res.text)
```

Running the `grocery_store.py` and `shopper.py` we should see a couple of calls in the trace looking at the [Jaeger UI](http://localhost:16686):

![error-in-span](/assets/images/otel-part-2-1.png)

Take a close look, the second span shows us an error as it should, since we have yet to implement the inventory service! Let's get to that! The inventory service will be built using [Gin](https://github.com/gin-gonic/gin). 

```go
// inventory.go
package main

import "github.com/gin-gonic/gin"

type Product struct {
	Name  string `json:"name"`
	Price string `json:"price"`
	ID    string `json:"id"`
}

type Inventory struct {
	Products []Product `json:"products"`
}

func getInventory() Inventory {
	return Inventory{
		Products: []Product{
			{Name: "potato", Price: 0.99, ID: "1"},
			{Name: "apple", Price: 0.50, ID: "2"},
			{Name: "mango", Price: 1.50, ID: "3"},
		},
	}
}

func main() {
	r := gin.Default()

	r.GET("/inventory", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"inventory": getInventory(),
		})
	})
	r.Run()
}
```

In a terminal, let's run the inventory server and make sure the entire system works:

```bash
go run inventory.go
[GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

[GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
 - using env:   export GIN_MODE=release
 - using code:  gin.SetMode(gin.ReleaseMode)

[GIN-debug] GET    /inventory                --> main.main.func1 (3 handlers)
[GIN-debug] Environment variable PORT is undefined. Using port :8080 by default
[GIN-debug] Listening and serving HTTP on :8080
```

This time, running `shopper.py` should output the inventory:

```bash
python shopper.py
Welcome to the grocery store!
{"inventory":{"products":[{"name":"potato","price":0.99,"id":"1"},{"name":"apple","price":0.5,"id":"2"},{"name":"mango","price":1.5,"id":"3"}]}}
```

Great! As we did in [Part 1](https://words.boten.ca/Practical-OpenTelemetry-part-1-Python/), we'll need to configure the exporter and tracer provider OpenTelemetry components to start tracing our application. Since we used Jaeger previously, we'll skip straight to that exporter in this example. The Go OpenTelemetry implementation provides a nice convenience method to configure the exporter and tracer provider with a single call `InstallNewPipeline`:

```go
"go.opentelemetry.io/otel/exporters/trace/jaeger"
...
func initTracer() {
	_, err := jaeger.InstallNewPipeline(
		jaeger.WithAgentEndpoint("localhost:6831"),
		jaeger.WithProcess(jaeger.Process{ServiceName: "inventory"}),
	)
	if err != nil {
		log.Fatal(err)
	}
}

func main() {
  initTracer()
...
```

Next, as we did before with the FlaskInstrumentor in the Python code, we'll use an instrumentation package for Gin available in the [opentelemetry-go-contrib](https://github.com/open-telemetry/opentelemetry-go-contrib) repo. Using the `otelgin` middleware, our server will be instrumented as follows:

```go
"go.opentelemetry.io/contrib/instrumentation/github.com/gin-gonic/gin/otelgin"
...
func main() {
	initTracer()

	r := gin.Default()
	r.Use(otelgin.Middleware("inventory-server"))
...
```

Let's try restarting the server and looking at the resulting trace, which should combine all our services.

![two-services](/assets/images/otel-part-2-2.png)

Hmm, that doesn't look right, what happened? We see the connected trace for the `shopper` and the `grocery_store` as we did before, but we should have seen a third service in that trace. Instead, it looks like the `inventory` service is not connected to anything:

![inventory-service](/assets/images/otel-part-2-3.png)

It looks like we've got ourselve an old fashioned propagation problem on our hands!

## Propagation

One thing we didn't talk about in part 1 is how to traces are connected. Distributed traces are connected via [Context Propagation](https://lightstep.com/blog/opentelemetry-context-propagation/). Very briefly, each time one service makes a request to another, a unique identifier (TraceID) is sent along with the request. This unique identifier is what allows a backend, Jaeger in this case, to connect the dots and give us a picture of what the execution path through our system is. For more information about Context Propagation, I highly recommend Ted Young's article linked above, it does a great job of explaining it.

In Part 1, propagation was done automatically for us because the Python API enables TraceContext propagation by default and the instrumentation libraries we used were able to use this propagator to pass along the TraceID. We need to tell our inventory service to use TraceContext propagation, which the Gin instrumentation library will use to extract the TraceID from.

```go
...
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/trace/jaeger"
	"go.opentelemetry.io/otel/propagation"
...
func initTracer() {
	otel.SetTextMapPropagator(propagation.TraceContext{})
...
```

With this we're ready to try sending our shopper to the store once more and look at the resulting trace.

![all-services](/assets/images/otel-part-2-4.png)

As we can see now, our trace is connecting all three services! Hurray!!! We've added one more service and explored another OpenTelemetry implementation along the way! All the code for this series can be found in this [GitHub repo](https://github.com/codeboten/practical-otel).

------

Photo by [Kiarash Mansouri](https://unsplash.com/@kiarash_mansouri?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)