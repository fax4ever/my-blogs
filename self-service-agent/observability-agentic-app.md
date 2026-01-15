<!-----

You have some errors, warnings, or alerts. If you are using reckless mode, turn it off to see useful information and inline alerts.
* ERRORs: 0
* WARNINGs: 0
* ALERTS: 5

Conversion time: 2.284 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs™ to Markdown version 2.0β1
* Thu Jan 15 2026 00:25:47 GMT-0800 (PST)
* Source doc: Debugging Autonomy: Advanced Observability for a Self-Service IT Agent
* Tables are currently converted to HTML tables.
* This document has images: check for >>>>>  gd2md-html alert:  inline image link in generated source and store images to your server. NOTE: Images in exported zip file from Google Docs may not appear in  the same order as they do in your doc. Please check the images!

----->


<p style="color: red; font-weight: bold">>>>>>  gd2md-html alert:  ERRORs: 0; WARNINGs: 0; ALERTS: 5.</p>
<ul style="color: red; font-weight: bold"><li>See top comment block for details on ERRORs and WARNINGs. <li>In the converted Markdown or HTML, search for inline alerts that start with >>>>>  gd2md-html alert:  for specific instances that need correction.</ul>

<p style="color: red; font-weight: bold">Links to alert messages:</p><a href="#gdcalert1">alert1</a>
<a href="#gdcalert2">alert2</a>
<a href="#gdcalert3">alert3</a>
<a href="#gdcalert4">alert4</a>
<a href="#gdcalert5">alert5</a>

<p style="color: red; font-weight: bold">>>>>> PLEASE check and correct alert issues and delete this message and the inline alerts.<hr></p>



# Debugging Autonomy: Advanced Observability for a Self-Service IT Agent

Agentic applications typically involve complex interactions between multiple components—routing agents, specialist agents, knowledge bases, MCP servers, and external systems—making production debugging challenging without proper visibility.

In this blog post, we will describe in detail a practical approach to setting up distributed tracing for an agentic workflow, based on lessons learned while developing the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) quickstart.

By the end of this journey, you will understand how to configure OpenTelemetry support for distributed tracing across all typical agentic application components, enabling you to track requests end-to-end through application workloads, MCP Servers, and Llama Stack.

AI quickstarts are a catalog of ready-to-run industry-specific use cases for your Red Hat AI environment. Each quickstart is designed to be simple to deploy, explore, and extend. They give teams a fast, hands-on way to see how AI can power solutions on enterprise-ready, open source infrastructure. You can read more about quickstarts in "AI Quickstarts: an easy and practical way to get started with Red Hat AI" and the it-self-service-agent in particular in  <span style="text-decoration:underline;">An introduction to the Self Service Agent - laptop refresh quickstart</span>.

This is the sixth post in a series covering what we learned while developing the it-self-service-agent quickstart, which will be as follows:



* An introduction to the Self Service Agent - laptop refresh quickstart
* AI Meets You Where You Are: Seamless Integration with Slack, Email, and ServiceNow
* Big Prompt vs. Small Prompt: Architecting Your Agent for Speed, Accuracy, and Efficiency 
* Unlocking Agents: Leveraging the Responses API within the Llama Stack Ecosystem
* Eval-Driven Development: Building Reliable Agentic Applications
* **Debugging Autonomy: Advanced Observability for a Self-Service IT Agent **- this blog post
* Elevating the Enterprise: Our Model Context Protocol (MCP)-Powered ServiceNow Integration Experience

If you are interested in a bit more detail on the business benefits of using agentic AI to automate IT processes, check out <span style="text-decoration:underline;">AI quickstart: Implementing IT processes with agentic AI on Red Hat OpenShift AI</span>.


## Agentic Workloads Distributed Tracing



<p id="gdcalert1" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image1.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert2">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image1.png "image_tooltip")


In the first part of this article, we will explain how to produce tracing spans from application workloads, assuming an OpenTelemetry collector is available at a given remote address. In the second part, we will explore ways to configure the tracing infrastructure backing the collector, particularly using [OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift/container-platform).

