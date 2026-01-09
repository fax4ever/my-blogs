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

One way to visualize the spans we've produced in the previous sections is to deploy an all-in-one Jaeger server instance in a specific namespace.

Below we'll show you how to deploy Jaeger on a Kubernetes cluster. For more information, please refer to the [Jaeger Getting Started guide](https://www.jaegertracing.io/docs/2.14/getting-started/).

Deploy the server instance:

```shell
kubectl create deployment jaeger --image=cr.jaegertracing.io/jaegertracing/jaeger:2.12.0 -n $NAMESPACE || echo "Deployment already exists"
```

Add labels to the deployment. These labels will be used by the network policy to control access to the Jaeger UI:

```shell
kubectl label deployment jaeger -n $NAMESPACE \
		app.kubernetes.io/instance=self-service-agent \
		app.kubernetes.io/name=self-service-agent \
		--overwrite || true
```

Also add the same labels to the pod template within the deployment:

```shell
kubectl patch deployment jaeger -n $NAMESPACE --type=json -p='[{"op":"add","path":"/spec/template/metadata/labels/app.kubernetes.io~1instance","value":"self-service-agent"},{"op":"add","path":"/spec/template/metadata/labels/app.kubernetes.io~1name","value":"self-service-agent"}]' || true
```

Create a network policy to allow ingress traffic to the Jaeger UI:

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

Create a service to expose the Jaeger query UI:

```shell
kubectl expose deployment jaeger --port=16686 --name=jaeger-ui -n $NAMESPACE || echo "Service already exists"
```

Create a service for the Jaeger collector to receive spans via HTTP:

```shell
kubectl expose deployment jaeger --port=4318 --name=jaeger-otlp-http -n $NAMESPACE || echo "OTLP HTTP service already exists"
```

Optionally, create a service for the Jaeger collector to receive spans via gRPC:

```shell
kubectl expose deployment jaeger --port=4317 --name=jaeger-otlp-grpc -n $NAMESPACE || echo "OTLP gRPC service already exists"
```

Finally, create an OpenShift route to access the Jaeger UI from outside the cluster:

```shell
oc create route edge jaeger-ui --service=jaeger-ui -n $NAMESPACE || echo "Route already exists"
```

To configure your application pods to send traces to this Jaeger collector, set the following environment variable:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger-otlp-http.$NAMESPACE.svc.cluster.local:4318
```

Note that the collector is only accessible from within the OpenShift cluster.

Below are some example traces produced by the [it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent):

![alt_text](images/image2.png "image_tooltip")

Here's the same data shown in a graph view:

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

In this example, the overall request time is dominated by the LLM inference call, that is pretty common for an agentic application.

## Collect the tracing with the OpenShift Distributed Tracing Platform

The solution we described in the previous chapter is more suitable for a test / development environment.
For production we reccomand to collect the tracing using the [OpenShift Distributed Tracing Platform](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/distributed_tracing/distr-tracing-rn), configuring the collector following the [OpenShift documentation for configuring the collector](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/red_hat_build_of_opentelemetry/configuring-the-collector). In the following we'll give an idea of the process.

First of all, this time we want to collect the tracing globally in the OpenShift main console and not a dedicated project / namespace. Since the console is unique for entire cluster, tracing data may come from multiple tenants.

You can access traces directly through the OpenShift console under `Observe > Traces`.

![alt_text](images/image4.png "image_tooltip")

The mask allows to filter tracing spans by different criteria: tenants, service name, namespace, status, span or trace durantion, custom attributes or according to a given time range.

This view provides:

* Duration Graph: Visual timeline showing trace distribution and duration over time
* Trace List: Filterable table of all traces with span counts, durations, and timestamps
* Service Filtering: Ability to filter traces by service (request-manager, agent-service, llamastack, snow-mcp-server, etc.)
* Quick Access: Click any trace to view detailed span breakdown

![alt_text](images/image5.png "image_tooltip")

The observability services are usually deployed in a dedicated namespace on the OpenShift platform.
In this example we image we choosen the name 'observability-hub'.
In this namespace we've defined an otel-collector resource like the following:

``` yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
spec:
  serviceAccount: otel-collector
  mode: deployment
  upgradeStrategy: automatic
  ingress:
    route:
      termination: passthrough
    type: route
  config:
    extensions:
      bearertokenauth:
        filename: "/var/run/secrets/kubernetes.io/serviceaccount/token"

    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      batch:
        send_batch_size: 100
        timeout: 1s
      memory_limiter:
        check_interval: 5s
        limit_percentage: 95
        spike_limit_percentage: 25

    exporters:
      debug:
        verbosity: basic
      otlphttp/dev:
        endpoint: https://tempo-tempostack-gateway.observability-hub.svc.cluster.local:8080/api/traces/v1/dev
        headers:
          X-Scope-OrgID: dev
        tls:
          insecure: false
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
        auth:
          authenticator: bearertokenauth    

    service:
      extensions:
        - bearertokenauth
      pipelines:
        traces:
          receivers:
            - otlp
          processors:
            - batch
            - memory_limiter
          exporters:
            - debug
            - otlphttp/dev
      telemetry:
        metrics:
          address: 0.0.0.0:8888
