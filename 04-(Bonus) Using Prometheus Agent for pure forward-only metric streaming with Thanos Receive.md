The [Thanos](https://www.katacoda.com/thanos/courses/thanos/thanos.io) project defines a set of components that can be composed together into a highly available metric system with **unlimited storage capacity** that **seamlessly** integrates into your existing Prometheus deployments.

But [Prometheus](https://prometheus.io/) project is far from slowing down the development. Together with the community, it constantly evolves to bring value to different network and cluster topologies that we see. Thanos brings distributed and cloud storage and querying to the table, built on the core Prometheus code and design. It allowed Prometheus to focus on critical collection and single cluster monitoring functionalities.

As we learned in the previous tutorial, certain situations require us to collect (pull) data from applications and stream them out of the cluster as soon as possible. `Thanos Receive` allows doing that by ingesting metrics using the Remote Write protocol that the sender can implement. Typically we recommended using Prometheus with short retention and blocked read and query API as a "lightweight" sender.

In November 2021, however, we, the Prometheus community, introduced a brand new Prometheus mode called "Agent mode". The implementation itself was already battle tested on https://github.com/grafana/agent, where it was available and authored by [Robert Fratto](https://github.com/rfratto) since 2020.

The agent mode is optimized for efficient metric scraping and forwarding (i.e. immediate metric removal once it's securely delivered to a remote location). Since this is incredibly helpful for the Thanos community, we wanted to give you first-hand experience deploying Prometheus Agent together with Thanos Receive in this course.

In this tutorial, you will learn:

- How to reduce Prometheus based client-side metric collection to a minimum, using the new Prometheus "Agent mode" with `Thanos Receiver`, explained in the previous tutorial.

> NOTE: This course uses docker containers with pre-built Thanos, Prometheus, and Minio Docker images available publicly.



#### Problem Statement & Setup

## Problem Statement

Let's get back to our example from [Tutorial 3](https://www.katacoda.com/thanos/courses/thanos/3-receiver). Imagine you run a company called `Wayne Enterprises`. In tutorial 3, we established monitoring of two special clusters: `Batcave` & `Batcomputer`. These are special because they do not expose public endpoints to the Prometheus instances running there for security reasons, so we used the Remote Write protocol to stream all metrics to Thanos Receive in our centralized space.

Let's imagine we want to expand our `Wayne Enterprises` by adding metrics collection to applications running on smaller devices inside more mobile Batman tools: `Batmobile` and `Batcopter`.

Each of these vehicles has a smaller computer that runs applications from which we want to scrape Prometheus-like metrics using OpenMetrics format.

As the person responsible for implementing monitoring in these environments, you have three requirements to meet:

1. Implement a global view of this data. `Wayne Enterprises` needs to know what is happening in all company parts - including secret ones!
2. `Batmobile` and `Batcopter` can be out of network for some duration of time. You don't want to lose precious data.
3. `Batmobile` and `Batcopter` environments are very **resource constrained**, so you want an efficient solution that avoids extra computations and storage.

Firstly, let us set up Thanos as we explained in the previous tutorial.

## Setup Central Platform

As you might remember from the previous tutorial, in the simplest form for streaming use cases, we need to deploy Thanos Receive and Thanos Querier.

Let's run `Thanos Receive`:

```
docker run -d  \
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

This starts Thanos Receive that listens on `http://127.0.0.1:10908/api/v1/receive` endpoint for Remote Write and on `127.0.0.1:10907` for Thanos StoreAPI.

Next, let us run a `Thanos Query` instance connected to Thanos Receive:

```
docker run -d --rm \
--net=host \
--name query \
quay.io/thanos/thanos:v0.21.0 \
query \
--http-address "0.0.0.0:39090" \
--store "127.0.0.1:10907"
```

Verify that `Thanos Query` is working and configured correctly by looking at the 'stores' tab [here](http://192.168.56.100:39090/stores) (try refreshing the Thanos UI if it does not show up straight away).

We should see Receive store on this page and, as expected, no metric data since we did not connect any Remote Write sender yet.

With our "central" platform that is ready to ingest metrics, we can now start to architect our collection pipeline that will stream all metrics to our `Wayne Enterprises`.



#### Setup Prometheus Agents

# Setup `batmobile` and `batcopter` lightweight metric collection

With `Wayne Enterprise` platform ready to ingest Remote Write data, we need to think about how we satisfy our three requirements:

1. Implement a global view of this data. `Wayne Enterprises` needs to know what is happening in all company parts - including secret ones!
2. `Batmobile` and `Batcopter` can be out of network for some duration of time. You don't want to lose precious data.
3. `Batmobile` and `Batcopter` do not have large compute power, so you want an efficient solution that avoids extra computations and storage.

How are we going to do this?

Previously, the recommended approach was to deploy Prometheus to our edge environment which scrapes required applications, then remote writes all metrics to Thanos Receive. This will work and satisfy 1st and 2nd requirements, however Prometheus does some things that we don't need in this specific scenario:

- We don't want to alert locally or use recording rules from our edge places.
- We don't want to query data locally, and we want to store it for an as short duration as possible.

Prometheus was designed as a stateful time-series database, and it adds certain mechanics which are not desired for full forward mode. For example:

- Prometheus builds additional memory structures for easy querying from memory.
- Prometheus does not remove data when it is safely sent via remote write. It waits for at least two hours and only after the TSDB block is persisted to the disk, it may or may not be removed, depending on retention configuration.

This is where Agent mode comes in handy! It is a native Prometheus mode built into the Prometheus binary. If you add the `--agent` flag when running Prometheus, it will run a dedicated, specially streamlined database, optimized for forwarding purposes, yet able to persist scraped data in the event of a crash, restart or network disconnection.

Let's try to deploy it to fulfil `batmobile` and `batcopter` monitoring requirements.

## Deploy `Prometheus Agent` on `batmobile`

Let's use a very simple configuration file that tells prometheus agent to scrape its own metrics page every 5 seconds and forwards it's to our running `Thanos Receive`.

```
cat > /vagrant/prometheus/conf/prom-agent-batmobile.yaml << EOF
global:
  scrape_interval: 5s
  external_labels:
    cluster: batmobile
    replica: 0

scrape_configs:
  - job_name: 'prometheus-agent'
    static_configs:
      - targets: ['127.0.0.1:9090']

remote_write:
- url: 'http://127.0.0.1:10908/api/v1/receive'
EOF
```

Run the prometheus in agent mode:

```
docker run -d --net=host \
-v /vagrant/prometheus/conf/prom-agent-batmobile.yaml:/etc/prometheus/prometheus.yaml \
-v /data/prom-batmobile-data:/prometheus \
-u root \
--name prom-agent-batmobile \
quay.io/prometheus/prometheus:v2.32.0-beta.0 \
--enable-feature=agent \
--config.file=/etc/prometheus/prometheus.yaml \
--storage.agent.path=/prometheus \
--web.listen-address=:9090
```

This runs Prometheus Agent, which will scrape itself and forward all to Thanos Receive. It also exposes UI with pages that relate to scraping, service discovery, configuration and build information.

Verify that `prom-agent-batmobile` is running by navigating to the [Batmobile Prometheus Agent UI](http://192.168.56.100:9090/targets).

You should see one target: Prometheus Agent on `batmobile` itself.

## Deploy `Prometheus Agent` on `batcopter`

Similarly, we can configure and deploy the second agent:

```
cat > /vagrant/prometheus/conf/prom-agent-batcopter.yaml << EOF
global:
  scrape_interval: 5s
  external_labels:
    cluster: batcopter
    replica: 0

scrape_configs:
  - job_name: 'prometheus-agent'
    static_configs:
      - targets: ['127.0.0.1:9091']

remote_write:
- url: 'http://127.0.0.1:10908/api/v1/receive'
EOF

docker run -d --net=host \
-v /vagrant/prometheus/conf/prom-agent-batcopter.yaml:/etc/prometheus/prometheus.yaml \
-v /data/prom-batcopter-data:/prometheus \
-u root \
--name prom-agent-batcopter \
quay.io/prometheus/prometheus:v2.32.0-beta.0 \
--enable-feature=agent \
--config.file=/etc/prometheus/prometheus.yaml \
--storage.agent.path=/prometheus \
--web.listen-address=:9091
```

Verify that `prom-agent-batcopter` is running by navigating to the [Batcopter Prometheus Agent UI](http://192.168.56.100:9091/targets).

You should see one target: Prometheus Agent on `batcopter` itself.

Now, let's navigate to the last step to verify our `Wayne Enterprises` setup!

# Verify Setup

At this point, we have:

- Two Prometheus instances configured to `remote_write` and running in `agent` mode.
- `Thanos Receive` component ingesting data from Prometheus
- `Thanos Query` component configured to query `Thanos Receive`'s Store API.

The final task on our list is to verify that data is flowing correctly.

Stop and think how you could do this before opening the answer below 👇

<details open="" style="box-sizing: inherit; display: block;"><summary style="box-sizing: inherit; display: list-item;">How are we going to check that the components are wired up correctly?</summary>Let's make sure that we can query data from each of our Prometheus instances from our<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">Thanos Query</code><span>&nbsp;</span>instance. Navigate to the<span>&nbsp;</span><a href="http://192.168.56.100:9090/" rel="nofollow" target="_blank" style="box-sizing: inherit; background-color: transparent; color: rgb(3, 155, 229); text-decoration: none; -webkit-tap-highlight-color: transparent;">Thanos Query UI</a>, and query for a metric like<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">up</code><span>&nbsp;</span>or<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">go_goroutines</code><span>&nbsp;</span>- inspect the output and you should see<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">batmobile</code><span>&nbsp;</span>and<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">batcopter</code><span>&nbsp;</span>in the<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">cluster</code><span>&nbsp;</span>label.<span>&nbsp;</span><code style="box-sizing: inherit; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, Courier, monospace; font-size: 14.85px; color: rgb(0, 0, 0); background: rgb(245, 242, 240); padding: 2px 5px; display: inline-block; white-space: pre-wrap;">go_goroutines</code><span>&nbsp;</span>should look something like on image below:<span>&nbsp;</span><img src="https://www.katacoda.com/thanos/courses/thanos/4-receiver-agent/assets/expected.png" alt="expected" data-featherlight="https://www.katacoda.com/thanos/courses/thanos/4-receiver-agent/assets/expected.png" style="box-sizing: inherit; border-style: none; cursor: zoom-in; max-width: 100%;"></details>