The [Thanos](https://www.katacoda.com/thanos/courses/thanos/thanos.io) project defines a set of components that can be composed together into a highly available metric system with **unlimited storage capacity** that **seamlessly** integrates into your existing Prometheus deployments.

In this course you get first-hand experience building and deploying this infrastructure yourself.

In this tutorial, you will learn:

- How to ingest metrics data from Prometheus instances without ingress traffic and need to store data locally for longer time.
- How to setup a Thanos Querier to access this data.
- How Thanos Receive is different from Thanos Sidecar, and when is the right time to use each of them.

> NOTE: This course uses docker containers with pre-built Thanos, Prometheus, and Minio Docker images available publicly.

#### Problem Statement & Setup

# Problem Statement & Setup

## Problem Statement

Let's imagine that you run a company called `Wayne Enterprises`. This company runs two clusters: `Batcave` & `Batcomputer`. Each of these sites runs an instance of Prometheus that collects metrics data from applications and services running there.

However, these sites are special. For security reasons, they do not expose public endpoints to the Prometheus instances running there, and so cannot be accessed directly from other parts of your infrastructure.

As the person responsible for implementing monitoring these sites, you have two requirements to meet:

1. Implement a global view of this data. `Wayne Enterprises` needs to know what is happening in all parts of the company - including secret ones!
2. Global view must be queryable in near real-time. We can't afford any delay in monitoring these locations!

Firstly, let us setup two Prometheus instances...

## Setup

### Batcave

Let's use a very simple configuration file, that tells prometheus to scrape its own metrics page every 5 seconds.

```shell
cat > /vagrant/prometheus/conf/prometheus-batcave.yaml << EOF
global:
  scrape_interval: 5s
  external_labels:
    cluster: batcave
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']
EOF
```

Run the prometheus instance:

```
docker run -d --net=host \
    -v /vagrant/prometheus/conf/prometheus-batcave.yaml:/etc/prometheus/prometheus.yaml \
    -v /data/prometheus-batcave-data:/prometheus \
    -u root \
    --name prometheus-batcave \
    quay.io/prometheus/prometheus:v2.27.0 \
    --config.file=/etc/prometheus/prometheus.yaml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9090 \
    --web.external-url=http://192.168.56.100:9090 \
    --web.enable-lifecycle
```

Verify that `prometheus-batcave` is running by navigating to the [Batcave Prometheus UI](http://192.168.56.100:9090).

Why do we enable the web lifecycle flag?

By specifying `--web.enable-lifecycle`, we tell Prometheus to expose the `/-/reload` HTTP endpoint. This lets us tell Prometheus to dynamically reload its configuration, which will be useful later in this tutorial.

### Batcomputer

Almost exactly the same configuration as above, except we run the Prometheus instance on port `9091`.

```shell
cat > /vagrant/prometheus/conf/prometheus-batcomputer.yaml << EOF
global:
  scrape_interval: 5s
  external_labels:
    cluster: batcomputer
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9091']
EOF

docker run -d --net=host \
    -v /vagrant/prometheus/conf/prometheus-batcomputer.yaml:/etc/prometheus/prometheus.yaml \
    -v /data/prometheus-batcomputer:/prometheus \
    -u root \
    --name prometheus-batcomputer \
    quay.io/prometheus/prometheus:v2.27.0 \
    --config.file=/etc/prometheus/prometheus.yaml \
    --storage.tsdb.path=/prometheus \
    --web.listen-address=:9091 \
    --web.external-url=http://192.168.56.100:9091 \
    --web.enable-lifecycle
```

Verify that `prometheus-batcomputer` is running by navigating to the [Batcomputer Prometheus UI](/).

With these Prometheus instances configured and running, we can now start to architect our global view of all of `Wayne Enterprises`.

#### Setup Thanos Receive

# Setup Thanos Receive

With `prometheus-batcave` & `prometheus-batcomputer` now running, we need to think about how we satisfy our two requirements:

1. Implement a global view of this data.
2. Global view must be queryable in near real-time.

How are we going to do this?

## Thanos Sidecar

After completing [Tutorial #1: Global View](https://www.katacoda.com/thanos/courses/thanos/1-globalview), you may think of running the following architecture:

- Run Thanos Sidecar next to each of the Prometheus instances.
- Configure Thanos Sidecar to upload Prometheus data to an object storage (S3, GCS, Minio).
- Run Thanos Store connected to the data stored in object storage.
- Run Thanos Query to pull data from Thanos Store.

However! This setup **does not** satisfy our requirements above.

<details style="box-sizing: inherit; display: block;"><summary style="box-sizing: inherit; display: list-item;">Can you think why?</summary><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;"></code><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;"></code><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;"></code><br style="box-sizing: inherit;"></details>

## Thanos Receive

Enter [Thanos Receive](https://thanos.io/tip/components/receive.md/).

`Thanos Receive` is a component that implements the [Prometheus Remote Write API](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write). This means that it will accept metrics data that is sent to it by other Prometheus instances (or any other process that implements Remote Write API).

Prometheus can be configured to `Remote Write`. This means that Prometheus will send all of its metrics data to a remote endpoint as they are being ingested - useful for our requirements!

In its simplest form, when `Thanos Receive` receives data from Prometheus, it stores it locally and exposes a `Store API` server so this data can be queried by `Thanos Query`.

`Thanos Receive` has more features that will be covered in future tutorials, but let's keep things simple for now.

## Run Thanos Receive

The first component we will run in our new architecture is `Thanos Receive`:

```
docker run -d \
    -v /data/receive-data:/receive/data \
    --net=host \
    --name receive \
    quay.io/thanos/thanos:v0.21.0 \
    receive \
    --tsdb.path "/receive/data" \
    --grpc-address 127.0.0.1:10907 \
    --http-address 127.0.0.1:10909 \
    --label "receive_replica=\"0\"" \
    --label "receive_cluster=\"wayne-enterprises\"" \
    --remote-write.address 127.0.0.1:10908
```

This starts Thanos Receive that listens on `http://127.0.0.1:10908/api/v1/receive' endpoint for Remote Write and on`127.0.0.1:10907` for Thanos StoreAPI.

Let's talk about some important parameters:

- `--label` - `Thanos Receive` requires at least one label to be set. These are called 'external labels' and are used to uniquely identify this instance of `Thanos Receive`.
- `--remote-write.address` - This is the address that `Thanos Receive` is listening on for Prometheus' remote write requests.

Let's verify that this is running correctly. Since `Thanos Receive` does not expose a UI, we can check it is up by retrieving its metrics page.

```
curl http://127.0.0.1:10909/metrics
```

## Run Thanos Query

Next, let us run a `Thanos Query` instance:

```
docker run -d --rm \
    --net=host \
    --name query \
    quay.io/thanos/thanos:v0.21.0 \
    query \
    --http-address "0.0.0.0:39090" \
    --store "127.0.0.1:10907"
```

`Thanos Receive` exposed its gRPC endpoint at `127.0.0.1:10907`, so we need to tell `Thanos Query` to use this endpoint with the `--store` flag.

Verify that `Thanos Query` is working and configured correctly by looking at the 'stores' tab [here](https://2886795294-39090-ollie09.environments.katacoda.com/stores).

Now we are done right? Try querying for some data...

![alt text](/Users/xingminwang/data/wayne/virtualhosts/thanos/images/query-graph-02.png)

<details open="" style="box-sizing: inherit; display: block;"><summary style="box-sizing: inherit; display: list-item;">Uh-oh! Do you know why are we seeing 'Empty Query Result' responses?</summary>We have correctly configured<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">Thanos Receive</code><span>&nbsp;</span>&amp;<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">Thanos Query</code>, but we have not yet configured Prometheus to write to remote write its data to the right place.</details>



<iframe title="recaptcha challenge expires in two minutes" src="https://www.google.com/recaptcha/api2/bframe?hl=en&amp;v=BycHQdSIhzR_1EcOLw2mOzYQ&amp;k=6LcuVhgUAAAAAMCgjmRjhfoTXoTAOVNoJlSRs0tI" name="c-of6clm40c75w" frameborder="0" scrolling="no" sandbox="allow-forms allow-popups allow-same-origin allow-scripts allow-top-navigation allow-modals allow-popups-to-escape-sandbox" style="box-sizing: inherit; background: rgb(255, 255, 255); width: 1790px; height: 150px;"></iframe>

#### Configure Prometheus Remote Write

# Configure Prometheus Remote Write

Our problem in the last step was that we have not yet configured Prometheus to `remote_write` to our `Thanos Receive` instance.

We need to tell `prometheus-batcave` & `prometheus-batcomputer` where to write their data to.

## Update Configuration

The docs for this configuration option can be found [here](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write).

```
# /vagrant/prometheus/conf/prometheus-batcave.yaml
global:
  scrape_interval: 5s
  external_labels:
    cluster: batcave
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']
remote_write:
- url: 'http://127.0.0.1:10908/api/v1/receive'

# /vagrant/prometheus/conf/prometheus-batcomputer.yaml
global:
  scrape_interval: 5s
  external_labels:
    cluster: batcomputer
    replica: 0

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9091']
remote_write:
- url: 'http://127.0.0.1:10908/api/v1/receive'
```

## Reload Configuration

Since we supplied the `--web.enable-lifecycle` flag in our Prometheus instances, we can dynamically reload the configuration by `curl`-ing the `/-/reload` endpoints.

```
curl -X POST http://127.0.0.1:9090/-/reload
curl -X POST http://127.0.0.1:9091/-/reload
```

Verify this has taken affect by checking the `/config` page on our Prometheus instances:

- `prometheus-batcave` [config page](http://192.168.56.100:9090/config)
- `prometheus-batcomputer` [config page](http://192.168.56.100:9091/config)

In both cases you should see the `remote_write` options in the configuration



#### Verify Setup

# Verify Setup

At this point, we have:

- Two prometheus instances configured to `remote_write`.
- `Thanos Receive` component ingesting data from Prometheus
- `Thanos Query` component configured to query `Thanos Receive`'s Store API.

The final task on our list is to verify that data is flowing correctly.

Stop and think how you could do this before opening the answer below 👇

<details open="" style="box-sizing: inherit; display: block;"><summary style="box-sizing: inherit; display: list-item;">How are we going to check that the components are wired up correctly?</summary>Let's make sure that we can query data from each of our Prometheus instances from our<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">Thanos Query</code><span>&nbsp;</span>instance. Navigate to the<span>&nbsp;</span><a href="https://2886795294-39090-ollie09.environments.katacoda.com/" rel="nofollow" target="_blank" style="box-sizing: inherit; background-color: transparent; color: rgb(3, 155, 229); text-decoration: none; -webkit-tap-highlight-color: transparent;">Thanos Query UI</a>, and query for a metric like<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">up</code><span>&nbsp;</span>- inspect the output and you should see<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">batcave</code><span>&nbsp;</span>and<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">batcomputer</code><span>&nbsp;</span>in the<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">cluster</code><span>&nbsp;</span>label.<span>&nbsp;</span><img src="https://www.katacoda.com/thanos/courses/thanos/3-receiver/assets/receive-cluster-result.png" alt="alt-text" data-featherlight="https://www.katacoda.com/thanos/courses/thanos/3-receiver/assets/receive-cluster-result.png" style="box-sizing: inherit; border-style: none; cursor: zoom-in; max-width: 100%;"></details>





 

<iframe title="recaptcha challenge expires in two minutes" src="https://www.google.com/recaptcha/api2/bframe?hl=en&amp;v=BycHQdSIhzR_1EcOLw2mOzYQ&amp;k=6LcuVhgUAAAAAMCgjmRjhfoTXoTAOVNoJlSRs0tI" name="c-of6clm40c75w" frameborder="0" scrolling="no" sandbox="allow-forms allow-popups allow-same-origin allow-scripts allow-top-navigation allow-modals allow-popups-to-escape-sandbox" style="box-sizing: inherit; background: rgb(255, 255, 255); width: 1790px; height: 150px;"></iframe>
