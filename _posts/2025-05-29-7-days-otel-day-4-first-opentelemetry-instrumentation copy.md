---
layout: post
title: "7 Days of OpenTelemetry: Day 4 - Your First OpenTelemetry Instrumentation"
date: 2025-05-29
author: "Vitor Vasconcellos"
lang: en
img: ":2025-05-29-7-days-otel-day-4-first-opentelemetry-instrumentation.png"
categories: [OpenTelemetry, Observability, 7 Days of OpenTelemetry]
tags: [opentelemetry, observability, 7-days-of-opentelemetry]
---

## Day 4: Your First OpenTelemetry Instrumentation

Welcome to Day 4 of our "7 Days of OpenTelemetry" challenge! In the previous days, we covered the fundamentals of observability and distributed tracing, and we set up the OpenTelemetry Collector. Today, we'll implement our first OpenTelemetry instrumentation in a Go application and see the resulting traces in our Collector's debug output.

### Manual Instrumentation Basics

OpenTelemetry provides two main approaches to instrumentation:

1. **Manual instrumentation**: Explicitly adding tracing code to your application
2. **Automatic instrumentation**: Using pre-built integrations for common frameworks and libraries

We'll start with manual instrumentation to understand the core concepts, then explore automatic instrumentation in tomorrow's installment.

### Setting Up a Simple Go Application

Let's create a simple HTTP server in Go that we'll instrument with OpenTelemetry. First, create a new directory for our application:

```bash
mkdir -p otel-demo/cmd/server
cd otel-demo
```

Initialize a Go module:

```bash
go mod init github.com/yourusername/otel-demo
```

Now, create a basic HTTP server in `cmd/server/main.go`:

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"
)

func main() {
	http.HandleFunc("/", handleRoot)
	http.HandleFunc("/hello", handleHello)

	fmt.Println("Server starting on :8080...")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatalf("Server failed to start: %v", err)
	}
}

func handleRoot(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Welcome to the OpenTelemetry demo!\n")
}

func handleHello(w http.ResponseWriter, r *http.Request) {
	// Simulate some work
	time.Sleep(50 * time.Millisecond)
	
	name := r.URL.Query().Get("name")
	if name == "" {
		name = "Guest"
	}
	
	// Simulate a database call
	userID := fetchUserID(name)
	
	fmt.Fprintf(w, "Hello, %s (ID: %s)!\n", name, userID)
}

func fetchUserID(name string) string {
	// Simulate a database query
	time.Sleep(100 * time.Millisecond)
	return fmt.Sprintf("user-%d", len(name)*7)
}
```

This simple server has two endpoints:
- `/`: Returns a welcome message
- `/hello`: Takes an optional `name` parameter, simulates some work, and returns a greeting

Let's run the server to make sure it works:

```bash
go run cmd/server/main.go
```

You should see "Server starting on :8080..." in the console. In a separate terminal, test the server:

```bash
curl http://localhost:8080/
curl http://localhost:8080/hello?name=Alice
```

You should see the expected responses. Now, let's add OpenTelemetry instrumentation to this application.

### Adding OpenTelemetry Dependencies

First, we need to add the necessary OpenTelemetry packages to our Go module:

```bash
go get go.opentelemetry.io/otel \
       go.opentelemetry.io/otel/trace \
       go.opentelemetry.io/otel/sdk \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc
```

These packages provide:
- The OpenTelemetry API
- The tracing API
- The OpenTelemetry SDK
- The OTLP exporter for traces
- The gRPC implementation of the OTLP exporter

### Initializing the OpenTelemetry SDK

Now, let's create a new file `cmd/server/telemetry.go` to initialize the OpenTelemetry SDK:

```go
package main

