---
layout: post
title: "7 Days of OpenTelemetry: Day 5 - Automatic Instrumentation and Framework Integration"
date: 2025-05-30
author: "Vitor Vasconcellos"
lang: en
img: ":2025-05-30-7-days-otel-day-5-auto-instrumentation-and-framework-integration.png"
categories: [OpenTelemetry, Observability, 7 Days of OpenTelemetry]
tags: [opentelemetry, observability, 7-days-of-opentelemetry]
---

## Day 5: Automatic Instrumentation and Framework Integration

Welcome to Day 5 of our "7 Days of OpenTelemetry" challenge! Yesterday, we implemented manual instrumentation in a Go application. Today, we'll explore automatic instrumentation, which can significantly reduce the amount of code you need to write while still providing comprehensive tracing.

### What is Automatic Instrumentation?

Automatic instrumentation uses pre-built integrations to add tracing to common frameworks and libraries without requiring you to modify your application code directly. This approach has several benefits:

1. **Reduced Development Effort**: Less code to write and maintain
2. **Consistent Coverage**: Standard operations are traced consistently
3. **Best Practices**: Implementations follow OpenTelemetry's recommended patterns
4. **Reduced Risk**: Less chance of introducing bugs in your instrumentation code

While manual instrumentation gives you complete control, automatic instrumentation provides a quick way to get started and cover common scenarios.

### Automatic Instrumentation in Go

In Go, automatic instrumentation is implemented through instrumentation packages that wrap standard libraries and popular frameworks. Let's explore how to use these packages to instrument our application.

### Setting Up a More Complex Application

Let's create a more complex application that uses multiple components we can instrument automatically:

1. HTTP server and client
2. Database connection
3. gRPC service

First, let's create the necessary directories:

```bash
mkdir -p otel-demo-auto/cmd/server
mkdir -p otel-demo-auto/internal/database
mkdir -p otel-demo-auto/internal/grpc
cd otel-demo-auto
```

Initialize a Go module:

```bash
go mod init github.com/yourusername/otel-demo-auto
```

### Adding Dependencies

We'll need several packages for our application and its instrumentation:

```bash
# Core OpenTelemetry packages
go get go.opentelemetry.io/otel \
       go.opentelemetry.io/otel/trace \
       go.opentelemetry.io/otel/sdk \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace \
       go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc

# Instrumentation packages
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp \
       go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc \
       go.opentelemetry.io/contrib/instrumentation/database/sql/otelsql

# Other dependencies
go get google.golang.org/grpc \
       github.com/mattn/go-sqlite3
```

### Setting Up the Telemetry

