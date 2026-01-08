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

Agentic applications typically involve complex interactions between multiple components—routing agents, specialist agents, knowledge bases, MCP servers, and external systems—making production debugging challenging without proper visibility.

In this blog post, we will describe in detail a practical approach to setting up distributed tracing for an agentic workflow application, based on lessons learned while developing the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent).

By the end of this journey, you will have configured OpenTelemetry support for distributed tracing across all typical agentic application components, enabling you to track requests end-to-end through application workloads, MCP Servers, and Llama Stack. By integrating with OpenShift's observability stack, you'll gain unified monitoring across all platform components alongside your existing infrastructure metrics.

Red Hat AI quickstarts are a catalog of ready-to-run industry-specific use cases for your Red Hat AI environment. Each quickstart is designed to be simple to deploy, explore, and extend. They give teams a fast, hands-on way to see how AI can power solutions on enterprise-ready, open source infrastructure. You can read more about quickstarts in "AI Quickstarts: an easy and practical way to get started with Red Hat AI" and the it-self-service-agent in particular in _An introduction to the Self Service Agent - laptop refresh quickstart_.

This is the third post in a series covering lessons learned while developing the it-self-service-agent quickstart:

* An introduction to the Self Service Agent - laptop refresh quickstart
* AI Meets You Where You Are: Seamless Integration with Slack, Email, and ServiceNow
* Big Prompt vs. Small Prompt: Architecting Your Agent for Speed, Accuracy, and Efficiency
* Unlocking Agents: Leveraging the Responses API within the Llama Stack Ecosystem
* Eval-Driven Development: Building Reliable Agentic Applications
* **Debugging Autonomy: Advanced Observability for a Self-Service IT Agent** - this blog post
* Elevating the Enterprise: Our Model Context Protocol (MCP)-Powered ServiceNow Integration Experience

If you are interested in a bit more detail on the business benefits of using agentic AI to automate IT processes, check out _AI quickstart: Implementing IT processes with agentic AI on Red Hat OpenShift AI_.

## Agentic Workloads Distributed Tracing



<p id="gdcalert1" ><span style="color: red; font-weight: bold">>>>>>  gd2md-html alert: inline image link here (to images/image1.png). Store image on your image server and adjust path/filename/extension if necessary. </span><br>(<a href="#">Back to top</a>)(<a href="#gdcalert2">Next alert</a>)<br><span style="color: red; font-weight: bold">>>>>> </span></p>


![alt_text](images/image1.png "image_tooltip")

In the first part of this article, we will explain how to produce tracing spans from application workloads, assuming an OpenTelemetry collector is available at a given remote address. In the second part, we will explore ways to configure the tracing infrastructure backing the collector, particularly using [OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift/container-platform).

This guide configures tracing using OpenTelemetry version 1.37.0, though other versions can be used. For more information about the standard, see the [OpenTelemetry page](https://opentelemetry.io/). While this guide focuses on producing traces from agentic and Python workloads, libraries for generating OpenTelemetry-compatible traces are available for most popular programming languages.

## Autoinstrumentation: HTTP Clients and FastAPI

The spans we want to produce typically depend on the frameworks used by the workloads. For instance, in the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) application, much of the communication between workloads is implemented as REST calls. We want to capture these calls in our tracing, producing a span for each invocation while maintaining the causal relationships between them.

