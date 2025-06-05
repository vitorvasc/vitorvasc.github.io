---
layout: post
title: "7 Days of OpenTelemetry: Day 6 - Context Propagation and Logs Correlation"
date: 2025-05-31
author: "Vitor Vasconcellos"
lang: en
img: ":2025-05-31-7-days-otel-day-6-context-propagation-and-logs-correlation.png"
categories: [OpenTelemetry, Observability, 7 Days of OpenTelemetry]
tags: [opentelemetry, observability, 7-days-of-opentelemetry]
---

## Day 6: Context Propagation and Logs Correlation

Welcome to Day 6 of our "7 Days of OpenTelemetry" challenge! In the previous days, we've explored the fundamentals of observability, set up the OpenTelemetry Collector, and implemented both manual and automatic instrumentation. Today, we'll dive into two critical aspects of distributed tracing: context propagation and logs correlation.

### The Challenge of Distributed Tracing

In a microservices architecture, a single user request might span multiple services. To create a complete trace of this request, we need to:

1. **Propagate context** between services, so each service knows it's handling part of the same request
2. **Correlate logs** with traces, so we can see the detailed logs associated with each span

Without these capabilities, we'd have disconnected spans and logs, making it difficult to understand the full picture of a request's journey through our system.

### Understanding Context Propagation

Context propagation is the mechanism that connects spans across service boundaries. It works by passing trace information (trace ID, span ID, etc.) from one service to another, typically through HTTP headers, message queues, or other inter-service communication channels.

### How Context Propagation Works

1. **Service A** creates a span and adds trace context to outgoing requests
2. **Service B** extracts the trace context from incoming requests and creates child spans
3. The spans from both services are connected in the resulting trace

This process creates a complete trace that spans multiple services, giving you end-to-end visibility into the request's journey.

### W3C Trace Context Standard

To ensure interoperability between different tracing systems, the W3C has defined a standard for trace context propagation called [W3C Trace Context](https://www.w3.org/TR/trace-context/). This standard defines:

1. **traceparent**: Contains the trace ID, span ID, and trace flags
2. **tracestate**: Allows vendors to add custom information

OpenTelemetry implements this standard by default, ensuring compatibility with other systems that follow the same standard.

### Implementing Context Propagation

Let's create a simple example with two services that communicate via HTTP to demonstrate context propagation:

1. **Frontend Service**: Receives user requests and calls the Backend Service
2. **Backend Service**: Processes requests from the Frontend Service

#### Setting Up the Project

First, let's create the necessary directories:

```bash
mkdir -p otel-context/cmd/frontend
mkdir -p otel-context/cmd/backend
cd otel-context
```

Initialize a Go module:

```bash
go mod init github.com/yourusername/otel-context
```

Add the necessary dependencies:

```bash
go get go.opentelemetry.io/otel \
       go.opentelemetry.io/otel/trace \
       go.opentelemetry.io/otel/sdk \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc \
       go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp
```

#### Creating a Shared Telemetry Package

Let's create a shared package for telemetry initialization. Create `internal/telemetry/telemetry.go`:

```go
package telemetry

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

// InitTracer initializes the OpenTelemetry tracer
func InitTracer(serviceName string) func(context.Context) error {
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
			semconv.ServiceNameKey.String(serviceName),
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
	
	// Set the global propagator to propagate context between services
	otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
		propagation.TraceContext{},  // W3C Trace Context
		propagation.Baggage{},       // W3C Baggage
	))

	// Return a function to shut down the exporter when the application exits
	return func(ctx context.Context) error {
		ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
		defer cancel()
		return tp.Shutdown(ctx)
	}
}
```

This code is similar to our previous telemetry initialization, but it takes a service name parameter and explicitly sets up the W3C Trace Context propagator.

#### Implementing the Backend Service

Now, let's implement the Backend Service. Create `cmd/backend/main.go`:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"github.com/yourusername/otel-context/internal/telemetry"
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/trace"
)

const tracerName = "github.com/yourusername/otel-context/backend"