First, let's create a telemetry initialization file similar to yesterday's example. Create `internal/telemetry/telemetry.go`:

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
func InitTracer() func(context.Context) error {
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
			semconv.ServiceNameKey.String("otel-demo-auto-service"),
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

### Implementing the Database Layer

Now, let's create a simple database layer with automatic instrumentation. Create `internal/database/database.go`:

```go
package database

import (
	"context"
	"database/sql"
	"log"
	"os"

	_ "github.com/mattn/go-sqlite3"
	"go.opentelemetry.io/contrib/instrumentation/database/sql/otelsql"
)

// UserRepository handles database operations for users
type UserRepository struct {
	db *sql.DB
}

// NewUserRepository creates a new UserRepository
func NewUserRepository() (*UserRepository, error) {
	// Remove any existing database file
	os.Remove("./users.db")

	// Create a new database connection with OpenTelemetry instrumentation
	db, err := otelsql.Open("sqlite3", "./users.db",
		otelsql.WithAttributes(
			// Add attributes to identify this database connection
			// in your telemetry data
			map[string]string{
				"db.system":  "sqlite",
				"db.name":    "users",
				"db.user":    "demo",
				"db.instance": "local",
			},
		),
	)
	if err != nil {
		return nil, err
	}

	// Initialize the database
	if err := initDB(db); err != nil {
		db.Close()
		return nil, err
	}

	return &UserRepository{db: db}, nil
}

// Close closes the database connection
func (r *UserRepository) Close() error {
	return r.db.Close()
}

// initDB creates the necessary tables
func initDB(db *sql.DB) error {
	_, err := db.Exec(`
		CREATE TABLE IF NOT EXISTS users (
			id INTEGER PRIMARY KEY AUTOINCREMENT,
			name TEXT NOT NULL,
			email TEXT UNIQUE
		)
	`)
	if err != nil {
		return err
	}

	// Insert some sample data
	_, err = db.Exec(`
		INSERT INTO users (name, email) VALUES 
		('Alice', 'alice@example.com'),
		('Bob', 'bob@example.com'),
		('Charlie', 'charlie@example.com')
	`)
	return err
}

// GetUserByName retrieves a user by name
func (r *UserRepository) GetUserByName(ctx context.Context, name string) (int64, string, error) {
	var id int64
	var email string

	// The query will be automatically traced by the otelsql instrumentation
	err := r.db.QueryRowContext(ctx, "SELECT id, email FROM users WHERE name = ?", name).Scan(&id, &email)
	if err != nil {
		if err == sql.ErrNoRows {
			// Insert a new user if not found
			result, err := r.db.ExecContext(ctx, "INSERT INTO users (name, email) VALUES (?, ?)", 
				name, name+"@example.com")
			if err != nil {
				return 0, "", err
			}
			id, err = result.LastInsertId()
			if err != nil {
				return 0, "", err
			}
			return id, name+"@example.com", nil
		}
		return 0, "", err
	}

	return id, email, nil
}

// GetAllUsers retrieves all users
func (r *UserRepository) GetAllUsers(ctx context.Context) ([]map[string]interface{}, error) {
	// The query will be automatically traced by the otelsql instrumentation
	rows, err := r.db.QueryContext(ctx, "SELECT id, name, email FROM users")
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var users []map[string]interface{}
	for rows.Next() {
		var id int64
		var name, email string
		if err := rows.Scan(&id, &name, &email); err != nil {
			return nil, err
		}
		users = append(users, map[string]interface{}{
			"id":    id,
			"name":  name,
			"email": email,
		})
	}

	if err := rows.Err(); err != nil {
		return nil, err
	}

	return users, nil
}
```

This code:
1. Uses `otelsql` to automatically instrument SQL operations
2. Creates a simple user repository with CRUD operations
3. Adds custom attributes to identify the database in telemetry data

### Implementing the gRPC Service

Now, let's create a simple gRPC service with automatic instrumentation. First, create `internal/grpc/service.proto`:

```protobuf
syntax = "proto3";

package userservice;
option go_package = "github.com/yourusername/otel-demo-auto/internal/grpc";

service UserService {
  rpc GetUserDetails (UserRequest) returns (UserResponse);
}

message UserRequest {
  string name = 1;
}

message UserResponse {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

You'll need to install the Protocol Buffers compiler and the Go plugin to generate the gRPC code. Once installed, run:

```bash
protoc --go_out=. --go-grpc_out=. internal/grpc/service.proto
```

Now, let's implement the gRPC service in `internal/grpc/server.go`:

```go
package grpc

import (
	"context"
	"log"
	"net"

	"github.com/yourusername/otel-demo-auto/internal/database"
	"go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
	"google.golang.org/grpc"
)

// Server implements the gRPC server
type Server struct {
	UnimplementedUserServiceServer
	userRepo *database.UserRepository
	server   *grpc.Server
}

// NewServer creates a new gRPC server
func NewServer(userRepo *database.UserRepository) *Server {
	// Create a new gRPC server with OpenTelemetry interceptors
	grpcServer := grpc.NewServer(
		grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
		grpc.StreamInterceptor(otelgrpc.StreamServerInterceptor()),
	)

	server := &Server{
		userRepo: userRepo,
		server:   grpcServer,
	}

	// Register the service
	RegisterUserServiceServer(grpcServer, server)

	return server
}

// Start starts the gRPC server
func (s *Server) Start() error {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		return err
	}

	log.Println("gRPC server listening on :50051")
	return s.server.Serve(lis)
}