Whenever a library supports auto-instrumentation, we should take advantage of it. For example, we used the [OpenTelemetry instrumentation HTTPx](https://pypi.org/project/opentelemetry-instrumentation-httpx/) and [OpenTelemetry FastAPI Instrumentation](https://opentelemetry-python-contrib.readthedocs.io/en/latest/instrumentation/fastapi/fastapi.html) libraries to instrument all REST server and client requests and responses. This approach produces spans for all calls without requiring potentially error-prone custom code. Additionally, causality and span correlations are automatically handled by these libraries, eliminating the need to manually pass context from parent calls to child calls.

In our project, we added these instrumentation libraries to all modules that use the frameworks, along with the base libraries for the OpenTelemetry APIs, SDK, and exporter.

For instance, in our project's modules we imported the following dependencies:

```
    "opentelemetry-exporter-otlp-proto-http==1.37.0",
    "opentelemetry-instrumentation-httpx==0.58b0",
    "opentelemetry-api>=1.37.0",
    "opentelemetry-sdk>=1.37.0",
    "opentelemetry-instrumentation-fastapi>=0.58b0"
```

When using autoinstrumentation, we typically define environment variables that may or may not be used by the instrumentation framework (depending on the implementation).

For instance, in our project's autoinstrumented service workloads, we set the following variables:

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

[It-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent) is deployed as a single Helm application, so tracing is configured globally using HELM values that are automatically propagated to all workloads.

For instance, if the HELM value `otelExporter` is set, all workloads will set the environment variable `OTEL_EXPORTER_OTLP_ENDPOINT` to the same value, enabling tracing export to the same OTEL collector globally.

## Manual Instrumentation: MCP Servers

For some libraries such as FastMCP, we didn't find any viable auto-instrumentation solution at the time of development. Therefore, we opted for manual instrumentation to trace MCP server calls.

Manual instrumentation should remain consistent with automatic instrumentation. We want causally related spans to be correlated, even when some are produced manually and others automatically. To achieve this, the manual instrumentation must correctly extract the parent context (if present) from the MCP request and use it as the parent span context when creating the child span.

Here is a code example inspired by the solution we implemented in the [It-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent):

```python
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

  try:
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

## Llama Stack tracing configuration

Note that our solution has been tested on Llama Stack 0.2.x and 0.3.x series.

To configure Llama Stack with more recent versions or for additional information, please see the [Llama Stack telemetry documentation](https://llamastack.github.io/docs/providers/telemetry/inline_meta-reference).

In the Llama Stack configuration, we define the telemetry to include the OTEL exporter and the `otel_trace` sink if an `otelExporter` is defined at the root application level.

For Llama Stack 0.2.x series, we can define the telemetry as follows:

```yaml
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

```yaml
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

One option to see the spans we’ve produced in the previous sections is just deploying an all-in-one Jager server instance in a given $(NAMESPACE).

In the following we'll present a way to deploy on a Kubernetes cluster, for any doubts please see the [Jaeger Getting Started guide](https://www.jaegertracing.io/docs/2.14/getting-started/).

Deploy the server instance, for instance 

```shell
kubectl create deployment jaeger --image=cr.jaegertracing.io/jaegertracing/jaeger:2.12.0 -n $NAMESPACE || echo "Deployment already exists"
```

Add some labels that will be used later to define the network policy to expose the route to the Jaeger UI:

```shell
kubectl label deployment jaeger -n $NAMESPACE \
		app.kubernetes.io/instance=self-service-agent \
		app.kubernetes.io/name=self-service-agent \
		--overwrite || true
```    

and

```shell
kubectl patch deployment jaeger -n $NAMESPACE --type=json -p='[{"op":"add","path":"/spec/template/metadata/labels/app.kubernetes.io~1instance","value":"self-service-agent"},{"op":"add","path":"/spec/template/metadata/labels/app.kubernetes.io~1name","value":"self-service-agent"}]' || true
```

Create the network policy that will be used by the route to the Jaeger UI:

```shell
printf '%s\n' \
  'apiVersion: networking.k8s.io/v1' \
  'kind: NetworkPolicy' \
  'metadata:' \
  '  name: jaeger-allow-ingress' \
  "  namespace: $NAMESPACE" \
  '  labels:' \
  '    app: jaeger' \
  'spec:' \
  '  podSelector:' \
  '    matchLabels:' \
  '      app.kubernetes.io/instance: self-service-agent' \
  '      app.kubernetes.io/name: self-service-agent' \
  '      app: jaeger' \
  '  policyTypes:' \
  '  - Ingress' \
  '  ingress:' \
  '  - from:' \
  '    - namespaceSelector:' \
  '        matchLabels:' \
  '          network.openshift.io/policy-group: ingress' \
  '    ports:' \
  '    - protocol: TCP' \
  '      port: 16686' \
  | kubectl apply -f - || echo "Network policy creation failed, may already exist"
```

Create the Jaeger query UI service:

```shell
kubectl expose deployment jaeger --port=16686 --name=jaeger-ui -n $NAMESPACE || echo "Service already exists"
```

Create the Jaeger collector service for collecting spans using HTTP protocol:

```shell
kubectl expose deployment jaeger --port=4318 --name=jaeger-otlp-http -n $NAMESPACE || echo "OTLP HTTP service already exists"
```

Optionally, create the Jaeger collector service for collecting spans using gGRP protocol:

```shell
kubectl expose deployment jaeger --port=4317 --name=jaeger-otlp-grpc -n $NAMESPACE || echo "OTLP gRPC service already exists"
```

Finally, create the route:

```shell
oc create route edge jaeger-ui --service=jaeger-ui -n $NAMESPACE || echo "Route already exists"
```

To use the server as collector, on the sender pods you can set:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger-otlp-http.$NAMESPACE.svc.cluster.local:4318"
```

The collector will be accessible within the OpenShift cluster only.

In the following you can find some traces produced by the [It-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent):

![alt_text](images/image2.png "image_tooltip")

Also a graph view:

![alt_text](images/image3.png "image_tooltip")

You can notice that the structure may be quite involved, for instance a tipical request several nested calls. For instance:

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

