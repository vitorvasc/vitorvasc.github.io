---
layout: post
title: "7 Days of OpenTelemetry: Day 01 - Introduction to Observability and OpenTelemetry"
date: 2025-05-26
author: "Vitor Vasconcellos"
lang: en
img: ":2025-05-26-7-days-otel-day-01-introduction-to-observability-and-otel.png"
categories: [OpenTelemetry, Observability, 7 Days of OpenTelemetry]
tags: [opentelemetry, observability, 7-days-of-opentelemetry]
---

# Day 1: Introduction to Observability and OpenTelemetry

Welcome to the first day of our "7 Days of OpenTelemetry" challenge! Over the next week, we'll take you from zero to hero with OpenTelemetry, equipping you with the knowledge and skills to implement effective observability in your applications.

## What is Observability?

Before diving into OpenTelemetry, let's understand what observability means and why it's crucial in modern software development.

Observability is the ability to understand a system's internal state by examining its outputs. In software systems, this means having enough information about your application to answer questions like:

- Why is this endpoint slow?
- What's causing this error?
- How is this service behaving when under load?
- What's the impact of a code change on performance?

Traditional monitoring tells you **when** something is wrong, while observability helps you understand **why** it's wrong.

### The Three Pillars of Observability

Observability is typically built on three foundational pillars:

1. **Logs**: Records of discrete events that happened in your system. They're like the journal entries of your application.
   ```go
   log.Printf("User %s logged in successfully", userID)
   ```

2. **Metrics**: Numeric representations of data measured over time. They're like the vital signs of your system.
   ```go
   requestCounter.Inc()
   requestDuration.Observe(duration.Seconds())
   ```

3. **Traces**: Records of a request as it flows through your distributed system, showing the path and timing of each component interaction.
   ```go
   span := tracer.StartSpan("processOrder")
   defer span.End()
   ```

While all three pillars are important, our challenge will focus primarily on tracing, with some coverage of logs correlation.

## The Challenge of Modern Distributed Systems

Modern applications are increasingly distributed, often comprising dozens or hundreds of microservices, serverless functions, and third-party APIs. This architectural shift has created significant challenges for debugging and performance optimization:

- **Complexity**: A single user request might touch dozens of services
- **Polyglot environments**: Different services might be written in different languages
- **Diverse infrastructure**: Applications span multiple clouds, data centers, and edge locations
- **Dynamic scaling**: Services scale up and down automatically based on demand

Traditional monitoring approaches fall short in these environments. When a problem occurs, it's difficult to trace the issue across service boundaries and understand the full context.

## Enter OpenTelemetry

OpenTelemetry is an open-source observability framework designed to solve these challenges. It provides a standardized way to generate, collect, and export telemetry data (logs, metrics, and traces) from your applications and infrastructure.

### A Brief History

OpenTelemetry was formed in 2019 through the merger of two previous projects:
- **OpenTracing**: A vendor-neutral API for distributed tracing
- **OpenCensus**: A collection of language-specific libraries for metrics and traces

This merger created a unified, vendor-neutral approach to observability that has quickly gained industry adoption.

### Key Benefits of OpenTelemetry

1. **Vendor Neutrality**: Collect data once, export it anywhere
2. **Comprehensive Coverage**: Support for all major programming languages and frameworks
3. **Standardization**: Consistent approach across different technologies
4. **Future-Proof**: Evolving with industry best practices
5. **Community-Driven**: Backed by the Cloud Native Computing Foundation (CNCF)

### OpenTelemetry in the CNCF Landscape

[OpenTelemetry](https://opentelemetry.io/){:target="_blank"} is an incubating project in the Cloud Native Computing Foundation (CNCF) [landscape](https://landscape.cncf.io/){:target="_blank"}. It sits alongside other observability projects like [Prometheus](https://prometheus.io/){:target="_blank"} (for metrics), [Fluentd](https://fluentd.org/){:target="_blank"} (for logging) and [Jaeger](https://www.jaegertracing.io/){:target="_blank"} (for tracing), but provides a more comprehensive approach that spans all three observability pillars.

As a CNCF incubating project, OpenTelemetry has demonstrated its potential and is being [adopted](https://opentelemetry.io/ecosystem/adopters/){:target="_blank"} by many companies, including eBay, MercadoLibre, Heroku and many others. 

## OpenTelemetry Components

OpenTelemetry consists of several key components, as outlined in the [official documentation](https://opentelemetry.io/docs/concepts/components/){:target="_blank"}:

1. **Specification**: Describes the cross-language requirements for APIs, SDKs, and data formats like OTLP
2. **Collector**: A vendor-agnostic proxy that receives, processes, and exports telemetry data
3. **Language-specific API & SDK implementations**: These provide the tools to instrument code in your chosen language and include:
  - **Instrumentation Libraries**: Pre-built integrations for popular frameworks and libraries
  - **Exporters**: Send data to various backends (like Jaeger, Prometheus, or the Collector)
  - **Zero-Code Instrumentation**: Allows instrumentation without modifying source code (language-dependent)
  - **Resource Detectors**: Automatically detect attributes of the environment producing telemetry
  - **Cross-Service Propagators**: Handle the propagation of context across service boundaries
  - **Samplers**: Control the volume of traces generated
4. **Kubernetes operator**: Manages the Collector and auto-instrumentation within Kubernetes
5. **Function as a Service assets**: Provides tools for monitoring FaaS environments (like AWS Lambda)

Throughout this challenge, we'll primarily interact with the API, SDK, Instrumentation Libraries, Exporters, and the Collector.

## Setting Up Your Development Environment

To prepare for the rest of this challenge, let's set up a basic development environment. We'll be using Go for our examples, but the concepts apply to any language supported by OpenTelemetry.

### Prerequisites

- Go 1.22 or later
- Git
- Docker and Docker Compose (for running the OpenTelemetry Collector and visualization tools)

### Creating a Project Directory

Let's create a directory for our challenge:

```bash
mkdir otel-challenge
cd otel-challenge
go mod init github.com/yourusername/otel-challenge
```

This will be our base directory for the next 7 days. We'll add more code and configuration as we progress through the challenge.

## What's Coming Next?

Here's what you can look forward to in the coming days:

- **Day 2**: Understanding Distributed Tracing
- **Day 3**: Setting Up the OpenTelemetry Collector
- **Day 4**: Your First OpenTelemetry Instrumentation
- **Day 5**: Automatic Instrumentation and Framework Integration
- **Day 6**: Context Propagation and Logs Correlation
- **Day 7**: Visualization and Analysis

By the end of this challenge, you'll have a solid understanding of OpenTelemetry and be able to implement effective observability in your applications.

## Conclusion

Observability is no longer a nice-to-have but a necessity in modern software development. OpenTelemetry provides a standardized, vendor-neutral approach to implementing observability across your entire stack.

In the next installment, we'll dive deeper into distributed tracing concepts, which form the foundation of OpenTelemetry's approach to observability.

Stay tuned, and happy observing!
