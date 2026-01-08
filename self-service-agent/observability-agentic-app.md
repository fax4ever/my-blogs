<!-----

You have some errors, warnings, or alerts. If you are using reckless mode, turn it off to see useful information and inline alerts.
* ERRORs: 0
* WARNINGs: 0
* ALERTS: 1

Conversion time: 0.799 seconds.


Using this Markdown file:

1. Paste this output into your source file.
2. See the notes and action items below regarding this conversion run.
3. Check the rendered output (headings, lists, code blocks, tables) for proper
   formatting and use a linkchecker before you publish this page.

Conversion notes:

* Docs™ to Markdown version 2.0β1
* Thu Jan 08 2026 03:50:09 GMT-0800 (PST)
* Source doc: Debugging Autonomy: Advanced Observability for a Self-Service IT Agent
* Tables are currently converted to HTML tables.
* This document has images: check for >>>>>  gd2md-html alert:  inline image link in generated source and store images to your server. NOTE: Images in exported zip file from Google Docs may not appear in  the same order as they do in your doc. Please check the images!

----->


<p style="color: red; font-weight: bold">>>>>>  gd2md-html alert:  ERRORs: 0; WARNINGs: 0; ALERTS: 1.</p>
<ul style="color: red; font-weight: bold"><li>See top comment block for details on ERRORs and WARNINGs. <li>In the converted Markdown or HTML, search for inline alerts that start with >>>>>  gd2md-html alert:  for specific instances that need correction.</ul>

<p style="color: red; font-weight: bold">Links to alert messages:</p><a href="#gdcalert1">alert1</a>

<p style="color: red; font-weight: bold">>>>>> PLEASE check and correct alert issues and delete this message and the inline alerts.<hr></p>



# Debugging Autonomy: Advanced Observability for a Self-Service IT Agent

Agentic applications usually involve complex interactions between multiple components—routing agents, specialist agents, knowledge bases, MCP servers, and external systems—making production debugging challenging without proper visibility.

In this blog post we will describe in detail a possible approach to setup distributed tracing on an agentic workflow application, based on what we learned while developing the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent).

At the end of this journey we will have configured OpenTelemetry support for distributed tracing across all the different typical agentic applications components, enabling you to track requests end-to-end through all the application workloads, MCP Servers, and Llama Stack. By integrating with OpenShift's observability stack, you gain unified monitoring across all platform components alongside your existing infrastructure metrics.

Red Hat AI quickstarts are a catalog of ready-to-run industry-specific use cases for your Red Hat AI environment. Each quickstart is designed to be simple to deploy, explore, and extend. They give teams a fast, hands-on way to see how AI can power solutions on enterprise-ready, open source infrastructure. You can read more about quickstarts in “AI Quickstarts: an easy and practical way to get started with Red Hat AI” and the it-self-service-agent in particular in <span style="text-decoration:underline;">An introduction to the Self Service Agent - laptop refresh quickstart</span>.

This is the third post in a series covering what we learned while developing the it-self-service-agent quickstart, which will be as follows:



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


In the first part of this article we will explain how to produce tracing spans from the application workloads, assuming that an OpenTelemetry collector is available on a given remote address. While in the second part we will suggest some way of configuring the tracing infrastructure backing the collector in particular using [OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift/container-platform).

In this guide the tracing is configured following the OpenTelemetry version 1.37.0, but of course different versions may be used. For more information on the standard, see the [OpenTelemetry page.](https://opentelemetry.io/) This guide will be focused on to produce tracing from agentic and Python workload, but libraries to produce tracing following OpenTelemetry are available from any popular programming languages.


## Autoinstrumentation: HTTP Clients and FastAPI

The spans we want to produce usually depend on the frameworks used by the workloads. For instance in the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) application several of the communication between the workloads are implemented as REST calls. It turns out that we want to collect in the tracing those calls producing a span for each invocation, keeping at the same time the causal relation between them.