In this post  tracing is configured following the OpenTelemetry version 1.37.0, but of course different versions may be used. For more information on the standard, see the [OpenTelemetry page.](https://opentelemetry.io/) In this post examples are in Python, however OpenTelemetry libraries are available for most any popular programming languages.


## OpenTelemetry

OpenTelemetry is an open-source observability framework that provides a unified set of APIs, libraries, and instrumentation to capture distributed traces, metrics, and logs from applications. It emerged from the merger of OpenTracing and OpenCensus projects in 2019 and has since become the de facto standard for application observability across the industry.

The key advantages of OpenTelemetry include:



1. **Vendor neutrality**: Instrument your code once and export telemetry data to any compatible backend (Jaeger, Tempo, Zipkin, commercial solutions, etc.)
2. **Broad language support**: Official SDKs available for all major programming languages including Python, Java, Go, JavaScript, and many others
3. **Automatic instrumentation**: Many popular frameworks and libraries offer auto-instrumentation, reducing the amount of custom code needed
4. **Context propagation**: Built-in mechanisms to maintain trace context across service boundaries, essential for distributed systems
5. **Active ecosystem**: Strong community support, regular updates, and backing from major cloud providers and observability vendors

For agentic applications with their complex, multi-component architectures, involving routing agents, specialist agents, LLM inference, MCP servers, and external integrations, distributed tracing becomes essential. OpenTelemetry allows us to trace requests end-to-end across these heterogeneous components, correlating spans to understand the complete execution flow and identify performance bottlenecks.

In the sections that follow, we'll demonstrate how to instrument various components of an agentic application using OpenTelemetry, both through automatic instrumentation where available and manual instrumentation where necessary.


## Context Propagation

The [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) quickstart demonstrates a realistic production architecture for agentic applications, consisting of multiple loosely-coupled services communicating via events. Understanding how requests flow through this system is essential for effective observability.


### Architecture Components

The quickstart deploys the following main components:



* **Request Manager**: The central entry point that normalizes incoming requests from multiple channels (Slack, email, CLI, webhooks, web UI) into a standardized format.
* **Agent Service**: The AI orchestration engine that routes requests between a routing agent and specialist agents.
* **Integration Dispatcher**: Handles bidirectional communication with external channels, routing agent responses back to users via their original communication channel.
* **Llama Stack**: Provides LLM inference endpoints for agent reasoning, manages tool calling and function execution, and supports vector database operations for knowledge base retrieval.
* **MCP Servers**: Model Context Protocol servers that provide tools for integrating with external systems.
* **Eventing Layer**: Routes CloudEvents between services. 


### The Challenge of Distributed Tracing

A single user request in this architecture typically traverses multiple service boundaries asynchronously:



1. User sends message via Slack
2. Integration Dispatcher receives webhook and forwards to Request Manager
3. Request Manager creates/updates session and publishes CloudEvent
4. Eventing layer routes event to Agent Service
5. Agent Service processes request, possibly invoking:
    1. Llama Stack for LLM inference (multiple times)
    2. Knowledge base queries for domain information
    3. MCP server tool calls to external systems
6. Agent Service publishes response CloudEvent
7. Eventing layer routes to Integration Dispatcher
8. Integration Dispatcher sends formatted response back to Slack

Without proper context propagation, each service would create isolated, disconnected spans. We would see individual operations in our tracing backend but lose **the causal relationships** that show which operations triggered which downstream calls.


### How Context Propagation Works

OpenTelemetry solves this through *trace context propagation* using the [W3C Trace Context](https://www.w3.org/TR/trace-context/) standard. Each trace is assigned a unique trace ID that remains constant across all services involved in processing that request. As calls cross service boundaries, the trace context (containing the trace ID and parent span ID in the format version-traceid-spanid-sampled ) is transmitted along with the request, allowing downstream services to continue the trace rather than starting a new, disconnected one.

This is an example of a trace and span ID injected as an additional HTTP header:


```
traceparent: 00-1e9a84a8c5ae45c30b1305a0f41ed275-215435bcec6efa72-00
```


The propagation mechanism works through a simple extract-and-inject pattern:

**Extract**: When a service receives an incoming request, it extracts the trace context from the request metadata.

**Inject**: When that same service makes an outbound call to another service, it injects the current trace context into the outbound request metadata, allowing the receiving service to continue the trace.

This propagation pattern ensures that when we view a trace in Jaeger or the OpenShift console, we see the complete end-to-end request flow with proper parent-child relationships between spans, even across asynchronous service boundaries. 

As we'll see in the examples later in this post, a typical trace shows all components working together—from the initial HTTP request through multiple agent interactions, LLM inference calls, knowledge base queries, and MCP tool invocations—all correlated under a single trace ID.


## Instrumenting Llama Stack

Llama Stack includes native OpenTelemetry support, so enabling tracing requires only configuration—no custom instrumentation code. Once enabled, Llama Stack automatically produces spans for LLM inference requests, tool invocations, and vector database operations, all with relevant attributes like model name, token counts, and tool parameters.

For instance default v. 0.2 Llama Stack configuration has a telemetry component defined like:


```
telemetry:
  - provider_id: meta-reference
    provider_type: inline::meta-reference
    config:
      sinks: ${env.TELEMETRY_SINKS:=console,sqlite,otel_trace}
      service_name: {{ include "llama-stack.fullname" . }}
      otel_exporter_otlp_endpoint: ${env.OTEL_EXPORTER_OTLP_ENDPOINT:=}
```


For Llama Stack 0.3.x, configuration is simpler—just set environment variables:


```
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: {{ .Values.otelExporter }}
  - name: TELEMETRY_SINKS
    value: console,sqlite,otel_trace
```



### A wrinkle with MCP servers

According to our research, in at least Llama Stack 0.2.x and 0.3.x, the span context is not automatically propagated from the Llama Stack server to the MCP servers when a call is made through the Responses API.  We, therefore, we needed to manually inject the parent tracing context so they appear in the HTTP headers when Llama Stack make a call to an MCP Server. We do this as follows:

*Adapted from [agent-service/src/agent_service/langgraph/responses_agent.py](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/agent-service/src/agent_service/langgraph/responses_agent.py):*


```
from opentelemetry.propagate import inject

# Prepare headers for MCP server request
tool_headers = {}

inject(tool_headers)
logger.debug(
    f"Injected tracing headers for MCP server {server_name}: {list(tool_headers.keys())}"
)

```


Which results in an additional transparent header in what we pass to the Responses API as the MCP server config as follows:

```python


```
 [
      {
          "type": "mcp",
          "server_label": "snow",
          "server_url": "http://mcp-self-service-agent-snow:8000/mcp",
          "require_approval": "never",
          "headers": {
              "traceparent": "00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01",
              "tracestate": "",
          }
      }
  ]
```


```


## Auto-instrumentation: HTTP Clients and FastAPI

The [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) quickstart consists of a number of different components. For the components we built ourselves we needed to figure out how to add support for OpenTelemetry.

The OpenTelemetry spans we want to capture typically depend on the frameworks used in the components. For instance, in the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) quickstart, much of the communication between workloads is implemented as REST API calls. We want to capture these calls in our tracing, producing a span for each invocation while maintaining the causal relationships between them.

Whenever a library supports auto-instrumentation, we should take advantage of it. For example, we used the [OpenTelemetry instrumentation HTTPx](https://pypi.org/project/opentelemetry-instrumentation-httpx/) and [OpenTelemetry FastAPI Instrumentation](https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/fastapi/fastapi.html) libraries to instrument all REST server and client requests and responses within the quickstart. This approach produces spans for all calls without requiring potentially error-prone custom code. In addition, context propagation is handled by these libraries, eliminating the need to manually pass context from parent calls to child calls.

In our project, we added these instrumentation libraries to all modules that use the frameworks, along with the base libraries for the OpenTelemetry APIs, SDK, and exporter.

In the quickstart components we imported the following dependencies:

*Adapted from [tracing-config/pyproject.toml](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/tracing-config/pyproject.toml):*


```
    "opentelemetry-exporter-otlp-proto-http==1.37.0",
    "opentelemetry-instrumentation-httpx==0.58b0",
    "opentelemetry-api>=1.37.0",
    "opentelemetry-sdk>=1.37.0",
    "opentelemetry-instrumentation-fastapi>=0.58b0"
```


When using auto-instrumentation, we typically define environment variables that may or may not be used by the instrumentation framework (depending on the implementation).

For instance, in our project's autoinstrumented components, we set the following variables:

*Adapted from [helm/templates/_service-deployment.tpl](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/helm/templates/_service-deployment.tpl):*


```
ENV OTEL_SERVICE_NAME=${SERVICE_NAME}
ENV OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true
ENV OTEL_EXPORTER_OTLP_ENDPOINT=${OTEL_EXPORTER_OTLP_ENDPOINT}
```


In particular, we used the presence of `OTEL_EXPORTER_OTLP_ENDPOINT` to programmatically configure the trace and the span and batch processors.

Context must be propagated when calls are made across components in the quickstart. To propagate the context in order to follow the causality in the span correlation we used the following code:

*Adapted from [tracing-config/src/tracing_config/auto_tracing.py](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/tracing-config/src/tracing_config/auto_tracing.py):*


```
from opentelemetry.propagate import set_global_textmap
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

# Set up propagator
set_global_textmap(TraceContextTextMapPropagator())
```


To enable the HTTPX client auto-instrumentation:

*Adapted from [request-manager/src/request_manager/main.py](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/request-manager/src/request_manager/main.py) and similar service main files:*


```
# Set up instrumentations
HTTPXClientInstrumentor().instrument()
```


and finally to enable the FastAPI instrumentation:

*Adapted from [request-manager/src/request_manager/main.py](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/request-manager/src/request_manager/main.py) and similar service main files:*


```
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# app is FastAPI instance
FastAPIInstrumentor.instrument_app(app)
```



## Manual Instrumentation: MCP Servers

At the time of development, for FastMCP, we could not find  an auto-instrumentation library which produced the results we were looking for. Therefore, we opted for manual instrumentation to trace MCP server calls.

Manual instrumentation requires that we ensure that manually created spans integrate with automatically instrumented spans. The key challenge is maintaining proper trace context propagation—ensuring that manually created spans correctly identify their parent spans and that child operations (including automatically instrumented HTTP calls) continue the trace correctly.


### Setting Up Manual Instrumentation

Before creating spans manually, you need to initialize the OpenTelemetry tracer. This setup is typically done once at module initialization:

*Adapted from [mcp-servers/snow/src/snow/tracing.py](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/mcp-servers/snow/src/snow/tracing.py):*


```
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
import os

# Initialize tracer provider if not already initialized
if not trace.get_tracer_provider():
    provider = TracerProvider()
    trace.set_tracer_provider(provider)

# Configure OTLP exporter if endpoint is provided
otel_endpoint = os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
if otel_endpoint:
    otlp_exporter = OTLPSpanExporter(endpoint=otel_endpoint)
    span_processor = BatchSpanProcessor(otlp_exporter)
    trace.get_tracer_provider().add_span_processor(span_processor)

# Get tracer for this service
tracer = trace.get_tracer(__name__, "1.0.0")
```



### Context Extraction and Span Creation

Next we need to ensure that the parent trace context is extracted from incoming requests. This ensures that your manually created spans appear as children of the appropriate parent span in the trace, maintaining the causal relationship between operations.

When an MCP server receives a tool invocation request, the trace context (if present) is transmitted in HTTP headers. As covered earlier, the W3C Trace Context standard specifies that trace context is carried in the `traceparent` header. We extract this from the context and use it as the parent when creating our span:

Here is a code example *adapted from [mcp-servers/snow/src/snow/tracing.py](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/mcp-servers/snow/src/snow/tracing.py):*


```
from opentelemetry.propagate import extract
from opentelemetry.trace import Status, StatusCode

headers = ctx.request_context.headers
# Convert headers to dict-like format expected by propagator
# Normalize header keys to lowercase as traceparent is case-insensitive
carrier = {k.lower(): v for k, v in dict(headers).items()}

# Extract context from headers using the global propagator
parent_context = extract(carrier)

# Create span with parent context
span_name = f"mcp.tool.{func.__name__}"
with tracer.start_as_current_span(span_name, context=parent_context) as span:
    # Set semantic attributes following OpenTelemetry conventions
    span.set_attribute("mcp.tool.name", func.__name__)
    span.set_attribute("mcp.server.name", server_name)
    
    # Record function arguments as attributes for debugging
    for i, arg in enumerate(args):
        span.set_attribute(f"mcp.tool.arg.{i}", str(arg))  

    for key, value in kwargs.items():
        span.set_attribute(f"mcp.tool.param.{key}", str(value))
    
    try:
        # Execute the tool function
        # The span context will automatically propagate to any
        # instrumented HTTP calls made within this function
        result = func(*args, **kwargs) 
        span.set_status(Status(StatusCode.OK))
        return result

    except Exception as e:
        # Record the exception and set error status
        span.record_exception(e)
        span.set_status(Status(StatusCode.ERROR, str(e)))
        raise
```



### Additional Considerations for Manual Instrumentation

**Span Naming Conventions**: Use consistent, hierarchical naming that reflects the operation being traced. For MCP tools, we use the pattern `mcp.tool.{tool_name}` which makes it easy to filter and group related spans in the tracing UI.

**Attribute Best Practices**: 



* Use semantic attribute names following [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/) where applicable
* For custom attributes, use a consistent prefix (e.g., `mcp.`) to avoid conflicts
* Be mindful of attribute value size, very large values can impact trace storage and query performance
* Include enough context to debug issues but avoid sensitive data (passwords, tokens, etc.)

**Error Handling**: Always set span status appropriately:



* `StatusCode.OK` for successful operations
* `StatusCode.ERROR` for failures, with an optional descriptive message
* Use `span.record_exception()` to capture exception details, which automatically adds stack traces and exception metadata to the span

**Context Propagation**: When you create a span with `start_as_current_span()`, it becomes the active span in the current context. This means any automatically instrumented operations (like HTTP calls) that occur within that span's context will automatically become child spans. This seamless integration between manual and automatic instrumentation is one of OpenTelemetry's key strengths.

**Performance Impact**: Manual instrumentation adds minimal overhead, typically microseconds per span. However, be cautious about:



* Creating too many fine-grained spans for trivial operations
* Setting attributes with expensive computations (evaluate lazily if needed)
* Including large data structures as attribute values


### Decorator Pattern for Clean Integration

To make manual instrumentation less intrusive in the quickstart, we wrap MCP tool functions with a decorator that handles all the tracing logic:

*Adapted from [mcp-servers/snow/src/snow/tracing.py](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/mcp-servers/snow/src/snow/tracing.py):*


```
from functools import wraps

def trace_mcp_tool(server_name: str):
    """Decorator to automatically trace MCP tool invocations."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Extract context from request (implementation depends on framework)
            headers = get_request_headers()  # Framework-specific
            carrier = {k.lower(): v for k, v in dict(headers).items()}
            parent_context = extract(carrier)
            
            span_name = f"mcp.tool.{func.__name__}"
            with tracer.start_as_current_span(span_name, context=parent_context) as span:
                span.set_attribute("mcp.tool.name", func.__name__)
                span.set_attribute("mcp.server.name", server_name)
                # ... rest of instrumentation ...
                return func(*args, **kwargs)
        return wrapper
    return decorator

# Usage:
@trace_mcp_tool(server_name="snow")
def get_employee_laptop_info(employee_id: str):
    # Tool implementation
    pass
```


This pattern keeps the tracing logic separate from business logic, making the codebase easier to maintain and ensuring consistent instrumentation across all MCP tools.


## Llama Stack tracing configuration

Note that our solution has been tested on Llama Stack 0.2.x and 0.3.x series.

To configure Llama Stack with more recent versions or for additional information, please see the [Llama Stack telemetry documentation](https://llamastack.github.io/docs/providers/telemetry/inline_meta-reference).

In the Llama Stack configuration, we define the telemetry to include the OTEL exporter and the `otel_trace` sink if an `otelExporter` is defined at the root application level.

For Llama Stack 0.2.x series, we can define the telemetry as follows:

*Adapted from Helm values passed to the `llama-stack` subchart (configured in [helm/values.yaml](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/helm/values.yaml)):*


```
telemetry:
      - provider_id: meta-reference
        provider_type: inline::meta-reference
        config:
          sqlite_db_path: ${env.SQLITE_DB_PATH:=~/.llama/distributions/starter/trace_store.db}
          {{- if .Values.otelExporter }}
          sinks: ${env.TELEMETRY_SINKS:=console,sqlite,otel_trace}
          service_name: {{ include "llama-stack.fullname" . }}
          otel_exporter_otlp_endpoint: ${env.OTEL_EXPORTER_OTLP_ENDPOINT:=}
          {{- else }}
          sinks: ${env.TELEMETRY_SINKS:=console,sqlite}
          {{- end }}
```


For Llama Stack 0.3.x series, we only need to set the environment variables:

*Adapted from Helm values passed to the `llama-stack` subchart (configured in [helm/values.yaml](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/main/helm/values.yaml)*):


```
env:
  {{- if .Values.otelExporter }}
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: {{ .Values.otelExporter }}
  {{- end }}
  {{- if .Values.telemetrySinks }}
  - name: TELEMETRY_SINKS
    value: {{ .Values.telemetrySinks }}
  {{- end }}
```



## Collect the tracing with an all-in-one Jaeger server

One way to visualize the spans we've produced in the previous sections is to deploy an all-in-one Jaeger server instance you can do with through podman with something like:

In the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) we have also included a make target called [jaeger-deploy](https://github.com/rh-ai-quickstart/it-self-service-agent/blob/d7d1f35f94030514ddc08906c03cdecffe2deeea/Makefile#L1398) that can be used to deploy an instance of Jaeger in the same namespace as the quickstart. For more details on a simple Jaeger deployment please refer to the [Jaeger Getting Started guide](https://www.jaegertracing.io/docs/2.14/getting-started/).

Below are some example traces produced by the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent).



<p id="gdcalert2" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image2.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert3">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image2.png "image_tooltip")


Here's the same data shown in a graph view:



<p id="gdcalert3" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image3.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert4">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image3.png "image_tooltip")


As you can see, the trace structure can be quite complex. A typical request includes several nested calls, as shown in this example:


```
http.request POST /api/v1/requests (request-manager)          [120ms]
  └─ publish_event agent.request (request-manager)            [10ms]
      └─ http.request POST /agent/chat (agent-service)        [95ms]
          ├─ knowledge_base_query laptop-refresh-policy       [15ms]
          ├─ http.request POST /inference/chat (llamastack)   [65ms]
          │   └─ mcp.tool.get_employee_laptop_info            [8ms]
          │       └─ http.request GET servicenow.com/api      [6ms]
          └─ http.request POST /inference/chat (llamastack)   [12ms]
              └─ mcp.tool.open_laptop_refresh_ticket          [8ms]
                  └─ http.request POST servicenow.com/api     [6ms]
```


In this example, the overall request time is dominated by the LLM inference call, which is typical for agentic applications.


## Collect the tracing with the OpenShift Distributed Tracing Platform

The solution we described in the previous section is more suitable for a test or development environment.

For production, we recommend collecting traces using the [OpenShift Distributed Tracing Platform](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/distributed_tracing/distr-tracing-rn), configuring the collector following the [OpenShift documentation for configuring the collector](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/red_hat_build_of_opentelemetry/configuring-the-collector). In the following sections, we'll provide an overview of the process.

Unlike the previous approach, we want to collect traces globally in the OpenShift main console rather than in a dedicated project or namespace. Since the console is unique for the entire cluster, tracing data may come from multiple tenants.

You can access traces directly through the OpenShift console under `Observe > Traces`.



<p id="gdcalert4" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image4.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert5">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image4.png "image_tooltip")


The interface allows you to filter tracing spans by different criteria: tenants, service name, namespace, status, span or trace duration, custom attributes, or according to a given time range.

This view provides:



* Duration Graph: Visual timeline showing trace distribution and duration over time
* Trace List: Filterable table of all traces with span counts, durations, and timestamps
* Service Filtering: Ability to filter traces by service (request-manager, agent-service, llamastack, snow-mcp-server, etc.)
* Quick Access: Click any trace to view detailed span breakdown



<p id="gdcalert5" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image5.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert6">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image5.png "image_tooltip")


The custom resource type `opentelemetry.io/v1beta1.OpenTelemetryCollector` is managed by the **Red Hat build of OpenTelemetry Operator**. This operator is available through OpenShift's Operator Lifecycle Manager (OLM) and can be installed from the `redhat-operators` catalog. The operator handles the deployment, configuration, and lifecycle management of OpenTelemetry Collector instances based on the `OpenTelemetryCollector` custom resource definitions.

The  [lls-observability quickstart](https://github.com/rh-ai-quickstart/lls-observability/blob/main/helm/02-observability/otel-collector/otel-collector.yaml), provides the full details on how to deploy the OpenShift Distributed Tracing Platform.


## Wrapping Up

In this post we’ve taken you through instrumenting a Llama Stack based agentic system with Open Telemetry and shared some of what we learned as we developed the  [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) quickstart. We hope you are now interested to learn more about the overall implementation.


## Next steps

**Try the quickstart yourself**: Run through the [quickstart](https://github.com/rh-ai-quickstart/it-self-service-agent) (60-90 minutes) to deploy a working multi-agent system.

**Time savings**: Rather than spending 2-3 weeks building agent orchestration, evaluation frameworks, and enterprise integrations from scratch, you'll have a working system in under 90 minutes. Start in testing mode (simplified setup, mock eventing) to explore quickly, then switch to production mode (Knative Eventing + Kafka) when ready to scale.

**What you'll learn**: Production patterns for AI agent systems that apply beyond IT automation—how to test non-deterministic systems, implement distributed tracing for async AI workflows, integrate LLMs with enterprise systems safely, and design for scale. These patterns transfer to any agentic AI project.

**Customization path**: The laptop refresh agent is just one example. The same framework supports Privacy Impact Assessments, RFP generation, access requests, software licensing—or your own custom IT processes. Swap the specialist agent, add your own MCP servers for different integrations, customize the knowledge base, and define your own evaluation metrics.


## To learn more

If this blog post has sparked your interest in the IT self-service agent quickstart, here are additional resources:

**Explore more quickstarts**: Browse the [AI Quickstarts catalog](https://docs.redhat.com/en/learn/ai-quickstarts) for other production-ready use cases including fraud detection, document processing and customer service automation.

**Get help**: Questions or issues? Open an issue on the [GitHub repository](https://github.com/rh-ai-quickstart/it-self-service-agent/issues) or file an issue.

**Learn more about the tech stack**:



* [Llama Stack documentation](https://github.com/meta-llama/llama-stack)
* [MCP Protocol specification](https://modelcontextprotocol.io/)
* [Red Hat OpenShift AI documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.0)