# How to Enable APM-Infra Correlation 

In the APM Service Dashboards, there are charts that indicate the health of
the underlying infrastructure. The default configuration of Splunk OpenTelemetry 
Connector automatically configures this for you, but when specifying a custom 
configuration it is important to keep the following in mind.



## Configuration to enable APM-Infra Correlation from a standalone VM (Agent)
In order to get your infrastructure data to show up, you need to enable the 
following in the otel-collector:

- `hostmetrics` receiver
    - cpu, memory, filesystem and network enabled
   
    <em>This will allow the collection of cpu, mem, disk and network metrics.</em>


- `signalfx` exporter 
    - sync_host_metadata enabled

    <em>The `signalfx` exporter will aggregate the metrics from the `hostmetrics` 
    receiver and send in metrics such as `cpu.utilization`, which are utilized in 
    the relevant charts.</em>
    
    - `signalfx` exporter must be enabled for metrics AND traces pipeline (example below)

    <em>The `sync_host_metadata` option of the `signalfx` exporter will allow the 
    collector to make relevant API calls to correlate the APM metrics with the 
    infrastructure metrics, but to do this the `signalfx` exporter must be placed 
    in the traces pipeline.</em>

- `resourcedetection` processor 
    - cloud provider/env variable to set `host.name`
    - override enabled
    
    <em>This processor will enable a unique `host.name` value to be set for metrics 
    and traces. The `host.name` is determined by either the ec2 host name, or the 
    system host name.</em>
   

-  `resource/add_environment` processor (optional)
    
    <em>This processor inserts a `deployment.environment` span tag to all the spans. The APM
    charts require the environment span tag to be correctly set. When possible, it is 
    recommended to configure this span tag in the instrumentation. However, if that is not
    possible then you can use this processor to insert the required `deployment.environment`
    span tag value.</em>

 
Here are the relevant snippets from each section:
```
...
receivers:
  hostmetrics:
    collection_interval: 10s
    scrapers:
      cpu:
      disk:
      filesystem:
      memory:
      network:
...
processors:
  resourcedetection:
    detectors: [system,env,gce,ec2]
    override: true
  resource/add_environment:
    attributes:
      - action: insert
        value: staging
        key: deployment.environment
...
exporters:
  # Traces
  sapm:
    access_token: "${SPLUNK_ACCESS_TOKEN}"
    endpoint: "${SPLUNK_TRACE_URL}"
  # Metrics + Events + APM correlation calls
  signalfx:
    access_token: "${SPLUNK_ACCESS_TOKEN}"
    api_url: "${SPLUNK_API_URL}"
    ingest_url: "${SPLUNK_INGEST_URL}"
    sync_host_metadata: true
...
service:
  extensions: [health_check, http_forwarder, zpages]
  pipelines:
    traces:
      receivers: [jaeger, zipkin]
      processors: [memory_limiter, batch, resourcedetection, resource/add_environment]
      exporters: [sapm, signalfx]
    metrics:
      receivers: [hostmetrics]
      processors: [memory_limiter, batch, resourcedetection]
      exporters: [signalfx]
```     

## Configuration to enable APM-Infra Correlation from an Agent -> Gateway
If you need to run the otel-collector as an Agent and a Gateway, then you should follow
the steps listed below for running the collector in each mode (Agent vs Gateway). 

### Agent
Follow the same steps as mentioned for a standalone VM and also include the following changes.

- `http_forwarder` extension
    - `egress` endpoint

    <em>The `http_forwarder` listens on port 6060 and sends all the REST API calls directly
    to the Splunk Observability Cloud. If your Agent does not have access to talk to the 
    Splunk SaaS backend directly, this should be changed to the URL of the Gateway.</em>

- `signalfx` exporter
    - `api_url` endpoint
    
    <em>The `api_url` endpoint should be set to the URL of the Gateway, and you MUST specify 
    the ingress port of the `http_forwarder` of the Gateway (6060 by default).</em>
    
    - `ingest_url` endpoint 
    
    <em>The `ingest_url` endpoint should be set to the URL of the Gateway, and just like the 
    `api_url` you MUST specify the ingress port of the `signalfx` receiver of the Gateway
    (9943 by default).</em>
    
    <em>You can choose to send all your metrics via otlp to the gateway, but you must send the
    metrics collected by the `hostmetrics` receiver via the `signalfx` exporter so that the
    aggregated metrics can be computed and sent to the Splunk Observability Cloud.</em>