import (
	"context"
	"log"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.4.0"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

// initTracer initializes the OpenTelemetry tracer
func initTracer() func(context.Context) error {
	// Create a new OTLP exporter
	ctx := context.Background()
	
	// Configure the exporter to use gRPC and connect to the Collector
	exporter, err := otlptrace.New(
		ctx,
		otlptracegrpc.NewClient(
			otlptracegrpc.WithInsecure(),
			otlptracegrpc.WithEndpoint("localhost:4317"),
			otlptracegrpc.WithDialOption(grpc.WithBlock()),
		),
	)
	if err != nil {
		log.Fatalf("Failed to create exporter: %v", err)
	}

	// Configure the resource with service information
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceNameKey.String("otel-demo-service"),
			semconv.ServiceVersionKey.String("0.1.0"),
		),
	)
	if err != nil {
		log.Fatalf("Failed to create resource: %v", err)
	}

	// Configure the trace provider with the exporter and resource
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithSampler(sdktrace.AlwaysSample()),
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
	)

	// Set the global trace provider
	otel.SetTracerProvider(tp)
	
	// Set the global propagator
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{},
		propagation.Baggage{},
	))

	// Return a function to shut down the exporter when the application exits
	return func(ctx context.Context) error {
		ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
		defer cancel()
		return tp.Shutdown(ctx)
	}
}
```

This code:
1. Creates an OTLP exporter that connects to our Collector
2. Configures a resource with service information
3. Creates a trace provider with the exporter and resource
4. Sets the global trace provider and propagator
5. Returns a shutdown function to clean up resources

### Instrumenting the HTTP Server

Now, let's update our `main.go` file to use the tracer:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/trace"
)

// Name of the tracer
const tracerName = "github.com/yourusername/otel-demo"

func main() {
	// Initialize the tracer
	shutdown := initTracer()
	defer func() {
		if err := shutdown(context.Background()); err != nil {
			log.Fatalf("Error shutting down tracer: %v", err)
		}
	}()

	// Set up HTTP handlers
	http.HandleFunc("/", handleRoot)
	http.HandleFunc("/hello", handleHello)

	// Start the server in a goroutine
	go func() {
		fmt.Println("Server starting on :8080...")
		if err := http.ListenAndServe(":8080", nil); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server failed to start: %v", err)
		}
	}()

	// Wait for interrupt signal
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, os.Interrupt)
	<-sigCh

	fmt.Println("Shutting down...")
}

func handleRoot(w http.ResponseWriter, r *http.Request) {
	// Get the tracer
	tracer := otel.Tracer(tracerName)
	
	// Create a span for this handler
	ctx, span := tracer.Start(r.Context(), "handleRoot")
	defer span.End()
	
	// Add attributes to the span
	span.SetAttributes(
		attribute.String("http.method", r.Method),
		attribute.String("http.path", r.URL.Path),
	)
	
	// Use the context with the span
	_ = ctx // In a real app, you'd pass this to downstream functions
	
	fmt.Fprintf(w, "Welcome to the OpenTelemetry demo!\n")
}

func handleHello(w http.ResponseWriter, r *http.Request) {
	// Get the tracer
	tracer := otel.Tracer(tracerName)
	
	// Create a span for this handler
	ctx, span := tracer.Start(r.Context(), "handleHello")
	defer span.End()
	
	// Add attributes to the span
	span.SetAttributes(
		attribute.String("http.method", r.Method),
		attribute.String("http.path", r.URL.Path),
	)
	
	// Simulate some work
	time.Sleep(50 * time.Millisecond)
	
	// Get the name parameter
	name := r.URL.Query().Get("name")
	if name == "" {
		name = "Guest"
	}
	
	// Add the name as an attribute
	span.SetAttributes(attribute.String("user.name", name))
	
	// Record an event
	span.AddEvent("Processing request", trace.WithAttributes(
		attribute.String("event.type", "process"),
		attribute.String("user.name", name),
	))
	
	// Call fetchUserID with the context containing the span
	userID := fetchUserID(ctx, name)
	
	fmt.Fprintf(w, "Hello, %s (ID: %s)!\n", name, userID)
}

func fetchUserID(ctx context.Context, name string) string {
	// Get the tracer
	tracer := otel.Tracer(tracerName)
	
	// Create a child span for the database operation
	ctx, span := tracer.Start(ctx, "fetchUserID")
	defer span.End()
	
	// Add attributes to the span
	span.SetAttributes(attribute.String("db.operation", "query"))
	
	// Simulate a database query
	time.Sleep(100 * time.Millisecond)
	
	// Generate a user ID
	userID := fmt.Sprintf("user-%d", len(name)*7)
	
	// Add the result as an attribute
	span.SetAttributes(attribute.String("db.result", userID))
	
	return userID
}
```

