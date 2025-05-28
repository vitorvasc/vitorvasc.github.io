---
layout: post
title: "7 Days of OpenTelemetry: Day 3 - Setting Up the OpenTelemetry Collector"
date: 2025-05-27
author: "Vitor Vasconcellos"
lang: en
img: ":2025-05-27-7-days-otel-day-3-setting-up-collector.png"
categories: [OpenTelemetry, Observability, 7 Days of OpenTelemetry]
tags: [opentelemetry, observability, 7-days-of-opentelemetry]
---

## Day 3: Setting Up the OpenTelemetry Collector

Welcome to Day 3 of our "7 Days of OpenTelemetry" challenge! In the previous days, we covered the fundamentals of observability and distributed tracing. Today, we're taking our first practical step by setting up the OpenTelemetry Collector, which will form the foundation of our observability pipeline.

### What is the OpenTelemetry Collector?

The OpenTelemetry Collector is a vendor-agnostic component that can receive, process, and export telemetry data. It serves as a central hub in your observability infrastructure, providing a unified way to:

1. **Receive** telemetry data from various sources
2. **Process** that data (filtering, batching, etc.)
3. **Export** the data to one or more backends

The Collector acts as a middleware between your instrumented applications and your observability backends, decoupling them and providing flexibility in your telemetry pipeline.

### Why Use the Collector?

You might wonder why we need the Collector when we could export telemetry data directly from our applications to our chosen backends. Here are some compelling reasons:

1. **Reduced Dependency**: Applications only need to know how to send data to the Collector, not to multiple backends
2. **Consistent Configuration**: Centralized configuration for processing and exporting telemetry
3. **Resource Efficiency**: Offloads telemetry processing from your application
4. **Buffering and Retries**: Handles temporary backend outages without data loss
5. **Preprocessing**: Can transform, filter, or enrich data before it reaches backends
6. **Protocol Translation**: Converts between telemetry formats and protocols

By setting up the Collector first, we'll have immediate visibility into the telemetry data we generate in the coming days, even before connecting to a full-featured visualization tool.

### Collector Architecture

The OpenTelemetry Collector has a modular architecture built around three types of components:

1. **Receivers**: Accept telemetry data from multiple sources
2. **Processors**: Transform, filter, or enrich the data
3. **Exporters**: Send data to multiple destinations

These components are connected in pipelines that define the flow of telemetry data through the Collector.

#### Receivers

Receivers accept data in multiple formats and protocols. Some common receivers include:

- **OTLP**: The OpenTelemetry Protocol receiver (the standard protocol for OpenTelemetry)
- **Jaeger**: Accepts data in Jaeger format
- **Zipkin**: Accepts data in Zipkin format
- **Prometheus**: Scrapes Prometheus metrics

#### Processors

Processors manipulate telemetry data as it passes through the Collector. Examples include:

- **Batch**: Groups data into larger batches for more efficient export
- **Memory Limiter**: Prevents the Collector from consuming too much memory
- **Attribute**: Modifies attributes on telemetry data
- **Sampling**: Reduces the volume of trace data by sampling

#### Exporters

Exporters send data to multiple backends. Common exporters include:

- **OTLP**: Sends data using the OpenTelemetry Protocol
- **Jaeger**: Exports to Jaeger backends
- **Zipkin**: Exports to Zipkin backends
- **Prometheus**: Exports metrics in Prometheus format
- **File**: Writes telemetry data to a file
- **Logging**: Outputs telemetry data to the console (useful for debugging)

### Deployment Patterns

The Collector can be deployed in different patterns, depending on your needs:

1. **Agent**: Runs alongside your application (as a sidecar or on the same host)
2. **Gateway**: Runs as a standalone service that receives data from multiple agents
3. **Standalone**: A single Collector instance that handles both collection and export

For our challenge, we'll use the standalone pattern for simplicity, but in production environments, a combination of agents and gateways is often used for scalability and reliability.

### Setting Up the Collector

Let's set up a basic OpenTelemetry Collector with a debug exporter that will show us the telemetry data we generate in the coming days.

#### Step 1: Create a Configuration File

First, let's create a directory for our Collector configuration:

```bash
mkdir -p otel-collector/config
cd otel-collector
```

Now, create a configuration file named `config.yaml` in the `config` directory:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