// Stop stops the gRPC server
func (s *Server) Stop() {
	s.server.GracefulStop()
}

// GetUserDetails implements the GetUserDetails RPC method
func (s *Server) GetUserDetails(ctx context.Context, req *UserRequest) (*UserResponse, error) {
	// Get user details from the database
	id, email, err := s.userRepo.GetUserByName(ctx, req.Name)
	if err != nil {
		return nil, err
	}

	return &UserResponse{
		Id:    id,
		Name:  req.Name,
		Email: email,
	}, nil
}
```

This code:
1. Uses `otelgrpc` interceptors to automatically instrument gRPC operations
2. Creates a simple user service that interacts with our database
3. Implements the gRPC service interface

### Implementing the HTTP Server

Finally, let's create an HTTP server that uses both the database and gRPC service. Create `cmd/server/main.go`:

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

	"github.com/yourusername/otel-demo-auto/internal/database"
	grpcservice "github.com/yourusername/otel-demo-auto/internal/grpc"
	"github.com/yourusername/otel-demo-auto/internal/telemetry"
	"go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	// Initialize OpenTelemetry
	shutdown := telemetry.InitTracer()
	defer func() {
		if err := shutdown(context.Background()); err != nil {
			log.Fatalf("Error shutting down tracer: %v", err)
		}
	}()

	// Initialize the database
	userRepo, err := database.NewUserRepository()
	if err != nil {
		log.Fatalf("Failed to initialize database: %v", err)
	}
	defer userRepo.Close()

	// Start the gRPC server in a goroutine
	grpcServer := grpcservice.NewServer(userRepo)
	go func() {
		if err := grpcServer.Start(); err != nil {
			log.Fatalf("Failed to start gRPC server: %v", err)
		}
	}()
	defer grpcServer.Stop()

	// Create a gRPC client with OpenTelemetry instrumentation
	grpcConn, err := grpc.Dial(
		"localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
		grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()),
		grpc.WithStreamInterceptor(otelgrpc.StreamClientInterceptor()),
	)
	if err != nil {
		log.Fatalf("Failed to connect to gRPC server: %v", err)
	}
	defer grpcConn.Close()
	grpcClient := grpcservice.NewUserServiceClient(grpcConn)

	// Set up HTTP handlers
	// Wrap each handler with otelhttp for automatic instrumentation
	http.Handle("/", otelhttp.NewHandler(http.HandlerFunc(handleRoot), "handleRoot"))
	http.Handle("/users", otelhttp.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		handleUsers(w, r, userRepo)
	}), "handleUsers"))
	http.Handle("/user", otelhttp.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		handleUser(w, r, grpcClient)
	}), "handleUser"))

	// Start the HTTP server in a goroutine
	server := &http.Server{Addr: ":8080"}
	go func() {
		fmt.Println("HTTP server starting on :8080...")
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("HTTP server failed to start: %v", err)
		}
	}()

	// Wait for interrupt signal
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, os.Interrupt)
	<-sigCh

	fmt.Println("Shutting down...")
	
	// Gracefully shut down the HTTP server
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	if err := server.Shutdown(ctx); err != nil {
		log.Fatalf("HTTP server shutdown failed: %v", err)
	}
}

func handleRoot(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Welcome to the OpenTelemetry Auto-Instrumentation Demo!\n")
	fmt.Fprintf(w, "Available endpoints:\n")
	fmt.Fprintf(w, "- /users: List all users\n")
	fmt.Fprintf(w, "- /user?name=<n>: Get user details\n")
}

func handleUsers(w http.ResponseWriter, r *http.Request, userRepo *database.UserRepository) {
	users, err := userRepo.GetAllUsers(r.Context())
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"users": users,
	})
}

func handleUser(w http.ResponseWriter, r *http.Request, client grpcservice.UserServiceClient) {
	name := r.URL.Query().Get("name")
	if name == "" {
		http.Error(w, "Missing name parameter", http.StatusBadRequest)
		return
	}

	// Call the gRPC service
	resp, err := client.GetUserDetails(r.Context(), &grpcservice.UserRequest{Name: name})
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(map[string]interface{}{
		"id":    resp.Id,
		"name":  resp.Name,
		"email": resp.Email,
	})
}
```