Whenever a library supports auto-instrumentation, we should consider the possibility of using it. For instance we used the libraries [OpenTelemetry instrumentation HTTPx](https://pypi.org/project/opentelemetry-instrumentation-httpx/) and [OpenTelemetry FastAPI Instrumentation](https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/fastapi/fastapi.html) to instrument all the REST server and client requests and responses. Thanks to this we produce span for all the calls without the need to write potentially unsafe custom code, also the causality and the span correlations are automatically handled by those libraries and we don’t need to manually pass the context from the parent calls to the child calls.

In our project, in all the modules that use those frameworks we added the libraries together with the base libraries for the OpenTelemetry APIs, SDK and the exporter.

For instance in our our project’s module we imported the following dependencies:


```
    "opentelemetry-exporter-otlp-proto-http==1.37.0",
    "opentelemetry-instrumentation-httpx==0.58b0",
    "opentelemetry-api>=1.37.0",
    "opentelemetry-sdk>=1.37.0",
    "opentelemetry-instrumentation-fastapi>=0.58b0"
```


When autoinstrumentation is used usually we define a couple of environment variables that may or may not be taken in account by the instrumentation framework (it depends on the implementations).

For instance in our project’s autoinstrumented service workloads we set the following variables:


```
ENV OTEL_SERVICE_NAME=${SERVICE_NAME}
ENV OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true
ENV OTEL_EXPORTER_OTLP_ENDPOINT=${OTEL_EXPORTER_OTLP_ENDPOINT}
```


In particular, we used the presence of the exporter OpenTelemetry protocol variable to programmatically configure the trace and the span and batch processors.

To propagate the context in order to follow the causality in the span correlation we used the following code:


```
from opentelemetry.propagate import set_global_textmap
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

# Set up propagator
set_global_textmap(TraceContextTextMapPropagator())
```


To enable the HTTPX client autoinstrumentation:


```
# Set up instrumentations
HTTPXClientInstrumentor().instrument()
```


To enable the FastAPI instrumentation:


```
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# app is FastAPI instance
FastAPIInstrumentor.instrument_app(app)
```



### Passing OTEL properties to multiple workloads

[It-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) is deployed as a single large Helm application, so tracing is configured globally using single HELM values that will be automatically propagated to all the workloads.

For instance if the HELM value otelExporter is set, all the workloads will set the environment variable OTEL_EXPORTER_OTLP_ENDPOINT to the same value, enabling tracing export to the very same OTEL collector globally.


## Manual Instrumentation: MCP Servers 

For some libraries such as FastMCP at the time of the development we didn’t find any viable solution for auto instrumentation. So we opted for manual instrumentation to trace mcp server calls.

Manual instrumentation should be consistent with the automatic instrumentation. That is we want the causally related spans to be correlated, even though some are produced manually and others automatically. In order to do that, in the manual instrumentation we need to correctly extract the parent context, if present, from the MCP request and use it as parent span context when the child span is created. 

Here is a code example inspired by the solution we implemented in the [It-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent):


```
from opentelemetry.propagate import extract
                
headers = ctx.request_context.headers
# Convert headers to dict-like format expected by propagator
# Normalize header keys to lowercase as traceparent is case-insensitive
carrier = {k.lower(): v for k, v in dict(headers).items()}

# Extract context from headers using the global propagator
parent_context = extract(carrier)

with tracer.start_as_current_span(span_name, context=parent_context) as span:
  span.set_attribute("mcp.tool.name", func.__name__)
  
  for i, arg in enumerate(args):
    span.set_attribute(f"mcp.tool.arg.{i}", str(arg))  

  for key, value in kwargs.items():
    span.set_attribute(f"mcp.tool.param.{key}", str(value))
  
  # Execute the tool function
  # The span context will automatically propagate to any
  # instrumented HTTP calls made within this function
  result = func(*args, **kwargs) 
  span.set_status(Status(StatusCode.OK))

  except Exception as e:
    # Record the exception and set error status
    span.record_exception(e)
    span.set_status(Status(StatusCode.ERROR, str(e)))
    raise
```



## Llama stack tracing configuration

Note that the solution we’ve developed has been tested on Llama Stack 0.2.x and 0.3.x series. 

To configure Llama Stack with more recent versions or for any other information, please see the [Llama Stack telemetry documentation](https://llamastack.github.io/docs/providers/telemetry/inline_meta-reference).

In the Llama Stack configuration we define the telemetry to include the OTEL exporter and the otel_trace sink if an otelExporter is defined at root application level.

For Llama Stack 0.2.x series we can define the telemetry like:


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


While with Llama Stack 0.3.x series we only need to set the environment variables:


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



## Configure Jaeger OTEL collector