func main() {
	// Initialize the tracer
	shutdown := telemetry.InitTracer("otel-context-backend")
	defer func() {
		if err := shutdown(context.Background()); err != nil {
			log.Fatalf("Error shutting down tracer: %v", err)
		}
	}()

	// Set up HTTP handlers with automatic instrumentation
	http.Handle("/process", otelhttp.NewHandler(http.HandlerFunc(handleProcess), "handleProcess"))

	// Start the server in a goroutine
	server := &http.Server{Addr: ":8081"}
	go func() {
		fmt.Println("Backend server starting on :8081...")
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server failed to start: %v", err)
		}
	}()

	// Wait for interrupt signal
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, os.Interrupt)
	<-sigCh

	fmt.Println("Shutting down...")
	
	// Gracefully shut down the server
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(ctx); err != nil {
		log.Fatalf("Server shutdown failed: %v", err)
	}
}

func handleProcess(w http.ResponseWriter, r *http.Request) {
	// Get the tracer
	tracer := otel.Tracer(tracerName)
	
	// The otelhttp handler has already created a span for us
	// We can access it through the request context
	ctx := r.Context()
	span := trace.SpanFromContext(ctx)
	
	// Log the trace and span IDs
	traceID := span.SpanContext().TraceID().String()
	spanID := span.SpanContext().SpanID().String()
	log.Printf("[TraceID: %s, SpanID: %s] Processing request", traceID, spanID)
	
	// Create a child span for processing
	ctx, processSpan := tracer.Start(ctx, "processData")
	defer processSpan.End()
	
	// Extract the data from the request
	var data map[string]interface{}
	if err := json.NewDecoder(r.Body).Decode(&data); err != nil {
		processSpan.RecordError(err)
		http.Error(w, err.Error(), http.StatusBadRequest)
		log.Printf("[TraceID: %s, SpanID: %s] Error decoding request: %v", 
			traceID, spanID, err)
		return
	}
	
	// Add attributes to the span
	processSpan.SetAttributes(
		attribute.String("data.id", fmt.Sprintf("%v", data["id"])),
		attribute.String("data.value", fmt.Sprintf("%v", data["value"])),
	)
	
	// Log with trace context
	log.Printf("[TraceID: %s, SpanID: %s] Processing data: %v", 
		traceID, processSpan.SpanContext().SpanID().String(), data)
	
	// Simulate processing
	time.Sleep(100 * time.Millisecond)
	
	// Create a result
	result := map[string]interface{}{
		"id":      data["id"],
		"value":   data["value"],
		"result":  fmt.Sprintf("Processed %v", data["value"]),
		"traceID": traceID,
	}
	
	// Log the result with trace context
	log.Printf("[TraceID: %s, SpanID: %s] Processing complete: %v", 
		traceID, processSpan.SpanContext().SpanID().String(), result)
	
	// Return the result
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(result)
}
```

This Backend Service:
1. Initializes OpenTelemetry with the service name "otel-context-backend"
2. Sets up an HTTP handler with automatic instrumentation
3. Extracts the trace context from incoming requests
4. Creates child spans for processing
5. Logs with trace and span IDs for correlation

#### Implementing the Frontend Service

Now, let's implement the Frontend Service. Create `cmd/frontend/main.go`:

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"github.com/yourusername/otel-context/internal/telemetry"
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/trace"
)

const tracerName = "github.com/yourusername/otel-context/frontend"

func main() {
	// Initialize the tracer
	shutdown := telemetry.InitTracer("otel-context-frontend")
	defer func() {
		if err := shutdown(context.Background()); err != nil {
			log.Fatalf("Error shutting down tracer: %v", err)
		}
	}()

	// Create an HTTP client with automatic instrumentation
	client := &http.Client{
		Transport: otelhttp.NewTransport(http.DefaultTransport),
	}

	// Set up HTTP handlers with automatic instrumentation
	http.Handle("/", otelhttp.NewHandler(http.HandlerFunc(handleRoot), "handleRoot"))
	http.Handle("/api", otelhttp.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		handleAPI(w, r, client)
	}), "handleAPI"))

	// Start the server in a goroutine
	server := &http.Server{Addr: ":8080"}
	go func() {
		fmt.Println("Frontend server starting on :8080...")
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server failed to start: %v", err)
		}
	}()

	// Wait for interrupt signal
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, os.Interrupt)
	<-sigCh

	fmt.Println("Shutting down...")
	
	// Gracefully shut down the server
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(ctx); err != nil {
		log.Fatalf("Server shutdown failed: %v", err)
	}
}

func handleRoot(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Welcome to the OpenTelemetry Context Propagation Demo!\n")
	fmt.Fprintf(w, "Try calling /api?id=123&value=test\n")
}

func handleAPI(w http.ResponseWriter, r *http.Request, client *http.Client) {
	// Get the tracer
	tracer := otel.Tracer(tracerName)
	
	// The otelhttp handler has already created a span for us
	// We can access it through the request context
	ctx := r.Context()
	span := trace.SpanFromContext(ctx)
	
	// Log the trace and span IDs
	traceID := span.SpanContext().TraceID().String()
	spanID := span.SpanContext().SpanID().String()
	log.Printf("[TraceID: %s, SpanID: %s] Handling API request", traceID, spanID)
	
	// Get query parameters
	id := r.URL.Query().Get("id")
	if id == "" {
		id = "unknown"
	}
	
	value := r.URL.Query().Get("value")
	if value == "" {
		value = "default"
	}
	
	// Add attributes to the span
	span.SetAttributes(
		attribute.String("request.id", id),
		attribute.String("request.value", value),
	)
	
	// Create a child span for calling the backend
	ctx, backendSpan := tracer.Start(ctx, "callBackend")
	defer backendSpan.End()
	
	// Log with trace context
	log.Printf("[TraceID: %s, SpanID: %s] Calling backend with id=%s, value=%s", 
		traceID, backendSpan.SpanContext().SpanID().String(), id, value)
	
	// Prepare the request to the backend
	data := map[string]interface{}{
		"id":    id,
		"value": value,
	}
	jsonData, err := json.Marshal(data)
	if err != nil {
		backendSpan.RecordError(err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
		log.Printf("[TraceID: %s, SpanID: %s] Error marshaling data: %v", 
			traceID, backendSpan.SpanContext().SpanID().String(), err)
		return
	}
	
	// Create the request
	req, err := http.NewRequestWithContext(ctx, "POST", "http://localhost:8081/process", bytes.NewBuffer(jsonData))
	if err != nil {
		backendSpan.RecordError(err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
		log.Printf("[TraceID: %s, SpanID: %s] Error creating request: %v", 
			traceID, backendSpan.SpanContext().SpanID().String(), err)
		return
	}
	req.Header.Set("Content-Type", "application/json")
	
	// Send the request to the backend
	// The otelhttp client will automatically propagate the trace context
	resp, err := client.Do(req)
	if err != nil {
		backendSpan.RecordError(err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
		log.Printf("[TraceID: %s, SpanID: %s] Error calling backend: %v", 
			traceID, backendSpan.SpanContext().SpanID().String(), err)
		return
	}
	defer resp.Body.Close()
	
	// Read the response
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		backendSpan.RecordError(err)
		http.Error(w, err.Error(), http.StatusInternalServerError)
		log.Printf("[TraceID: %s, SpanID: %s] Error reading response: %v", 
			traceID, backendSpan.SpanContext().SpanID().String(), err)
		return
	}
	
	// Log the response with trace context
	log.Printf("[TraceID: %s, SpanID: %s] Received response: %s", 
		traceID, backendSpan.SpanContext().SpanID().String(), string(body))
	
	// Return the response to the client
	w.Header().Set("Content-Type", "application/json")
	w.Write(body)
}
```