exporters:
  debug:
    verbosity: detailed
  
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, logging]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, logging]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, logging]
```

This configuration:

1. Sets up an OTLP receiver that accepts data via gRPC (port 4317) and HTTP (port 4318)
2. Configures a batch processor to group telemetry data for more efficient processing
3. Sets up two exporters:
   - A debug exporter that prints detailed telemetry data
   - A logging exporter that outputs to the console
4. Defines pipelines for traces, metrics, and logs, connecting the receivers, processors, and exporters

#### Step 2: Run the Collector with Docker

The easiest way to run the Collector is with Docker. Create a `docker-compose.yaml` file in the `otel-collector` directory:

```yaml
version: '3'
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector/config.yaml"]
    volumes:
      - ./config:/etc/otel-collector
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    restart: unless-stopped
```

This Docker Compose file:

1. Uses the official OpenTelemetry Collector Contrib image
2. Mounts our configuration directory
3. Exposes the necessary ports for OTLP receivers
4. Configures the container to restart automatically unless explicitly stopped

#### Step 3: Start the Collector

Now, let's start the Collector:

```bash
docker-compose up -d
```

This command starts the Collector in detached mode, running in the background.

#### Step 4: Verify the Collector is Running

Let's check that the Collector is running correctly:

```bash
docker-compose logs
```

You should see output indicating that the Collector has started successfully, with messages about the configured receivers, processors, and exporters.

#### Step 5: Test the Collector

To verify that our Collector is working correctly, we can send a test trace using a simple curl command:

```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [
      {
        "resource": {
          "attributes": [
            {
              "key": "service.name",
              "value": { "stringValue": "test-service" }
            }
          ]
        },
        "scopeSpans": [
          {
            "scope": {
              "name": "test-scope"
            },
            "spans": [
              {
                "traceId": "5b8aa5a2d2a9fb5a5b8aa5a2d2a9fb5a",
                "spanId": "5b8aa5a2d2a9fb5a",
                "name": "test-span",
                "kind": 1,
                "startTimeUnixNano": "1644238316010000000",
                "endTimeUnixNano": "1644238316020000000",
                "attributes": [
                  {
                    "key": "test.attribute",
                    "value": { "stringValue": "test-value" }
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }'
```

After sending this request, check the Collector logs again:

```bash
docker-compose logs
```

You should see the test trace in the logs, indicating that the Collector is correctly receiving and processing telemetry data.

### Understanding the Collector Configuration

Let's take a closer look at the configuration we created:

#### Receivers Configuration

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
```

This section configures the OTLP receiver to accept data via both gRPC and HTTP protocols. The `0.0.0.0` address means it will accept connections from any network interface.

#### Processors Configuration

```yaml
processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
```

The batch processor groups telemetry data into batches before sending it to exporters. This improves efficiency by reducing the number of export operations. The configuration specifies:

- `timeout`: Maximum time to wait before sending a batch
- `send_batch_size`: Maximum number of items in a batch

#### Exporters Configuration

```yaml
exporters:
  debug:
    verbosity: detailed
  
  logging:
    loglevel: debug
```

We've configured two exporters:

1. The debug exporter, which prints detailed information about the telemetry data
2. The logging exporter, which outputs to the console with debug-level verbosity

#### Pipelines Configuration

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, logging]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, logging]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, logging]
```

This section defines three pipelines, one for each telemetry signal type (traces, metrics, and logs). Each pipeline connects the OTLP receiver to the batch processor and then to both the debug and logging exporters.

### Collector Deployment Patterns

While we're using a standalone Collector for this challenge, it's worth understanding the common deployment patterns for production environments:

#### Agent Pattern

In the agent pattern, a Collector instance runs alongside each application, typically as a sidecar container in Kubernetes or on the same host for traditional deployments. The agent:

- Receives telemetry data from the local application
- Performs initial processing (batching, filtering, etc.)
- Forwards the data to a gateway Collector or directly to backends

This pattern reduces network traffic and provides isolation between applications.

#### Gateway Pattern

In the gateway pattern, one or more Collector instances run as a centralized service that receives data from multiple agents. The gateway:

- Aggregates telemetry data from multiple sources
- Performs additional processing
- Exports the data to one or more backends

This pattern provides centralized control and can reduce the number of connections to backends.

#### Combined Pattern

In practice, many organizations use a combination of agents and gateways, creating a hierarchical collection architecture that balances local processing with centralized control.

### What's Next?

Now that we have our Collector up and running, we're ready to start generating telemetry data. In tomorrow's installment, we'll implement our first OpenTelemetry instrumentation in a Go application and see the resulting traces in our Collector's debug output.

Having the Collector already set up will give us immediate feedback as we implement instrumentation, making the learning process more interactive and rewarding.

### Conclusion

The OpenTelemetry Collector is a powerful component that provides a vendor-agnostic way to receive, process, and export telemetry data. By setting it up early in our observability journey, we've created a foundation that will make the rest of our implementation more straightforward and provide immediate visibility into our telemetry data.

In Day 4, we'll build on this foundation by implementing our first OpenTelemetry instrumentation in a Go application and seeing the resulting traces in our Collector's debug output.

Stay tuned, and happy observing!
