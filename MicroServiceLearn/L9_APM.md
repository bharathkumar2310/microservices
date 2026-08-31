1. Trace

        A trace represents one complete request journey through your system.

Suppose you call:

        GET /orders/1

That single request gets a Trace.

If it goes through multiple components:

        Client
        ↓
        Order Service
        ↓
        Payment Service
        ↓
        Database

the entire journey belongs to one trace.

Think:

    Trace = the complete story of one request.

2. Span

        A span represents one operation within that trace.

For example:

        Trace: GET /orders/1
        │
        ├── Span 1: Order Controller       50 ms
        │
        ├── Span 2: Order Service          20 ms
        │
        ├── Span 3: Payment Service        500 ms
        │
        └── Span 4: Database Query        100 ms

So:

Trace
└── multiple Spans

A span normally contains information such as:

    operation name
    start time
    duration
    service
    status
    attributes

3. Trace ID

        The Trace ID uniquely identifies the entire request journey.

Example:

    Trace ID = abc123

Every span belonging to that request is associated with that trace:

Trace abc123


    Order Service
         ↓
    Payment Service
         ↓
       MySQL

This is extremely useful when you have thousands of requests.

You can say:

"Show me everything that happened for Trace ID abc123."

4. The complete picture
    
       TRACE
       ID: abc123
       │
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
       Span           Span           Span
       Order Service  Payment Service   MySQL
       50ms           500ms          100ms

Now imagine the API takes 5 seconds:

    GET /orders/1
    │
    └── Trace abc123
    │
    ├── Order Service     50 ms
    │
    ├── Payment Service   100 ms
    │
    └── Inventory Service 4.8 sec 🔴

Immediately we can see:

Inventory Service is the bottleneck.

That's what we couldn't determine from basic http.server.requests alone.




    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-opentelemetry</artifactId>
    </dependency>



1. Install OpenTelemetry Collector

Open PowerShell as Administrator and run:

msiexec /i "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.157.0/otelcol_0.157.0_windows_x64.msi"

This installs the Collector as a Windows service. The official documentation currently lists v0.157.0 for this installation command.

You can also download the installer from the official OpenTelemetry Collector releases page.

2. Check that it was installed

Open Windows Services:

Win + R

then:

services.msc

Look for:

OpenTelemetry Collector

The MSI registers it as a Windows service.

You should be able to:

OpenTelemetry Collector
Status: Running
Startup Type: Automatic



      management.opentelemetry.tracing.export.otlp.endpoint=http://localhost:4318/v1/traces


Option 2 — Easier: run the Collector manually

For our learning/debugging, I recommend this.

Don't uninstall the service. Just stop it temporarily:

      Stop-Service otelcol

Then open Administrator PowerShell and run:

      cd "C:\Program Files\OpenTelemetry Collector"

Then:

      .\otelcol.exe --config="config.yaml"

Now the Collector runs directly in your PowerShell window.

You should see startup logs.

Then, while that window remains open, call:

GET http://localhost:8080/orders/1

If Spring Boot is successfully sending traces, the Collector's console should show something containing:

Traces
Trace ID
Span ID
Important

Keep this PowerShell window open.



Step 1 — Extract Jaeger

You currently have:

jaeger-2.20.0-windows-amd64.tar.gz

Right-click it → WinRAR → Extract Here.

You should end up with a folder containing:

jaeger.exe

I'd recommend moving the extracted folder to:

C:\Jaeger

So you have:

C:\Jaeger\jaeger.exe


add this jaegar.yaml file

      service:
      telemetry:
      metrics:
      readers:
      - pull:
      exporter:
      prometheus:
      host: 127.0.0.1
      port: 8889
      
      extensions:
      - jaeger_storage
        - jaeger_query
      
      pipelines:
      traces:
      receivers:
      - otlp
      processors: []
      exporters:
        - jaeger_storage_exporter
      
      extensions:
      jaeger_storage:
      backends:
      memory:
      memory:
      max_traces: 100000
      
      jaeger_query:
      storage:
      traces: memory
      base_path: /
      grpc:
      endpoint: 0.0.0.0:16685
      http:
      endpoint: 0.0.0.0:16686
      
      receivers:
      otlp:
      protocols:
      grpc:
      endpoint: 0.0.0.0:15317
      http:
      endpoint: 0.0.0.0:15318
      
      exporters:
      jaeger_storage_exporter:
      trace_storage: memory


Step 2 — Open PowerShell

Open PowerShell and run:

cd C:\Jaeger

Then:

      jaeger.exe --config=jaeger.yaml

Keep this PowerShell window open.

You should see Jaeger startup messages.

Step 3 — Open Jaeger UI

In your browser, open:

http://localhost:16686/search

You should see the Jaeger interface.

If it opens, Jaeger is successfully running. ✅



---------------------------------------------------------------------------------------------------------------------------------------

For other technologies

Some integrations need additional instrumentation.

| What you want to trace        | Instrumentation                 |
| ----------------------------- | ------------------------------- |
| Incoming HTTP/Spring MVC      | Already covered by your starter |
| JDBC/MySQL                    | JDBC instrumentation            |
| REST calls to another service | HTTP client instrumentation     |
| Kafka                         | Kafka instrumentation           |
| Redis                         | Redis instrumentation           |
| Messaging systems             | Corresponding instrumentation   |

need to add dependencies to show db spans, http spans ect


add
   
      <dependency>
      <groupId>net.ttddyy.observation</groupId>
      <artifactId>datasource-micrometer-spring-boot</artifactId>
      </dependency>

for db cal tracing