This Frontend Service:
1. Initializes OpenTelemetry with the service name "otel-context-frontend"
2. Creates an HTTP client with automatic instrumentation
3. Sets up HTTP handlers with automatic instrumentation
4. Creates spans for processing and calling the backend
5. Uses the instrumented HTTP client to propagate trace context to the backend
6. Logs with trace and span IDs for correlation

### Running the Services

Let's run both services. In one terminal, start the Backend Service:

```bash
go run cmd/backend/main.go
```

In another terminal, start the Frontend Service:

```bash
go run cmd/frontend/main.go
```

Now, make a request to the Frontend Service:

```bash
curl "http://localhost:8080/api?id=123&value=test"
```

You should see logs in both terminals with the same trace ID, indicating that the context has been propagated between the services.

### Viewing Traces in the Collector

Check the Collector logs to see the distributed trace:

```bash
docker-compose -f otel-collector/docker-compose.yaml logs
```

You should see a single trace that spans both services, with spans for:
1. The HTTP request to the Frontend Service
2. The processing in the Frontend Service
3. The HTTP request from the Frontend to the Backend Service
4. The processing in the Backend Service

This demonstrates successful context propagation between services.

## Understanding Logs Correlation

While distributed tracing provides a high-level view of a request's journey, logs provide detailed information about what happened during each step. Correlating logs with traces allows you to see the logs associated with each span, giving you both the big picture and the details.

