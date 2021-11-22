### Building a distributed tracing pipeline with open telemetry collector, data prepper and opensearch trace analytics

Over the past few years, the importance of observability when developing and managing applications has spiked with the spotlight firmly on micro-services, service mesh. Distributed services can be unpredictable and despite our best efforts, failures and performance bottlenecks in such systems are inevitable—​and can be difficult to isolate. In such an environment, having deep visibility into the behavior of your applications is critical for software development teams and operators.

The landscape for the observability continues to grow as we speak. In particular, when it comes to metrics, error logging and distributed traces; these can provide valuable information to optimize performance, troubleshoot application issues to make a service more reliable. With this in mind, it makes sense to create a distributed tracing pipeline to ingest, process and vizualize tracing data with query/alerting featureset.

At Dow Jones, we started a similar journey as we continue to move our applications and microservices to our next generation service mesh based on EKS and [istio](https://istio.io/). With the move to this PaaS, one of the requirements was to have a distributed tracing pipeline that could scale to the amount of traces we generate, and would work for both our microservices running the PaaS and more legacy applications. This is where the idea of opentelmetry fit perfectly into our requirement set. As OpenTelemetry’s goal is to provide full support for traces, metrics, and logs and provide a single implementation that can be leveraged.

[Traces was the first to reach stable status for some of its component in open telemetry project](https://opentelemetry.io/status/). Since then almost all but collector has reached a stable maturity level and metrics are well on their way with initial work on logs also started. Opentelemetry was also added within CNCF incubating landscape, with great community support and contributions. We recently launched a distributed pipeline for our service meshes and more in production based on OpenTelemetry and I will be going over some of the design decisions we took, our setup and source code to DIY.

#### Tracing Pipeline Components

With OpenTelmetry as the core of our pipeline we still needed to decide on the sink for our traces, tools to vizualize, query on the traces, and tools that help in exporting these traces to create a Federated architecture. After much deliberation, we decided on the following componets:

- **Opentelemetry** for creation, propagation, collection, processing and exporting trace data.
- **AWS opensearch** (formerly elasticsearch) as the sink for the traces
- **Data prepper** and **Jaeger** to re-format opentelemetry trace data and export it to opensearch in formats that both kibana and jaeger understand. (This should go away soon as both jaeger and AWS kibana will natively be able to use otlp traces)
- **Trace Analytics Plugin** for opensearch and Jaeger to vizualize and query traces

OpenTelemetry (OTEL) was formed by the merging of OpenTracing and OpenCensus. Currently a CNCF incubating project and its second most active in terms of contributions; Kubernetes being the first. OTEL since its inception aimed to offer a single set of APIs and libraries that standardise how you collect and transfer telemetry data.

Jaeger with its community engagement and helping track and measure requests and transactions by analyzing end-to-end data from service call chains so they can better understand latency issues in microservice architectures. Its exclusively meant for distributed tracing and provides a simple web UI that can be used to see traces and spans across services. Trace Analytics plugin and Jaeger enables views of each individual trace in a waterfall-style graph of all the related trace and span executions (similar to jaeger), which makes to easy to identify all the service invocations for each trace, time spent in each service and each span, and the payload content of each span, including errors.

The Trace Analytics plugin also aggregates trace data into Trace Group views and Service Map views, to enable monitoring insights of application performance on trace data. By going beyond the ability to search and analyze individual traces, these features enable developers to proactively identify application performance issues, not just react to problems when they occur.

Data Prepper is a new component of opensearch that receives trace data from the OpenTelemetry collector, and aggregates, transforms, and normalizes it for analysis and visualization in Kibana

Lastly, Opensearch is a community-driven, open source search and analytics suite derived from Apache 2.0 licensed Elasticsearch 7.10.2 & Kibana 7.10.2. Additonaly it comes with managed Trace Analytics plugin and a visualization and user interface, OpenSearch Dashboards.

#### Architecture

Once we got the components for the pipeline pencilled in, the remaining work to get these components to play nice with each others and scale to our requirement. The pipeline was launched to production and has been processing 4-6 tb of trace data daily. In a nutshell Applications use opentelemetry libraries/API to instrument traces and send it to open telemetry agents. Opentelemetry agents to batch and send traces from microservices to the opnetelemetry gateway collector. The collectors sample the traces and export them to backends which in our case is data prepper and jaeger collector. These backends format and batch push the data to opensearch. Lastly, we have Trace Analytics pluign for Kibana and Jaeger vizualizing this data from opensearch and queried by our internal users

<img src="img/tracing_step4.png" align="center" >

Let's break down each component ...

#### Breakdown

##### Step 1 : Creating and Propagating traces

###### Creating traces

We need to create/propagate traces to be able to use a distributed tracing pipeline. Open telemetry provides collection of tool such as API, SDK and integrates with popular languages and framework to integrate with greater OpenTelemetry ecosystem, such as OpenTelemetry Protocol (OTLP) and the OpenTelemetry Collector.

Open Telemetry provides a [status page](https://opentelemetry.io/status/) to keep track of its multiple tools as they go stable. It also provides for documentation on how to create distributed traces for your service both [manually or with auto instrumentation](https://opentelemetry.io/docs/concepts/instrumenting/)

While this is out of scope for this blog, the documentation provided should get you started with creating standard traces within your service.

###### Propagating traces

Once we have traces created for applications, it is important to be able to do context propagation to convert these traces to distributed traces. Context propagation facilitates the movement of context between services and processes. Context is injected into a request and extracted by a receiving service to parent new spans. That service may then make additional requests, and inject context to be sent to other services…and so on.

There are several protocols for context propagation that OpenTelemetry recognizes.

- [W3C Trace-Context HTTP Propagator](https://w3c.github.io/trace-context/)
- [W3C Correlation-Context HTTP Propagator](https://w3c.github.io/correlation-context/)
- [B3 Zipkin HTTP Propagator](https://github.com/openzipkin/b3-propagation)

At dow jones, we operate our own istio based service mesh hosted on AWS EKS PaaS. Istio leverages Envoy’s distributed tracing feature to provide tracing integration out of the box. Specifically, Istio provides options to install various tracing backend and configure proxies to send trace spans to them automatically. It requires application to propagate the [B3 Zipkin HTTP Propagator](https://github.com/openzipkin/b3-propagation) headers so that when the proxies send span information, the spans can be correlated correctly into a single trace. This natively works with open telemetry as well since this is a supported context open telemetry context propagation.

##### Data Collection

Once we have the telemetry data created and propagated through services, the OpenTelemetry project facilitates the collection of telemetry data via the [Open Telemetry Collector](https://opentelemetry.io/docs/collector/). The OpenTelemetry Collector offers a vendor-agnostic implementation on how to receive, process, and export telemetry data. It removes the need to run, operate, and maintain multiple agents/collectors in order to support open-source observability data formats (e.g. Jaeger, Prometheus, etc.) sending to one or more open-source or commercial back-ends. In addition, the Collector gives end-users control of their data. The Collector is the default location instrumentation libraries export their telemetry data.

This works with improved scalability and supports open-source observability data formats (e.g. Jaeger, Prometheus, Fluent Bit, etc.) sending to one or more open-source or commercial back-ends. The local Collector agent is the default location to which instrumentation libraries export their telemetry data. Open Telemetry binary can be deployed in two primary deployment methods. For production workloads it is recommended to go with a mix of both both methods to make sure traces are not dropped and open telemetry can scale to your traces in the setup.

The two primary deployment methods:

- **Agent**: A Collector instance running with the application or on the same host as the application (e.g. binary, sidecar, or daemonset).
- **Gateway**: One or more Collector instances running as a standalone service (e.g. container or deployment) typically per cluster, data center or region.

We will be deploying a mix of both the agent and the gateway in our setup.

##### Step 2 : OpenTelemetry Agents

We have open telemetry agents deployed as daemonset to recieve and batch ship traces from every EKS worker node. Agent is capable of receiving telemetry data (push and pull based) as well as enhancing telemetry data with metadata such as custom tags or infrastructure information. In addition, the Agent can offload responsibilities that client instrumentation would otherwise need to handle including batching, retry, encryption, compression and more. enrich the traces (offloading work from the app itself) and can help in batch shipping/retrying the push of these traces from application.

The agent can be deployed either as a daemonset or as a sidecar in a kubernetes cluster. This step can be skipped(not reccomended) if you are setting this pipleine in test environment and would rather just send traces straight to the open telemetry collector which is running as a deployment (horizontally scalable)

<img src="img/tracing_step1.png" align="center">

<details>
  <summary>K8s Manifest for OpenTelemetry Agent</summary>

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-agent-conf
  namespace: tracing
  labels:
    app: opentelemetry
    component: otel-agent-conf
data:
  otel-agent-config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    exporters:
      otlp:
        endpoint: "otel-collector.tracing:4317" 
        tls:
          insecure: true
        sending_queue:
          num_consumers: 20
          queue_size: 10000
        retry_on_failure:
          enabled: true
      loadbalancing:
        protocol:
          otlp:
            # all options from the OTLP exporter are supported
            # except the endpoint
            tls:
              insecure: true
            sending_queue:
              num_consumers: 20
              queue_size: 10000
            retry_on_failure:
              enabled: true
        resolver:
          dns:
            hostname: otel-collector.tracing
            port: 4317
    processors:
      resource:
        attributes:
        - key: k8s.cluster.region
          value: "region-name"
          action: insert
        - key: k8s.cluster.name
          value: "cluster-name"
          action: insert
        - key: k8s.cluster.env
          value: "environment-name"
          action: insert
      # The resource detector injects the pod IP
      # to every metric so that the k8sattributes can
      # fetch information afterwards.
      resourcedetection:
        detectors: ["eks"]
        timeout: 5s
        override: true
      memory_limiter:
        check_interval: 1s
        limit_percentage: 50
        spike_limit_percentage: 30
    extensions:
      memory_ballast:
        size_in_percentage: 20
    service:
      pipelines:
        traces/1:
          receivers: [otlp]
          processors: [memory_limiter, batch, resourcedetection, resource]
          exporters: [loadbalancing]
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-agent
  namespace: tracing
  labels:
    app: opentelemetry
    component: otel-agent
spec:
  selector:
    matchLabels:
      app: opentelemetry
      component: otel-agent
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8888"
        prometheus.io/path: "/metrics"
      labels:
        app: opentelemetry
        component: otel-agent
    spec:
      containers:
        - command:
            - "/otelcontribcol"
            - "--config=/conf/otel-agent-config.yaml"
          image: otel/opentelemetry-collector-contrib:0.37.1
          name: otel-agent
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            # This is picked up by the resource detector
            - name: OTEL_RESOURCE
              value: "k8s.pod.ip=$(POD_IP)"
          resources:
            limits:
              cpu: 500m #TODO - adjust this to your own requirements
              memory: 500Mi #TODO - adjust this to your own requirements
            requests:
              cpu: 100m #TODO - adjust this to your own requirements
              memory: 100Mi #TODO - adjust this to your own requirements
          ports:
            - containerPort: 55680 # Default OpenTelemetry receiver port.
              hostPort: 55680
            - containerPort: 4317 # New OpenTelemetry receiver port.
              hostPort: 4317
          volumeMounts:
            - name: otel-agent-config-vol
              mountPath: /conf
          livenessProbe:
            httpGet:
              path: /
              port: 13133 # Health Check extension default port.
          readinessProbe:
            httpGet:
              path: /
              port: 13133 # Health Check extension default port.
      volumes:
        - configMap:
            name: otel-agent-conf
            items:
              - key: otel-agent-config
                path: otel-agent-config.yaml
          name: otel-agent-config-vol
---
```

</details>

##### Step 3 : OpenTelemetry Gateway

The open telemetry agent forward the telemetry data to an open telemetry collector gateway. A Gateway cluster runs as a standalone service and can offer advanced capabilities over the Agent including tail-based sampling, which is how we will be sampling today. In addition, a Gateway cluster can limit the number of egress points required to send data as well as consolidate API token management. Each Collector instance in a Gateway cluster operates independently so it is easy to scale the architecture based on performance needs with a simple load balancer. We deploy gateway as a kubernetes deployment.

For every EKS cluster, we deploy collector as a deployment. K8s deployments are highly elastic, potentially automatically scaling up and down via Horizontal Pod Autoscale which works well with scaling for the number of traces flowing through the pipeline.

<img src="img/tracing_step2.png" align="center">

As you notice we have agent-gateway architecture for processing our distributed traces. Besides high availability, it also allows us to use [Tail Sampling Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/README.md) more organically with this setup. Tail Sampling Processor enables us to make more intelligent choices when it comes to sampling ing traces. This is especially true for latency measurements, which can only be measured after they’re complete. Since the collector sits at the end of the the pipeline and has a complete picture of a distributed trace, sampling determinations are made in open telemetry collectors which decide to sample based on isolated, independent portions of the trace data.

Today, this processor only works with a single instance of the collector. The workaround is to utilize [Trace ID aware load balancing](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/loadbalancingexporter/README.md) to support multiple collector instances as a single instance of collector acts a single point of problem. This load balancer exporter is added to the agents . It is responsible for consistently exporting spans and logs belonging to the same trace to the same backend gateway collector so we can do tail based sampling.

##### Filtering traces for tail based sampling

We will a combination of following filters to sample the traces. The filters for tail sampling are positive selections so if a trace is caught by any of the filter it will be sampled; Alternatively if no filter catches a trace, then it will not be sampled. The filters for tail based sampling can be chained together to get the desired effect:

- latency: Sample based on the duration of the trace. The duration is determined by looking at the earliest start time and latest end time, without taking into consideration what happened in between.
- probabilistic: Sample a percentage of traces.
- status_code: Sample based upon the status code (OK, ERROR or UNSET)
- rate_limiting: Sample based on rate of spans per trace per second
- string_attribute: Sample based on string attributes value matches, both exact and regex value matches are supported

While the top 4 are self explanatory, we will be using the string_attribute filter to "sample all" traces but dropping specific service traces that we do not want sampled. Since there is no blocklist for filters, this is a workaround to sample everything but pick and drop services that you might not want sampled. We can do this by : in the agent adding a span attribute for all traces with a key/value pair (e.g - retain_span/false) using attribute processor. We can add another attribute processor now to override that value to false for specific services that we dont want sampled. Once the trace reach the gateway collector, we can now use the string_atrribute filter on tail based sampling with the key: retain_span and value: true. This would then lead to all traces that do not have retain_span: true not being sampled.

<details>
  <summary>K8s Manifest for OpenTelemetry Collector</summary>

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-conf
  namespace: tracing
  labels:
    app: opentelemetry
    component: otel-collector-conf
data:
  otel-collector-config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    processors:
      # This processor inspects spans looking for matching attributes.
      # Spans that DONT MATCH this will have a new attribute added named retain-span
      # This is then used by the tail_sampling processor to only export spans that
      # have the retain-span attribute.
      # All conditions have to match here to be excluded.
      attributes/filter_spans1:
        exclude:
          match_type: strict
          attributes:
            - {key: "foo", value: "bar"}
        actions:
          - key: retain-span
            action: insert
            value: "true"
      attributes/filter_spans2:
        include:
          match_type: regexp
          regexp:
            # cacheenabled determines whether match results are LRU cached to make subsequent matches faster.
            # Cache size is unlimited unless cachemaxnumentries is also specified.
            cacheenabled: true
          span_names: ["serviceA*"]
        actions:
          - key: retain-span
            action: update
            value: "false"
      # Any policy match will make the trace be sampled !, enable regex didnt caused nothing to match
      tail_sampling:
        decision_wait: 10s
        expected_new_traces_per_sec: 300
        policies:
          [   
            {
              name: policy-retain-span,
              type: string_attribute,
              string_attribute: {key: retain-span, values: ['true']}
            },
            {
            name: rate-limiting-policy,
            type: rate_limiting,
            rate_limiting: {spans_per_second: 35}
            },
            {
            name: probabilistic-sampling,
            type: probabilistic,
            probabilistic: {sampling_percentage: 50}
            }
          ]
      # The k8sattributes in the Agent is in passthrough mode
      # so that it only tags with the minimal info for the
      # collector k8sattributes to complete
      k8sattributes:
        passthrough: true
      memory_limiter:
        check_interval: 1s
        limit_percentage: 50
        spike_limit_percentage: 30
    extensions:
      memory_ballast:
        size_in_percentage: 20
    exporters:
      logging:
        loglevel: info
      otlp/data-prepper:
        endpoint: data-prepper-headless:21890
        tls:
          insecure: true
      jaeger:
        endpoint: "http://jaeger-collector.tracing.svc.cluster.local:14250"
        tls:
          insecure: true
    service:
      pipelines:
        traces/1:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, attributes/filter_spans1, attributes/filter_spans2, tail_sampling]
          exporters: [jaeger, otlp/data-prepper]
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: tracing
  labels:
    app: opentelemetry
    component: otel-collector
spec:
  ports:
    - name: otlp # Default endpoint for OpenTelemetry receiver.
      port: 55680
      protocol: TCP
      targetPort: 55680
    - name: grpc-otlp # New endpoint for OpenTelemetry receiver.
      port: 4317
      protocol: TCP
      targetPort: 4317
    - name: metrics # Default endpoint for querying metrics.
      port: 8888
  selector:
    component: otel-collector
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: tracing
  labels:
    app: opentelemetry
    component: otel-collector
spec:
  selector:
    matchLabels:
      app: opentelemetry
      component: otel-collector
  minReadySeconds: 5
  progressDeadlineSeconds: 120
  replicas: 4 #TODO - adjust this to your own requirements
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8888"
        prometheus.io/path: "/metrics"
      labels:
        app: opentelemetry
        component: otel-collector
    spec:
      containers:
        - command:
            - "/otelcontribcol"
            - "--log-level=debug"
            - "--config=/conf/otel-collector-config.yaml"
          image: otel/opentelemetry-collector-contrib:0.37.1
          name: otel-collector
          resources:
            limits:
              cpu: 2 #TODO - adjust this to your own requirements
              memory: 4Gi #TODO - adjust this to your own requirements
            requests:
              cpu: 1 #TODO - adjust this to your own requirements
              memory: 2Gi #TODO - adjust this to your own requirements
          ports:
            - containerPort: 55680 # Default endpoint for OpenTelemetry receiver.
            - containerPort: 4317 # Default endpoint for OpenTelemetry receiver.
            - containerPort: 8888 # Default endpoint for querying metrics.
          volumeMounts:
            - name: otel-collector-config-vol
              mountPath: /conf
          livenessProbe:
            httpGet:
              path: /
              port: 13133 # Health Check extension default port.
          readinessProbe:
            httpGet:
              path: /
              port: 13133 # Health Check extension default port.
      volumes:
        - configMap:
            name: otel-collector-conf
            items:
              - key: otel-collector-config
                path: otel-collector-config.yaml
          name: otel-collector-config-vol
---
```

</details>

##### Formatting and Exporting Traces

We now have distributed traces created, context propagated and sampled using tail based sampling. We need to format the trace data in a way that our query engine can analyze it. At dow jones we have teams that rely on both Jaeger and Trace Analytics Kibana to query and vizualize this data . With open telemetry we can push the data to multiple backends from the same collector which makes this process pretty simple to achieve. Today, both [Trace Analytics OpenSearch Dashboards plugin](https://opensearch.org/docs/monitoring-plugins/trace/ta-dashboards/) and [Jaeger](https://www.jaegertracing.io/) needs open telemetry data to be transformed to be able to vizualize it. This is why we need a last mile server-side component :

- Jaeger Collector : To transform data for jaeger
- Data Prepper : To transform this data for Trace Analytics OpenSearch Dashboards plugin

##### Sink : OpenSearch 

As you must have noticed in the architecture, we utilize opensearch to ship to and store traces. [Opensearch](https://aws.amazon.com/blogs/opensource/introducing-opensearch/) is a community-driven, open source fork of Elasticsearch and Kibana. This project includes OpenSearch (derived from Elasticsearch 7.10.2) and OpenSearch Dashboards (derived from Kibana 7.10.2). Additionally, the OpenSearch project is the new home for previous distribution of Elasticsearch (Open Distro for Elasticsearch), which includes features such as enterprise security, alerting, machine learning, SQL, index state management, and more. 

Elasticsearch has long been a primary storage backend for Jaeger. Due to its fast search capabilities and horizontal scalability, Elasticsearch makes an excellent choice for storing and searching trace data, along with other observability data. As AWS open sourced data prepperl which recently went 1.2, we were able to use it to ship the distributed traces to open search. 

<img src="img/tracing_step3.png" align="center" >

- **Jaeger Collector** :

[Jaeger is actively working toward the future Jaeger backend components to be based on OpenTelemetry collector](https://www.jaegertracing.io/docs/1.21/opentelemetry/). This integration will make all OpenTelemetry Collector features available in the Jaeger backend components. Till this is experimental, jaeger collector is needed to transform the traces before shipping to opensearch so that jaeger UI is able to consume and vizualize the trace data. Jaeger collector can be deployed as a deployment with a horizontal pod autoscaler attached.

<details>
  <summary>Helm Chart for Jaeger Collector</summary>

```yaml
# All operations of service foo are sampled with probability 0.8 except for operations op1 and op2 which are probabilistically sampled with probabilities 0.2 and 0.4 respectively.
# All operations for service bar are rate-limited at 5 traces per second.
# Any other service will be sampled with probability 1 defined by the default_strategy.

provisionDataStore:
  cassandra: false
storage:
  type: elasticsearch
  elasticsearch:
    scheme: https
    usePassword: false
    host: "opensearch-arn.us-east-1.es.amazonaws.com"
    port: "443"

tag: 1.22.0

agent:
  enabled: false

collector:
  autoscaling:
    enabled: true
    targetMemoryUtilizationPercentage: 80
  service:
  serviceAccount:
    name: jaeger
  samplingConfig: |-
    {
      "service_strategies": [
        {
          "service": "foo",
          "type": "probabilistic",
          "param": 0.8
        },
        {
          "service": "bar",
          "type": "ratelimiting",
          "param": 5
        }
      ],
      "default_strategy": {
        "type": "probabilistic",
        "param": 1.0
      }
    }

query:
  enabled: false
```

</details>

- **Data Prepper** :

Similarly as jaeger, to vizualize disributed traces through Trace Analytics feature in OpenSearch, we need to transform these traces for opensearch (formerly known as AWS managed elasticsearch). Data Prepper is a key component in providing Trace Analytics feature in Opensearch. Data Prepper is a last mile server-side component which collects telemetry data from AWS Distro OpenTelemetry collector or OpenTelemetry collector and transforms it for Elasticsearch. Data Prepper is a data ingestion component of the OpenSearch project that pre-processes documents before storing and indexing in OpenSearch. To pre-process documents, Data Prepper allows you to configure a pipeline that specifies a source, buffers, a series of processors, and sinks. Once you have configured a data pipeline, Data Prepper takes care of managing source, sink, buffer properties, and maintaining state across all instances of Data Prepper on which the pipelines are configured. A single instance of Data Prepper can have one or more pipelines configured. A pipeline definition requires at least a source and sink attribute to be configured, and will use the default buffer and no processor if they are not configured. Data prepper can be deployed as a deployment with a horizontal pod autoscaler attached.

<details>
  <summary>K8s Manifest for Data Prepper</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: tracing
  labels:
    app: data-prepper
  name: data-prepper-config
data:
  pipelines.yaml: |
    entry-pipeline:
      workers : 8
      delay: "100"
      buffer:
        bounded_blocking:
          # buffer_size is the number of ExportTraceRequest from otel-collector the data prepper should hold in memeory. 
          # We recommend to keep the same buffer_size for all pipelines. 
          # Make sure you configure sufficient heap
          # default value is 512
          buffer_size: 4096
          # This is the maximum number of request each worker thread will process within the delay.
          # Default is 8.
          # Make sure buffer_size >= workers * batch_size
          batch_size: 512
      source:
        otel_trace_source:
          health_check_service: true
          ssl: false
      prepper:
        - peer_forwarder:
            discovery_mode: "dns"
            domain_name: "data-prepper-headless"
            ssl: false
      sink:
        - pipeline:
            name: "raw-pipeline"
        - pipeline:
            name: "service-map-pipeline"
    raw-pipeline:
      workers : 8
      buffer:
        bounded_blocking:
          # Configure the same value as in otel-trace-pipeline
          # Make sure you configure sufficient heap
          # default value is 512
          buffer_size: 4096
          # The raw prepper does bulk request to your elasticsearch sink, so configure the batch_size higher.
          # If you use the recommended otel-collector setup each ExportTraceRequest could contain max 50 spans. https://github.com/opendistro-for-elasticsearch/data-prepper/tree/v0.7.x/deployment/aws
          # With 64 as batch size each worker thread could process upto 3200 spans (64 * 50)
          batch_size: 512
      source:
        pipeline:
          name: "entry-pipeline"
      prepper:
        - otel_trace_raw_prepper:
      sink:
        - elasticsearch:
            hosts: 
              - "https://opensearch-arn.us-east-1.es.amazonaws.com"
            insecure: true
            # putting aws_sigv4: false causes auth issues (TBFixed)
            aws_sigv4: false 
            aws_region: "region-name"
            trace_analytics_raw: true
    service-map-pipeline:
      workers : 1
      delay: "100"
      source:
        pipeline:
          name: "entry-pipeline"
      prepper:
        - service_map_stateful:
      buffer:
        bounded_blocking:
          # buffer_size is the number of ExportTraceRequest from otel-collector the data prepper should hold in memeory. 
          # We recommend to keep the same buffer_size for all pipelines. 
          # Make sure you configure sufficient heap
          # default value is 512
          buffer_size: 512
          # This is the maximum number of request each worker thread will process within the delay.
          # Default is 8.
          # Make sure buffer_size >= workers * batch_size
          batch_size: 8
      sink:
        - elasticsearch:
            hosts: 
              - "https://opensearch-arn.us-east-1.es.amazonaws.com"
            insecure: true
            # putting aws_sigv4: false causes auth issues (TBFixed)
            aws_sigv4: false 
            aws_region: "region-name"
            trace_analytics_service_map: true
  data-prepper-config.yaml: |
    ssl: false
---
apiVersion: v1
kind: Service
metadata:
  namespace: tracing
  labels:
    app: data-prepper
  name: data-prepper-headless
spec:
  clusterIP: None
  ports:
    - name: "21890"
      port: 21890
      targetPort: 21890
  selector:
    app: data-prepper
---
apiVersion: v1
kind: Service
metadata:
  namespace: tracing
  labels:
    app: data-prepper
  name: data-prepper-metrics
spec:
  type: NodePort
  ports:
    - name: "4900"
      port: 4900
      targetPort: 4900
  selector:
    app: data-prepper
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: tracing
  labels:
    app: data-prepper
  name: data-prepper
spec:
  replicas: 4
  selector:
    matchLabels:
      app: data-prepper
  strategy:
    type: Recreate
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "4900"
        prometheus.io/path: "/metrics"
        sidecar.istio.io/inject: "false"
      labels:
        app: data-prepper
    spec:
      containers:
        - args:
            - java
            - -jar
            - /usr/share/data-prepper/data-prepper.jar
            - /etc/data-prepper/pipelines.yaml
            - /etc/data-prepper/data-prepper-config.yaml
            - -Dlog4j.configurationFile=config/log4j2.properties
          image: amazon/opendistro-for-elasticsearch-data-prepper:1.0.0
          imagePullPolicy: IfNotPresent
          name: data-prepper
          resources:
            limits:
              cpu: 1
              memory: 2Gi
            requests:
              cpu: 200m
              memory: 400Mi
          ports:
            - containerPort: 21890
          volumeMounts:
            - mountPath: /etc/data-prepper
              name: prepper-configmap-claim0
            - mountPath: config
              name: prepper-log4j2
      restartPolicy: Always
      volumes:
        - name: prepper-configmap-claim0
          configMap:
            name: data-prepper-config
        - name: prepper-log4j2
          configMap:
            name: data-prepper-log4j2
---
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: tracing
  labels:
    app: data-prepper
  name: data-prepper-log4j2
data:
  log4j2.properties: |
    status = error
    dest = err
    name = PropertiesConfig

    property.filename = log/data-prepper/data-prepper.log

    appender.console.type = Console
    appender.console.name = STDOUT
    appender.console.layout.type = PatternLayout
    appender.console.layout.pattern = %d{ISO8601} [%t] %-5p %40C - %m%n

    rootLogger.level = warn
    rootLogger.appenderRef.stdout.ref = STDOUT

    logger.pipeline.name = com.amazon.dataprepper.pipeline
    logger.pipeline.level = info

    logger.parser.name = com.amazon.dataprepper.parser
    logger.parser.level = info

    logger.plugins.name = com.amazon.dataprepper.plugins
    logger.plugins.level = info
---
```

</details>

Jaeger collector and data prepper can be configured to ship the distributed traces to the same opensearch. Both deployment create their own index with a mutually exclusive prefix for index name. The opensearch can be configured with index patterns to create two views, one for jaeger traces and the other for otel traces. Trace analytics and Jaeger UI(Query component) are auto configured to reach their own indexes and do not collide with each other. This results in a cheaper setup as it allows for a single open search with multiple data formats instead of spreading them on different opensearches. 

##### Step 4 : Vizualization

Once you have reach this point, the difficult part is over. You should have distributed traces created, context propagated and sampled using tail based sampling, transformed f and exported to opensearch under different index patterns.

Now is the part where you actually get to use the pipeline and vizualize these traces to run queries, build dashboard, setup alerting. While Trace Analytics OpenSearch Dashboards plugin is a managed kibana plugin that comes with open search, we host our own [Jaeger Frontend/UI](https://www.jaegertracing.io/docs/1.28/frontend-ui/) which provides a simple jaeger UI to query for distributed traces with Traces View
and Traces Detail View. We deployer Jaeger Query as a deployment within the EKS cluster. Jaeger query can be deployed as a deployment with a horizontal pod autoscaler attached

<img src="img/tracing_step4.png" align="center" >

- **Jaeger UI(query)**

To vizualize your traces with jaeger we need to run a jaeger query. jaeger-query serves the API endpoints and a React/Javascript UI. The service is stateless and is typically run behind a load balancer :

<details>
  <summary>Helm Chart for Jaeger Query</summary>

```yaml
# All operations of service foo are sampled with probability 0.8 except for operations op1 and op2 which are probabilistically sampled with probabilities 0.2 and 0.4 respectively.
# All operations for service bar are rate-limited at 5 traces per second.
# Any other service will be sampled with probability 1 defined by the default_strategy.

provisionDataStore:
  cassandra: false
storage:
  type: elasticsearch
  elasticsearch:
    scheme: https
    usePassword: false
    host: "opensearch-arn.us-east-1.es.amazonaws.com"
    port: "443"

tag: 1.22.0

agent:
  enabled: false

collector:
  enabled: false

query:
  enabled: true
  agentSidecar:
    enabled: false
  service:
    type: LoadBalancer
    port: 443
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-ssl-cert: ""
      service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0
  config: |-
    {
      "dependencies": {
        "dagMaxNumServices": 200,
        "menuEnabled": true
      },
      "menu": [
    {
      "label": "Open Telemetry Resources",
      "items": [
        {
          "label": "Open Telemetry Client Instrumentation",
          "url": "https://opentelemetry.io/docs/"
        }
          ]
        }
      ],
      "archiveEnabled": true,
      "search": {
        "maxLookback": {
          "label": "7 Days",
          "value": "7d"
        },
        "maxLimit": 1500
      }
    }

spark:
  enabled: false

esIndexCleaner:
  enabled: false

esRollover:
  enabled: false

esLookback:
  enabled: false
```

</details>

- **Trace Analytics feature in OpenSearch/Kibana**

[Amazon Elasticsearch Service released Trace Analytics](https://aws.amazon.com/blogs/big-data/getting-started-with-trace-analytics-in-amazon-elasticsearch-service/), a new feature for distributed tracing that enables developers and operators to troubleshoot performance and availability issues in their distributed applications, giving them end-to-end insights not possible with traditional methods of collecting logs and metrics from each component and service individually. Trace Analytics supports OpenTelemetry, enabling customers to leverage Trace Analytics without having to re-instrument they applications.

Once you have traces flowing through your pipeline, you should see indexes being created in your open search under the otel-v1* index prefix. Once this is created, trace analytics plugin within opensearch kibana should now have vizuals for your traces including the service map, error/throughput graphs and waterfall trace view for your services. 

<img src="img/kibana.png" align="center" height="50%"  width="50%">

#### Testing

We now have the distributed tracing pipeline, it’s time to start an application that generates traces and sends it the our agents. Depending on your application code and whether you want to do manual or automatic instrumentation, the [Open Telementry Instrumentation Docs](https://opentelemetry.io/docs/) should get you started.

Since in our setup Otel agents are running as daemonset, we will be sending the traces to the agent on the same host(worker-node). To get the host IP, we can set the HOST_IP environment variable via the [Kubernetes downwards API](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api). and then reference it in our instrumentation as the destination address.

Make sure you are injecting the HOST_IP environment variable in your application manifest :

```
env:
  - name: HOST_IP
    valueFrom:
      fieldRef:
          fieldPath: status.hostIP
```

Once you have the application running and firing traces, you should be able to start firing some traces. To setup the pipeline, you can apply the kubernetes manifest files and helm chart(for jaeger) which is published and provided [here](source)

#### Next Steps

We built a scalable distributed tracing pipeline based on open telemetry running in our EKS cluster. As you would have realized(if you use open telemetry), one of the most important reasons to move to open telemetry is the fact that a single binary can be used to ingest, process and export not just traces but metrics and logs as well (telemetry data). We can collect and send telemetry data to multiple backend platform by creating [pipelines](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/design.md#pipelines) and using exporters to send that data to the observability platform that you choose. 

Data Prepper 1.2 (December 2021) release is going to provide users the ability to send logs from Fluent Bit to OpenSearch or Amazon OpenSearch Service and use Grok to enhance the logs. These logs can then be correlated to traces coming from the OTEL Collectors to further enhance deep diving into your service problems using OpenSearch Dashboards. This experience should make deep diving into your service much more simple with a unified experience and give a really powerful feature into the hands of developer and operators. 

So the next step would naturally be to extend this pipeline to be more than just a distributed tracing pipeline and morph to a distributed telemetry pipeline ! 

*Comments, questions, corrections? Just let me know on [twitter](https://twitter.com/kub3rkaul), via email kuber.kaul@dowjones.com, or submit a github issue*