This code:
1. Uses `otelhttp` to automatically instrument HTTP handlers
2. Uses `otelgrpc` to automatically instrument the gRPC client
3. Creates three endpoints:
   - `/`: A welcome page
   - `/users`: Lists all users from the database
   - `/user`: Gets user details via the gRPC service

### Running the Application

Now, let's run our application:

```bash
go run cmd/server/main.go
```

You should see output indicating that both the HTTP and gRPC servers have started. In a separate terminal, make some requests to the server:

```bash
curl http://localhost:8080/
curl http://localhost:8080/users
curl http://localhost:8080/user?name=Alice
curl http://localhost:8080/user?name=Dave
```

### Viewing Traces in the Collector

Now, let's check the Collector logs to see the traces:

```bash
docker-compose -f otel-collector/docker-compose.yaml logs
```

You should see detailed trace information in the logs, including:
- HTTP requests and responses
- gRPC calls
- Database queries
- The relationships between these operations

The automatic instrumentation has created a comprehensive trace of each request's journey through our system, without requiring us to add explicit tracing code to our application logic.

### Understanding Automatic Instrumentation

Let's break down the key components of our automatic instrumentation:

#### HTTP Instrumentation

```go
http.Handle("/users", otelhttp.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    handleUsers(w, r, userRepo)
}), "handleUsers"))
```

The `otelhttp` package provides a handler wrapper that:
1. Creates a span for each incoming request
2. Adds HTTP-specific attributes (method, URL, status code, etc.)
3. Propagates context to the handler function
4. Ends the span when the request is complete

#### gRPC Instrumentation

```go
// Server-side
grpcServer := grpc.NewServer(
    grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
    grpc.StreamInterceptor(otelgrpc.StreamServerInterceptor()),
)

// Client-side
grpcConn, err := grpc.Dial(
    "localhost:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()),
    grpc.WithStreamInterceptor(otelgrpc.StreamClientInterceptor()),
)
```

The `otelgrpc` package provides interceptors for both client and server that:
1. Create spans for each RPC call
2. Add gRPC-specific attributes (method, status, etc.)
3. Propagate context between client and server
4. End spans when the RPC call is complete

#### Database Instrumentation

```go
db, err := otelsql.Open("sqlite3", "./users.db",
    otelsql.WithAttributes(
        map[string]string{
            "db.system":  "sqlite",
            "db.name":    "users",
            "db.user":    "demo",
            "db.instance": "local",
        },
    ),
)
```

The `otelsql` package provides a wrapper around the standard `database/sql` package that:
1. Creates spans for database operations (queries, executions, etc.)
2. Adds database-specific attributes (query, operation type, etc.)
3. Propagates context to the database driver
4. Ends spans when the database operation is complete

### Combining Manual and Automatic Instrumentation

While automatic instrumentation is powerful, you may still want to add manual instrumentation for specific business logic. Let's modify our `handleUser` function to include some manual instrumentation:

```go
func handleUser(w http.ResponseWriter, r *http.Request, client grpcservice.UserServiceClient) {
    // Get the current context, which already has the span from otelhttp
    ctx := r.Context()
    
    // Get a tracer
    tracer := otel.Tracer("github.com/yourusername/otel-demo-auto/cmd/server")
    
    // Create a child span for parameter validation
    ctx, validateSpan := tracer.Start(ctx, "validateUserParams")
    
    name := r.URL.Query().Get("name")
    if name == "" {
        validateSpan.SetStatus(codes.Error, "Missing name parameter")
        validateSpan.End()
        http.Error(w, "Missing name parameter", http.StatusBadRequest)
        return
    }
    
    // Add an attribute to the span
    validateSpan.SetAttributes(attribute.String("user.name", name))
    
    // End the validation span
    validateSpan.End()
    
    // Create a span for the gRPC call (this will be a parent of the automatic gRPC client span)
    ctx, grpcSpan := tracer.Start(ctx, "callUserService")
    
    // Call the gRPC service (this will create its own span as a child of grpcSpan)
    resp, err := client.GetUserDetails(ctx, &grpcservice.UserRequest{Name: name})
    if err != nil {
        grpcSpan.SetStatus(codes.Error, err.Error())
        grpcSpan.End()
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Add the result to the span
    grpcSpan.SetAttributes(attribute.Int64("user.id", resp.Id))
    
    // End the gRPC span
    grpcSpan.End()
    
    // Create a span for response formatting
    ctx, formatSpan := tracer.Start(ctx, "formatResponse")
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "id":    resp.Id,
        "name":  resp.Name,
        "email": resp.Email,
    })
    
    // End the format span
    formatSpan.End()
}
```

This combined approach gives you the best of both worlds:
1. Automatic instrumentation for standard operations
2. Manual instrumentation for business-specific logic
3. A complete trace that shows the entire request flow

### Cross-Language Implementation

While our examples use Go, OpenTelemetry provides automatic instrumentation for many languages. Here's a quick comparison of how automatic instrumentation works in other languages:

#### Java

In Java, automatic instrumentation can be added using the Java agent:

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=my-service \
     -Dotel.traces.exporter=otlp \
     -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
     -jar my-application.jar
```

This automatically instruments many popular Java frameworks and libraries, including:
- Spring
- JDBC
- Hibernate
- Apache HTTP Client
- Netty
- gRPC

#### Python

In Python, automatic instrumentation can be added using the `opentelemetry-instrument` command:

```bash
opentelemetry-instrument \
    --service_name my-service \
    --traces_exporter otlp \
    --exporter_otlp_endpoint http://localhost:4317 \
    python my_application.py
```

This automatically instruments many popular Python frameworks and libraries, including:
- Flask
- Django
- SQLAlchemy
- Requests
- aiohttp
- gRPC

#### Node.js

In Node.js, automatic instrumentation can be added using the `@opentelemetry/auto-instrumentations-node` package:

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-proto');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://localhost:4317',
  }),
  instrumentations: [getNodeAutoInstrumentations()]
});

sdk.start();
```

This automatically instruments many popular Node.js frameworks and libraries, including:
- Express
- Koa
- Fastify
- MongoDB
- MySQL
- Redis
- gRPC

### Best Practices for Automatic Instrumentation

Based on our exploration, here are some best practices for using automatic instrumentation:

1. **Start with Automatic, Add Manual as Needed**: Begin with automatic instrumentation for standard components, then add manual instrumentation for business-specific logic.

2. **Use Consistent Naming**: Use consistent naming conventions for spans and attributes across both automatic and manual instrumentation.

3. **Add Custom Attributes**: Enhance automatic spans with custom attributes that provide business context.

4. **Monitor Performance Impact**: While automatic instrumentation is designed to be lightweight, monitor its performance impact in your application.

5. **Keep Dependencies Updated**: Regularly update your OpenTelemetry dependencies to benefit from improvements and bug fixes.

6. **Understand What's Being Traced**: Familiarize yourself with what each automatic instrumentation package traces to avoid gaps in coverage.

7. **Configure Sampling Appropriately**: Use sampling to control the volume of traces generated, especially in high-traffic applications.

### Conclusion

In this installment, we've explored automatic instrumentation in OpenTelemetry. We've seen how to:

1. Use pre-built integrations to instrument HTTP, gRPC, and database operations
2. Combine automatic and manual instrumentation for comprehensive tracing
3. Understand how automatic instrumentation works across different languages

Automatic instrumentation provides a quick and easy way to add tracing to your applications, especially when using common frameworks and libraries. By combining it with manual instrumentation, you can create a complete picture of your application's behavior.

In tomorrow's installment, we'll explore context propagation and logs correlation, which are essential for creating a complete observability picture in distributed systems.

Stay tuned, and happy tracing!