### Why Logs Correlation Matters

Imagine you're investigating a performance issue in a distributed system:

1. **Traces** show you which service is slow
2. **Logs** tell you why it's slow (e.g., a database query is taking too long)

Without correlation, you'd have to manually match logs to traces, which is time-consuming and error-prone.

### How Logs Correlation Works

Logs correlation works by including trace and span IDs in log entries. This allows you to:

1. Find all logs associated with a specific trace
2. See the logs in the context of the trace timeline
3. Understand what happened during each span

In our example, we've already implemented basic logs correlation by including trace and span IDs in our log messages:

```go
log.Printf("[TraceID: %s, SpanID: %s] Processing request", traceID, spanID)
```

### Implementing Structured Logging with Trace IDs

While our basic approach works, a more robust solution is to use a structured logging library that supports OpenTelemetry integration. Let's update our Backend Service to use the `zap` logging library with OpenTelemetry integration.

First, add the necessary dependencies:

```bash
go get go.uber.org/zap
```

Now, create a new file `internal/logging/logging.go`:

```go
package logging

import (
	"context"

	"go.opentelemetry.io/otel/trace"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

// NewLogger creates a new zap logger
func NewLogger() (*zap.Logger, error) {
	config := zap.NewProductionConfig()
	config.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
	return config.Build()
}

// WithTraceContext adds trace context to a zap logger
func WithTraceContext(ctx context.Context, logger *zap.Logger) *zap.Logger {
	span := trace.SpanFromContext(ctx)
	if !span.SpanContext().IsValid() {
		return logger
	}
	
	traceID := span.SpanContext().TraceID().String()
	spanID := span.SpanContext().SpanID().String()
	
	return logger.With(
		zap.String("traceID", traceID),
		zap.String("spanID", spanID),
	)
}
```

Now, update the Backend Service to use this structured logging:

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"time"

	"github.com/yourusername/otel-context/internal/logging"
	"github.com/yourusername/otel-context/internal/telemetry"
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/trace"
	"go.uber.org/zap"
)

const tracerName = "github.com/yourusername/otel-context/backend"

var logger *zap.Logger

func main() {
	// Initialize the logger
	var err error
	logger, err = logging.NewLogger()
	if err != nil {
		fmt.Printf("Failed to create logger: %v\n", err)
		os.Exit(1)
	}
	defer logger.Sync()

	// Initialize the tracer
	shutdown := telemetry.InitTracer("otel-context-backend")
	defer func() {
		if err := shutdown(context.Background()); err != nil {
			logger.Error("Error shutting down tracer", zap.Error(err))
		}
	}()

	// Set up HTTP handlers with automatic instrumentation
	http.Handle("/process", otelhttp.NewHandler(http.HandlerFunc(handleProcess), "handleProcess"))

	// Start the server in a goroutine
	server := &http.Server{Addr: ":8081"}
	go func() {
		logger.Info("Backend server starting", zap.String("address", ":8081"))
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Fatal("Server failed to start", zap.Error(err))
		}
	}()

	// Wait for interrupt signal
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, os.Interrupt)
	<-sigCh

	logger.Info("Shutting down...")
	
	// Gracefully shut down the server
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(ctx); err != nil {
		logger.Fatal("Server shutdown failed", zap.Error(err))
	}
}

