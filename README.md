# Introduction to Metrics, Logs, Traces and Profiles in Grafana

This was originally the  companion repository to a series of presentations over the [three pillars of
observability within Grafana](https://grafana.com/blog/2022/04/01/get-started-with-metrics-logs-and-traces-in-our-new-grafana-labs-asia-pacific-webinar-series/). Whilst that series is now over a year old, we have kept this repository up-to-date with the latest versions of our products and added more functionality as our products have grown.

It is presented as a self-enclosed Docker sandbox that includes all of the components required to run on a local machine and experiment with the products provided.

Since the original series, this repository has seen its use grow. Whilst we still highly recommend everyone to sign up for a Grafana Cloud account, this repository exists as an easy way to get started with Grafana's offerings in a non-cloud, local-based setup. In addition, you can use this setup to experiment with configuration settings on those offerings.

You can also send data from the example microservice application to Grafana Cloud products.

## Prerequisites

The following demonstration environment requires:
* [Docker](https://www.docker.com/products/docker-desktop/)
* [Docker Compose (if not using a version of Docker that has it inbuilt)](https://docs.docker.com/compose/install/)

## Overview
The demos from this series were based on the application and code in this repository, which includes:

* Docker Compose manifest for easy setup.
* Four-service application:
  * A service requesting data from a REST API server.
  * A REST API server that receives requests and utilises a Database for storing/retrieving data for those requests.
  * A recorder service for storing messages to an AMQP bus.
  * A Postgres Database for storing/retrieving data from.
* k6 service running a load test against the above application.
* Tempo service for storing and querying trace information.
* Loki service for storing and querying log information.
* Mimir service for storing and querying metric information.
* Pyroscope service for storing and querying profiling information.
* Grafana service for visualising observability data.
* Grafana Agent service for receiving traces and producing metrics and logs based on these traces.
* A Node Exporter service to retrieve resource metrics from the local host.

## Run local otel grafana stack

```
docker compose -f .\docker-compose-otel.yml up -d
```

## Running the Demonstration Environment

Docker Compose will download the required Docker images, before starting the demonstration environment.

In the following examples, the in-built `compose` command is used with a latest version of Docker (for example, `docker compose up`). If using an older version of Docker with a separate Docker Compose binary, ensure that `docker compose` is replaced with `docker-compose`.

Data will be emitted from the microservice application and be stored in Loki, Tempo, and Prometheus. You can login to the Grafana service to visualize this data.

To execute the environment and login:

1. Start a new command-line interface in your Operating System and run:
   ```bash
   docker compose up
   ```
2. Login to the local Grafana service at http://localhost:3000/.

   *NOTE:* This assumes that port 3000 is not already in use. If this port is not free, edit the `docker-compose.yml` file and alter the line
   ```
   - "3000:3000"
   ```
   to some other host port that is free, for example:
   ```
   - "3123:3000"
   ```
3. Navigate to the [MLT dashboard](http://localhost:3000/d/4VSk5Lank/mlt-dashboard?orgId=1&refresh=5s).
4. Explore the data sources using the [Grafana Explorer](http://localhost:3000/explore?orgId=1&left=%7B%22datasource%22:%22Mimir%22,%22queries%22:%5B%7B%22refId%22:%22A%22%7D%5D,%22range%22:%7B%22from%22:%22now-1h%22,%22to%22:%22now%22%7D%7D).

The [pre-provisioned dashboard](grafana/definitions/mlt.json) demonstrates a [RED (Rate, Error, Duration)](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/) overview of the microservice application, where almost all metrics are being generated via trace spans. The dashboard also provides an example of logging.

[Data links](https://grafana.com/docs/grafana/latest/panels-visualizations/configure-data-links/), [exemplars](https://grafana.com/docs/grafana-cloud/data-configuration/traces/exemplars/), and logs are utilized to allow jumping from the dashboard to a Grafana Explore page to observe traces, metrics, and logs in more detail.

The following sections are a brief explanation of each of the most important provided components.

### Using the OpenTelemetry Collector

You can swap out the Grafana Agent for the OpenTelemetry collector using an alternative configuration.

Read the 'OpenTelemetry Collector' section below to use this environment instead.

## Grafana

Grafana is a multi-platform open source analytics and interactive visualisation web application. For more details about Grafana, read the [documentation](https://grafana.com/docs/grafana/latest/).

The Grafana service is described in the `grafana` section of the [`docker-compose.yml`](docker-compose.yml) manifest.

The Docker Compose manifest:
* Mounts two repository directories to provide pre-provisioned data sources for data (`grafana/definitions`, `grafana/provisioning`).
* A [pre-provisioned dashboard](grafana/definitions/mlt.json) for correlating metrics, logs and traces.
  * The dashboard uses metrics from span traces to provide RED (Rate/Error/Duration) signals.
  * Data links are built into Grafana panels to pre-populate TraceQL queries based on the data. These act as an initial guide for digging into more detailed trace queries by extending the TraceQL.
* Exposes port `3000` for local login.
* Enables two Tempo features, namely span search and service graph support.

The updated `topnav` navigation within Grafana is enabled. If you wish to default back to the old UI, remove the `topnav` feature flag in the `GF_FEATURE_TOGGLES_ENABLE` environment variable for the `grafana` service in the [`docker-compose.yml`](docker-compose.yml) manifest.

## Mimir

Mimir is a backend store for metrics data from various sources. For more details about Mimir, read the [documentation](https://grafana.com/docs/mimir/latest/).

The Mimir service is described in the `mimir` section of the [`docker-compose.yml`](docker-compose.yml) manifest.

The configuration file ([`mimir/mimir.yml`](mimir/mimir.yml)):
* Configures a single service container acting as all relevant microservices.
* Stores the metrics data in-container (this will be lost on container deletion).

In addition to the scraped metrics, the Mimir service also receives remotely written metrics from the Tempo service, which derives metrics from incoming trace spans.

[This example](http://localhost:3000/explore?left=%7B%22datasource%22:%22mimir%22,%22queries%22:%5B%7B%22datasource%22:%7B%22type%22:%22prometheus%22,%22uid%22:%22mimir%22%7D,%22exemplar%22:true,%22expr%22:%22histogram_quantile%280.95,%20sum%28rate%28mythical_request_times_bucket%5B15s%5D%29%29%20by%20%28le,%20beast%29%29%22,%22interval%22:%22%22,%22refId%22:%22A%22%7D%5D,%22range%22:%7B%22from%22:%22now-5m%22,%22to%22:%22now%22%7D%7D&orgId=1) of the Mimir data source shows a histogram with [exemplars](https://grafana.com/docs/grafana-cloud/data-configuration/traces/exemplars/) (links to relevant traces). The example is available once the system is running and has collected enough data.

## Loki

Loki is a backend store for long-term log retention. For more details about Loki, read the [documentation](https://grafana.com/docs/loki/latest/).

The Loki service is described in the `loki` section of the [`docker-compose.yml`](docker-compose.yml) manifest.

Loki's default configuration is used, and as such no custom configuration file is bound to the container (unlike Mimir and Tempo).

[This example](http://localhost:3000/explore?left=%7B%22datasource%22:%22loki%22,%22queries%22:%5B%7B%22refId%22:%22A%22,%22expr%22:%22%7Bjob%3D%5C%22mythical-beasts-requester%5C%22%7D%20%7C%20logfmt%20%7C%20http_method%20%3D%20%5C%22POST%5C%22%20and%20__error__%3D%5C%22%5C%22%22,%22queryType%22:%22range%22,%22datasource%22:%7B%22type%22:%22loki%22,%22uid%22:%22loki%22%7D,%22editorMode%22:%22code%22,%22range%22:true,%22instant%22:true%7D%5D,%22range%22:%7B%22from%22:%22now-5m%22,%22to%22:%22now%22%7D%7D&orgId=1) shows the Loki data source using LogQL.

The microservices application sends its logs directly to the Loki service in this environment, via its REST API. There is the potential to switch this to the [Loki Docker driver](https://grafana.com/docs/loki/latest/clients/docker-driver/), if desired. To do so, follow the instructions for the driver, and then remove the three occurences of:
```
- LOGS_TARGET=http://loki:3100/loki/api/v1/push
```
in the [`docker-compose.yml`](docker-compose.yml) manifest for the `mythical-receiver`, `mythical-server` and `mythical-recorder` services. This will instead force the microservices to output logs to `stdout` which will be picked up by the Loki Docker driver.

## Tempo

Tempo is a backend store for longterm trace retention. For more details about Tempo, read the [documentation](https://grafana.com/docs/tempo/latest/).

The Tempo service is described in the `tempo` section of the [`docker-compose.yml`](docker-compose.yml) manifest.

The Tempo service imports a configuration file ([`tempo/tempo.yaml`](tempo/tempo.yaml)) that initialises the service with some sensible defaults as well as allowing the receiving of traces in a variety of different formats.

Tempo is also configured to generate metrics from incoming trace spans as part of it's configuration. As such, this no longer occurs via Grafana Agent (although the original configuration for the Agent to carry this out has been left in the Agent configuration file as a guide).

For an example of a simple search, look at the Explorer page using the Tempo data source, [here](http://localhost:3000/explore?orgId=1&left=%7B%22datasource%22:%22tempo%22,%22queries%22:%5B%7B%22refId%22:%22A%22,%22datasource%22:%7B%22type%22:%22tempo%22,%22uid%22:%22tempo%22%7D,%22queryType%22:%22traceqlSearch%22,%22limit%22:20,%22filters%22:%5B%7B%22id%22:%224ad1d67f%22,%22operator%22:%22%3D%22,%22scope%22:%22span%22%7D,%7B%22id%22:%22service-name%22,%22tag%22:%22service.name%22,%22operator%22:%22%3D%22,%22scope%22:%22resource%22,%22value%22:%5B%22mythical-server%22%5D,%22valueType%22:%22string%22%7D,%7B%22id%22:%22min-duration%22,%22tag%22:%22duration%22,%22operator%22:%22%3E%22,%22valueType%22:%22duration%22,%22value%22:%22100ms%22%7D%5D%7D%5D,%22range%22:%7B%22from%22:%22now-1h%22,%22to%22:%22now%22%7D%7D). **Note:** Native searches no longer exist, and these are interpretted as TraceQL before execution. See the bottom of the search panel to show the equivalent TraceQL

[This example](http://localhost:3000/explore?orgId=1&left=%7B%22datasource%22:%22tempo%22,%22queries%22:%5B%7B%22refId%22:%22A%22,%22datasource%22:%7B%22type%22:%22tempo%22,%22uid%22:%22tempo%22%7D,%22queryType%22:%22traceql%22,%22serviceName%22:%22mythical-requester%22,%22minDuration%22:%22100ms%22,%22limit%22:20,%22query%22:%22%7B%20.service.name%20%3D%20%5C%22mythical-server%5C%22%20%26%26%20duration%20%3E%20100ms%20%7D%22%7D%5D,%22range%22:%7B%22from%22:%22now-1h%22,%22to%22:%22now%22%7D%7D) uses the same parameters as above, but in TraceQL (a fully featured tracing query language).

For an example of the mini-APM table and Service Graphs, use the 'Service Graph' tab [here](http://localhost:3000/explore?orgId=1&left=%7B%22datasource%22:%22tempo%22,%22queries%22:%5B%7B%22refId%22:%22A%22,%22datasource%22:%7B%22type%22:%22tempo%22,%22uid%22:%22tempo%22%7D,%22queryType%22:%22serviceMap%22%7D%5D,%22range%22:%7B%22from%22:%22now-1h%22,%22to%22:%22now%22%7D%7D).

Traces are instrumented using the OpenTelemetry SDK, more details on which can be found [here](https://opentelemetry.io/docs/).
## Pyroscope

Pyroscope is a continuous profiling backend store.

The Pyroscope service is described in the `pyroscope` section of the [`docker-compose.yml`](docker-compose.yml) manifest.

Pyroscope uses a configuration file ([`pyroscope/pyroscope.yaml`](pyroscope/pyroscope.yaml)) that is configured to scrape [pprof](https://github.com/google/pprof) based profiles from the Mythical microservices. It uses the [Pyroscope NodeJS](https://github.com/grafana/pyroscope-nodejs) bindings in source instrumentation.

Samples are scraped directly from the application on the `/debug/pprof/profile` and `/debug/pprof/heap` endpoints.

You can see an example of profiling in action once the system is running by using the Explorer to visualise the profiles stored [here](http://localhost:3000/explore?orgId=1&left=%7B%22datasource%22:%22pyroscope%22,%22queries%22:%5B%7B%22refId%22:%22A%22,%22datasource%22:%7B%22type%22:%22phlare%22,%22uid%22:%22pyroscope%22%7D,%22groupBy%22:%5B%5D,%22labelSelector%22:%22%7B%7D%22,%22queryType%22:%22both%22,%22profileTypeId%22:%22mythical-beasts-server.cpu%22%7D%5D,%22range%22:%7B%22from%22:%22now-1h%22,%22to%22:%22now%22%7D%7D).

## k6

k6 is a load testing suite that allows you to synthetically load and monitor your application. For more details about k6, read the [documentation](https://k6.io/docs/).

The k6 service is described in the `k6` section of the [`docker-compose.yml`](docker-compose.yml) manifest.

The k6 service uses the script ([`k6/mythical-loadtest.js`](k6/mythical-loadtest.js)) to define the tests that it should run. These are currently a `GET`, `POST` and `DELETE` set of tests on the application's API endpoints.

k6 can run one of more VU (Virtual Users) concurrently, to simulate parallel load on the application. Currently, the number of VUs is set to 1, although this may be changed by altering the value for the `K6_VUS` environment variable in the relevant Docker Compose YAML file.
**Note:** The higher the number of VUs executing, the higher the load on the machine running the Docker Compose sandbox, as this will transfer a significant amount of data. You may find tests being throttled if you ramp this number up without enough resource/bandwidth.

k6 will generate [metrics](https://k6.io/docs/using-k6/metrics/) about the tests that it carries out, and will send these to the running Mimir instance. These metrics can then be used to determine the latencies of endpoints, number of errors occuring, etc. The official Grafana dashboard for k6 is included, and once the sandbox is running, may be found [here](http://localhost:3000/d/01npcT44k/official-k6-test-result?orgId=1&refresh=10s).

## Grafana Agent

**Note:** The intention is to move this repository over to Flow/River as the default configuration in early September, to fall in line with Grafana's stance on Flow also becoming the default for Agent configuration.

Grafana Agent is a configurable local agent for receiving metrics, logs and traces and forwarding them to relevant database stores. For more details about Grafana Agent, read the [documentation](https://grafana.com/docs/agent/latest/).

Grafana Agent is a locally installed agent that acts as:
* A Prometheus scraping service and metric/label rewriter.
* A Promtail (Loki logs receiver) service and processor.
* A Tempo trace receiver and span processor.
* Remote writer for MLT data to Grafana Cloud (or any other compatible storage system).

In this example environment, Grafana Agent:
* Receives metrics data, via scrape configs, emitted by:
  * The microservice application.
  * The Mimir service for operational monitoring.
  * The Loki service for operational monitoring.
  * The Tempo service for operatational monitoring.
  * The Agent itself, for operational monitoring.
  * The installed Node Exporter service.
* Receives trace data, via trace configs, emitted by the microservice application.
* Generates automatic logging lines based on the trace data received.
* Sends metric, log and trace data onwards to the Mimir, Loki and Tempo services, respectively.

Note that as Grafana Agent scrapes metrics for every service defined in the [`docker-compose.yml`](docker-compose.yml) that a significant number of metric [active series](https://grafana.com/docs/grafana-cloud/billing-and-usage/active-series-and-dpm/) are produced (approximately 11,000 at time of writing).

The full configuration for Grafana Agent can be found [here](agent/config.yaml).

### Metrics Generation

It should be noted that since [v1.4.0](https://github.com/grafana/tempo/blob/main/CHANGELOG.md#v140--2022-04-28), Tempo has included the ability to generate [RED (Rate, Error, Duration)](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/) [span](https://grafana.com/docs/tempo/latest/metrics-generator/span_metrics/) and [service graph](https://grafana.com/docs/tempo/latest/metrics-generator/service_graphs/) metrics.

As such, the Grafana Agent configuration now includes a commented section where those metrics used to be generated; this is now handled directly in Tempo via server-side metrics generation.

Whilst this is convenient for many users, you may prefer to generate metrics locally via Grafana Agent rather than Tempo server-side. These include environments where tail-based sampling may be utilized to discard certain traces.

Tempo metrics generation will only generate span and service graph metrics for trace spans that Tempo receives. If tail sampling is active, then a full view of the metrics in a system will not be available.

In these instances, using Grafana Agent to generate metrics can ensure a complete set of metrics for all traces span data are generated, as the Agent carries out tail sampling post-metrics generation.

### Flow and River Configuration

Whilst the default configuration is via the 'traditional' Grafana Agent configuration YAML file, you can switch this to a provided [Flow](https://grafana.com/docs/agent/latest/flow/) configuration defined in Grafana's configuration language, River.

River provides a more programmatic way of defining Grafana Agent functionality, and, whilst currently in Beta, is rapidly gaining parity with the current YAML configuration feature set.

Grafana Agent already supports a large number of components.

To use the Flow configuration instead, comment out the following lines in the `agent` service definition in the [`docker-compose.yml`](docker-compose.yml) file:

```
      - "./agent/config.yaml:/etc/agent/agent.yaml"
```

```
    command: [
      "-config.file=/etc/agent/agent.yaml",
      "-server.http.address=0.0.0.0:12345",
    ]
```


And then uncomment the following lines:
```
      #- "./agent/config.river:/etc/agent/config.river"
```

```
    #environment:
    #  - AGENT_MODE=flow
    #command: [
    #  "run",
    #  "--server.http.listen-addr=0.0.0.0:12345",
    #  "/etc/agent/config.river",
    #]
```

The new changes can be used by restarting Docker Compose if it is currently running:
```bash
docker compose restart
```

Or using the startup commands in the 'Running the Demonstration Environment' section.

Once running, you can observe the Flow configuration running on the Grafana Agent itself by navigating to [http://localhost:12347](http://localhost:12347).

This webpage will allow you to view all of the current components being used for receiving MLT signals,
as well as graphs denoting source and target relationships between components.

Read the [Debugging](https://grafana.com/docs/agent/latest/flow/monitoring/debugging/) documentation for Grafana Agent for more details.

The [tutorial](https://grafana.com/docs/agent/latest/flow/tutorials/) guide to working with Flow and River is a great first starting point, whilst the full [reference guide](https://grafana.com/docs/agent/latest/flow/reference/) for Flow shows the currently supported components and configuration blocks.
## Microservice Source

The source for the microservice application can be found in the [`source`](source) directory. This three-service application utilizes a [PostgreSQL](https://www.postgresql.org/) database and an [AMQP](https://www.amqp.org/) bus to store data.

The services are written in [JavaScript](https://www.javascript.com/) and execute under [NodeJS](https://nodejs.org/en) inside [Docker](https://www.docker.com/products/docker-desktop/) containers.

The [`requester`](source/mythical-beasts-requester/index.js) service makes 'random' requests to the [`server`](source/mythical-beasts-server/index.js), which then inserts, retrieves or deletes data from the Postgres database. The `requester` service also stores data to the AMQP queue via the [`recorder`](source/mythical-beasts-recorder/index.js) service.

All three services use common code to deal with the [`queue`](source/common/queue.js), [`logging`](source/common/logging.js) and [`tracing`](source/common/tracing.js) requirements they have. The latter is an example of a simple shim API library for utilising the OpenTelemetry SDK in an application.

There is a common [`Dockerfile`](source/docker/Dockerfile) that is used to build all three services.

## Grafana Cloud

>**Note**: By default, as mentioned in the Grafana Agent section, metrics are scraped by default from every service. If sending metrics to Grafana Cloud, check the number of metric [active series](https://grafana.com/docs/grafana-cloud/billing-and-usage/active-series-and-dpm/) that you can store without additional cost.

This demo can be run against Grafana Cloud using Docker Compose as follows:

1. Configure and source the Metrics, Logs, and Traces environment variables in the file [`cloud/envvars-grafana-cloud-unconfigured.sh`](cloud/envvars-grafana-cloud-unconfigured.sh) from a Grafana Cloud service. Generally, only the environment variables tagged with “__ID__” and “__API_KEY__” need to be updated, sometimes the _HOST variables will need to be modified. You can retrieve these details from your Grafana Cloud organization pages for the relevant stack.
   ```bash
   source cloud/envvars-grafana-cloud-unconfigured.sh
   ```
2. Configure the environment
   ```bash
   ./ctl.sh cloud-configure
   ```
3. Run the demo using Docker Compose
   ```bash
   ./ctl.sh cloud-up
   ```
4. Load the example dashboard `cloud/dashboard-metrics-logs-traces-1.json` into the Grafana Cloud Instance to visualize the metrics, logs and traces being generated by this demo

5. Stop the demo using
   ```bash
   ./ctl.sh cloud-down
   ```

>**Note:** The configuration for Grafana Agent to send data to Grafana Cloud includes configuration that generates span metrics and service graph metrics.

## OpenTelemetry Collector

You can also use an alternative environment that uses the OpenTelemetry Collector in place of Grafana Agent.

### Running the Demonstration Environment

Docker Compose downloads the required Docker images, before starting the demonstration environment.

In the following examples, the in-built `compose` command is used with a latest version of Docker (for example, `docker compose up`). If using an older version of Docker with a separate Docker Compose binary, ensure that `docker compose` is replaced with `docker-compose`.

Data is emitted from the microservice application and stored in Loki, Tempo, and Prometheus. You can login to the Grafana service to visualize this data.

To execute the environment and login:

1. Start a new command-line interface in your Operating System and run:
   ```bash
   docker compose -f docker-compose-otel.yml up
   ```
2. Login to the local Grafana service at http://localhost:3000/.

   *NOTE:* This assumes that port 3000 is not already in use. If this port is not free, edit the `docker-compose.yml` file and alter the line
   ```
   - "3000:3000"
   ```
   to some other host port that is free, for example:
   ```
   - "3123:3000"
   ```
3. Explore the data sources using the [Grafana Explorer](http://localhost:3000/explore?orgId=1&left=%7B%22datasource%22:%22Mimir%22,%22queries%22:%5B%7B%22refId%22:%22A%22%7D%5D,%22range%22:%7B%22from%22:%22now-1h%22,%22to%22:%22now%22%7D%7D).

The OpenTelemetry Collector is defined as the `opentelemetry-collector` service in the [`docker-compose-otel.yml`](docker-compose-otel.yml) manifest.

A basic configuration that mimics that of the Grafana Agent configuration can be found in the [`otel/otel.yml`](otel/otel.yml) configuration file.

In much the same way that the Grafana Agent configuration operates, this scrapes several targets to retrieve Prometheus metrics before batching them and remote writing them to the local Mimir service.

Additionally, the OpenTelemetry Collector receives traces via OTLP gRPC, batches them, and then remote writes them to the local Tempo instance.

Span metrics and service metrics are also available, but have been commented out from the configuration file as generation is handled in Tempo by default. You may uncomment these configuration sections, whilst commenting out the metrics generation in [`tempo/tempo.yaml`](tempo/tempo.yaml) to generate metrics from within the OpenTelemetry Collector.

>**Note:** Span metrics generation in the OpenTelemetry Collector emits metrics in a different nomenclature to that of Grafana Agent and Tempo. The metrics are emitted without the `traces_spanmetrics` prefix (eg. `calls_total` instead of `traces_spanmetrics_calls_total`). As such, the included provisioned dashboard will not show data as the PromQL queries expect the Grafana Agent/Tempo nomenclature. It is trivial to alter the queries however. Additionally, automatic logging is not available in the OpenTelemetry Collector. Service graph generation uses the same nomenclature, and should work as expected.