```

The custom resource type `opentelemetry.io/v1beta1.OpenTelemetryCollector` is managed by the **Red Hat build of OpenTelemetry Operator**. This operator is available through OpenShift's Operator Lifecycle Manager (OLM) and can be installed from the `redhat-operators` catalog. The operator handles the deployment, configuration, and lifecycle management of OpenTelemetry Collector instances based on the `OpenTelemetryCollector` custom resource definitions.

The operator will create a service in the same namespace, using the name we provided in the resource, `otel-collector` in this example. The hostname should be used to fill the `OTEL_EXPORTER_OTLP_ENDPOINT` variable in all the pods need to export the tracing.

In our case will be:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector-collector.observability-hub.svc.cluster.local:4318
```

In the `OpenTelemetryCollector` under the config specification it is possible to specify all the [OpenTelemetry standard components](https://opentelemetry.io/docs/collector/components/), such as receivers to collect telemetry data from various sources and formats, processors to transform, filter, and enrich telemetry data or exporters to send telemetry data to observability backends.

Other specifications are provided for instance to enable an external route to the collector service and to speficity the service account used to access to the service. Usually a cluster role is binded to a specific service account, in order to selectivly allow the services to send the tracing data to the collector. Please look at [the official documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/red_hat_build_of_opentelemetry/configuring-the-collector) to configure the collector properly on your OpenShift platform.

In the configuration we are exporting the tracing to a Tempo Stack gateway, that has been installed using another custom resource of kind `tempo.grafana.com/v1alpha1.TempoStack` defined in the same 'observability-hub' namespace:

```yaml
apiVersion: tempo.grafana.com/v1alpha1
kind: TempoStack
metadata:
  name: tempostack
  namespace: observability-hub
  labels:
    app.kubernetes.io/component: tempo
    app.kubernetes.io/instance: tempo
    app.kubernetes.io/name: tempo-stack
    app.kubernetes.io/part-of: observability
spec:
  storage:
    secret:
      name: minio-tempo
      type: s3
  storageSize: 15Gi
  resources:
    total:
      limits:
        cpu: '5'
        memory: 10Gi
  tenants:
    mode: openshift
    authentication:
      - tenantId: 1610b0c3-c509-4592-a256-a1871353dbfa
        tenantName: dev
  template:
    gateway:
      enabled: true
    queryFrontend:
      jaegerQuery:
        # we're using the OpenShift console as for querying tracing data
        enabled: false
```

Once the resource is added to the `observability-hub` namespace, the `Tempo Operator` will create the service gateway and all the infrastructure to collect the tracing data, and persist it for instance on S3 MinIO storage. For more information see [Distributed Tracing documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/distributed_tracing/index).

The last step is to use the OpenShift observability UI plugin to see the tracing data on the OpenShift console. Choosing `UI Plugin` tab on the `Cluster Observability Operator` page, as explained in the [Cluster Observability Operator documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_cluster_observability_operator/1-latest/html/ui_plugins_for_red_hat_openshift_cluster_observability_operator/distributed-tracing-ui-plugin).

If you want go deeper in the configuration of the observability intrastructure on the OpenShift platform, there is [this other quickstart project](https://github.com/rh-ai-quickstart/lls-observability) that shown in great detail all the reccomanded options that you have at the moment.