func handleProcess(w http.ResponseWriter, r *http.Request) {
	// Get the tracer
	tracer := otel.Tracer(tracerName)
	
	// The otelhttp handler has already created a span for us
	// We can access it through the request context
	ctx := r.Context()
	
	// Get a logger with trace context
	log := logging.WithTraceContext(ctx, logger)
	
	log.Info("Processing request")
	
	// Create a child span for processing
	ctx, processSpan := tracer.Start(ctx, "processData")
	defer processSpan.End()
	
	// Update the logger with the new span context
	log = logging.WithTraceContext(ctx, logger)
	
	// Extract the data from the request
	var data map[string]interface{}
	if err := json.NewDecoder(r.Body).Decode(&data); err != nil {
		processSpan.RecordError(err)
		http.Error(w, err.Error(), http.StatusBadRequest)
		log.Error("Error decoding request", zap.Error(err))
		return
	}
	
	// Add attributes to the span
	processSpan.SetAttributes(
		attribute.String("data.id", fmt.Sprintf("%v", data["id"])),
		attribute.String("data.value", fmt.Sprintf("%v", data["value"])),
	)
	
	// Log with trace context
	log.Info("Processing data", zap.Any("data", data))
	
	// Simulate processing
	time.Sleep(100 * time.Millisecond)
	
	// Create a result
	result := map[string]interface{}{
		"id":      data["id"],
		"value":   data["value"],
		"result":  fmt.Sprintf("Processed %v", data["value"]),
		"traceID": trace.SpanFromContext(ctx).SpanContext().TraceID().String(),
	}
	
	// Log the result with trace context
	log.Info("Processing complete", zap.Any("result", result))
	
	// Return the result
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(result)
}
```

This updated Backend Service:
1. Uses the `zap` logging library for structured logging
2. Adds trace and span IDs to log entries automatically
3. Updates the logger context when creating new spans

The logs will now include trace and span IDs in a structured format, making it easier to correlate them with traces.

### Best Practices for Context Propagation and Logs Correlation

Based on our example, here are some best practices:

#### Context Propagation

1. **Use Standard Propagators**: Stick to standard propagators like W3C Trace Context for interoperability.

2. **Propagate Context Across All Boundaries**: Ensure context is propagated across all service boundaries, including HTTP, gRPC, message queues, etc.

3. **Use Automatic Instrumentation When Possible**: Libraries like `otelhttp` handle context propagation automatically.

4. **Maintain Context in Asynchronous Operations**: Pass context to goroutines, workers, and other asynchronous operations.

5. **Test Context Propagation**: Verify that trace IDs are consistent across services.

#### Logs Correlation

1. **Include Trace and Span IDs in Logs**: Always include these IDs to enable correlation.

2. **Use Structured Logging**: Structured logs are easier to parse and correlate.

3. **Update Logger Context with Span Changes**: When creating new spans, update the logger context.

4. **Consider Log Levels**: Use appropriate log levels to avoid overwhelming your logging system.

5. **Include Relevant Context**: Add other relevant information to logs, such as user IDs, request IDs, etc.

### Cross-Language Implementation

While our example uses Go, context propagation and logs correlation work similarly in other languages. Here's a quick comparison:

#### Java
```java
// Context propagation with Spring Boot
@Bean
public WebClient.Builder webClientBuilder() {
    return WebClient.builder()
        .filter(new TracingExchangeFilterFunction());
}

// Logs correlation with SLF4J
private static final Logger logger = LoggerFactory.getLogger(MyClass.class);

void processRequest(Context context) {
    Span span = Span.current();
    MDC.put("traceId", span.getSpanContext().getTraceId());
    MDC.put("spanId", span.getSpanContext().getSpanId());
    logger.info("Processing request");
}
```

#### Python
```python
# Context propagation with Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)

# HTTP client with context propagation
session = requests.Session()
RequestsInstrumentor().instrument_session(session)

# Logs correlation
def process_request():
    span = trace.get_current_span()
    trace_id = span.get_span_context().trace_id
    span_id = span.get_span_context().span_id
    logger.info(f"Processing request", extra={"trace_id": trace_id, "span_id": span_id})
```

#### JavaScript (Node.js)
```javascript
// Context propagation with Express
const app = express();
const opentelemetry = require('@opentelemetry/api');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');

const expressInstrumentation = new ExpressInstrumentation();
const httpInstrumentation = new HttpInstrumentation();
expressInstrumentation.enable();
httpInstrumentation.enable();

// Logs correlation
function processRequest(req, res) {
    const span = opentelemetry.trace.getSpan(opentelemetry.context.active());
    const traceId = span.spanContext().traceId;
    const spanId = span.spanContext().spanId;
    console.log(`[TraceID: ${traceId}, SpanID: ${spanId}] Processing request`);
}
```

The specific APIs vary, but the concepts are the same: propagate context between services and include trace and span IDs in logs.

### Conclusion

In this installment, we've explored context propagation and logs correlation, which are essential for creating a complete observability picture in distributed systems. We've learned how to:

1. Propagate context between services using W3C Trace Context
2. Create distributed traces that span multiple services
3. Correlate logs with traces using trace and span IDs
4. Implement structured logging with trace context

These capabilities allow you to track requests across service boundaries and see the detailed logs associated with each span, giving you both the big picture and the details.

In tomorrow's final installment, we'll explore visualization and analysis, connecting our telemetry data to proper visualization tools and learning how to derive insights from it.

Stay tuned, and happy tracing!