This updated code:
1. Initializes the tracer in the `main` function
2. Creates spans for each handler and the database operation
3. Adds attributes and events to the spans
4. Propagates context between functions

### Running the Instrumented Application

Now, let's run our instrumented application:

```bash
go run cmd/server/*.go
```

You should see "Server starting on :8080..." in the console. In a separate terminal, make some requests to the server:

```bash
curl http://localhost:8080/
curl http://localhost:8080/hello?name=Alice
curl http://localhost:8080/hello?name=Bob
```

### Viewing Traces in the Collector

Now, let's check the Collector logs to see the traces:

```bash
docker-compose -f otel-collector/docker-compose.yaml logs
```

You should see detailed trace information in the logs, including:
- The service name (`otel-demo-service`)
- The span names (`handleRoot`, `handleHello`, `fetchUserID`)
- The attributes we added
- The events we recorded
- The timing information

Congratulations! You've successfully instrumented a Go application with OpenTelemetry and sent traces to the Collector.

### Understanding the Instrumentation

Let's break down the key components of our instrumentation:

#### Tracer Initialization

```go
tracer := otel.Tracer(tracerName)
```

This gets a tracer instance from the global provider, using a name that identifies our module.

#### Span Creation

```go
ctx, span := tracer.Start(r.Context(), "handleHello")
defer span.End()
```

This creates a new span with the given name and makes it a child of any span in the incoming context. The `defer span.End()` ensures the span is properly closed when the function returns.

#### Adding Attributes

```go
span.SetAttributes(
    attribute.String("http.method", r.Method),
    attribute.String("http.path", r.URL.Path),
)
```

Attributes provide additional context about the operation. They're key-value pairs that help you understand what happened during the span.

#### Recording Events

```go
span.AddEvent("Processing request", trace.WithAttributes(
    attribute.String("event.type", "process"),
    attribute.String("user.name", name),
))
```

Events are time-stamped logs within a span. They're useful for recording specific actions or state changes.

#### Context Propagation

```go
userID := fetchUserID(ctx, name)
```

Passing the context to downstream functions allows them to create child spans that are properly connected to the parent span.

### Best Practices for Manual Instrumentation

Based on our example, here are some best practices for manual instrumentation:

1. **Use Descriptive Span Names**: Choose names that clearly identify the operation being performed.

2. **Add Relevant Attributes**: Include attributes that provide context about the operation, such as HTTP method, path, user ID, etc.

3. **Record Significant Events**: Use events to mark important points in the span's lifetime.

4. **Propagate Context**: Always pass the context to downstream functions to maintain the trace.

5. **Handle Errors**: Record errors in spans to make debugging easier.

6. **Use Semantic Conventions**: Follow OpenTelemetry's semantic conventions for attribute names to ensure consistency.

7. **Limit Cardinality**: Be careful with high-cardinality attributes (like user IDs) that can explode the number of unique traces.

8. **Clean Up Resources**: Always close spans and shut down the tracer provider when your application exits.

### Cross-Language Implementation

While our example uses Go, the concepts apply to all languages supported by OpenTelemetry. Here's a quick comparison of how span creation looks in different languages:

#### Go
```go
ctx, span := tracer.Start(ctx, "spanName")
defer span.End()
```

#### Java
```java
Span span = tracer.spanBuilder("spanName").startSpan();
try (Scope scope = span.makeCurrent()) {
    // Your code here
} finally {
    span.end();
}
```

#### Python
```python
with tracer.start_as_current_span("spanName") as span:
    # Your code here
```

#### JavaScript
```javascript
const span = tracer.startSpan("spanName");
try {
    // Your code here
} finally {
    span.end();
}
```

The API details vary, but the core concepts (tracers, spans, attributes, events) are consistent across languages.

### Conclusion

In this installment, we've implemented our first OpenTelemetry instrumentation in a Go application and sent traces to the Collector we set up yesterday. We've learned how to:

1. Initialize the OpenTelemetry SDK
2. Create spans for operations
3. Add attributes and events to spans
4. Propagate context between functions
5. View traces in the Collector

This manual instrumentation gives us complete control over what we trace and what information we include. However, it requires adding code to every operation we want to trace.

In tomorrow's installment, we'll explore automatic instrumentation, which can reduce the amount of code we need to write while still providing comprehensive tracing.

Stay tuned, and happy tracing!