- All pipelines
    - Metrics, Traces and Logs pipeline should be sent to the appropriate receivers on the 
    Gateway. 

    - `otlp` exporter (optional)
    
    <em>The `otlp` exporter uses the grpc protocol, so the endpoint must be defined as the IP 
    address of the gateway. Using the `otlp` exporter is optional, but recommended for the majority 
    of your traffic from the Agent to the gateway (see NOTE below)</em>

    **NOTE**: <em>Apart from the requirement of using the `signalfx` exporter [for aggregated 
    `hostmetrics` in the metrics pipeline and for making REST API calls in the traces pipeline], 
    the rest of the data between the Agent to the Gateway should ideally be sent via the `otlp` 
    exporter. Since `otlp` is the internal format that all data gets converted to upon receival, 
    it is the most efficient way to send data to the Gateway (example below)</em>

Here are the relevant snippets from each section:
```
...
receivers:
  hostmetrics:
    collection_interval: 10s
    scrapers:
      cpu:
      disk:
      filesystem:
      memory:
      network:
...
processors:
  resourcedetection:
    detectors: [system,env,gce,ec2]
    override: true
  resource/add_environment:
    attributes:
      - action: insert
        value: staging
        key: deployment.environment
...
exporters:
  # Traces
  otlp:
    endpoint: "${SPLUNK_GATEWAY_URL}:4317"
    insecure: true
  # Metrics + Events + APM correlation calls
  signalfx:
    access_token: "${SPLUNK_ACCESS_TOKEN}"
    api_url: "http://${SPLUNK_GATEWAY_URL}:6060"
    ingest_url: "http://${SPLUNK_GATEWAY_URL}:9943"
    sync_host_metadata: true
...
service:
  extensions: [health_check, http_forwarder, zpages]
  pipelines:
    traces:
      receivers: [jaeger, zipkin]
      processors: [memory_limiter, batch, resourcedetection, resource/add_environment]
      exporters: [otlp, signalfx]
    metrics:
      receivers: [hostmetrics]
      processors: [memory_limiter, batch, resourcedetection]
      exporters: [signalfx]
``` 

### Gateway
On the side of the Gateway, you'd need the relevant receivers to match the exporters from
the Agent. In addition, you'd need the following specific changes.

- `http_forwarder` extension
    - `egress` endpoint

    <em>The `http_forwarder` listens on port 6060 and sends all the REST API calls directly
    to the Splunk Observability Cloud. On the gateway, this should be changed to the 
    Splunk Observability Cloud SaaS endpoint.</em>

- `signalfx` exporter
    - `translation_rules` and `exclude_metrics` flag
    
    <em>Both flags should be set to an empty list. This will ensure that the metric aggregations performed at the Agent by the `signalfx` exporter do NOT get overriden by any translation/exclusion rules at the Gateway.</em>

Here are the relevant snippets from each section:
```
...
extensions:
  http_forwarder:
    egress:
      endpoint: "https://api.${SPLUNK_REALM}.signalfx.com"
...
receivers:
  otlp:
    protocols:
      grpc:
      http:
  signalfx:
...
exporters:
  # Traces
  sapm:
    access_token: "${SPLUNK_ACCESS_TOKEN}"
    endpoint: "https://ingest.${SPLUNK_REALM}.signalfx.com/v2/trace"
  # Metrics + Events
  signalfx:
    access_token: "${SPLUNK_ACCESS_TOKEN}"
    realm: "${SPLUNK_REALM}"
    translation_rules: []
    exclude_metrics: []
...
service:
  extensions: [http_forwarder]
  pipelines:
    traces:
      receivers: [otlp]
      processors:
      - memory_limiter
      - batch
      exporters: [sapm, signalfx]
    metrics:
      receivers: [otlp, signalfx]
      processors: [memory_limiter, batch]
      exporters: [signalfx]
``` 