#### Configuring Initial Prometheus Server

# Step 1 - Initial Prometheus Setup

In this tutorial, we will mimic the usual state with a Prometheus server running for... a year!. We will use it to seamlessly backup all old data in the object storage and configure Prometheus for continuous backup mode, which will allow us to cost-effectively achieve unlimited retention for Prometheus.

Last but not the least, we will go through setting all up for querying and automated maintenance (e.g compactions, retention and downsampling).

In order to showcase all of this, let's start with a single cluster setup from the previous course. Let's start this initial Prometheus setup, ready?

## Generate Artificial Metrics for 1 year

Actually, before starting Prometheus, let's generate some **artificial data**. You most likely want to learn about Thanos fast, so you probably don't have months to wait for this tutorial until Prometheus collects the month of metrics, do you? (:

We will use our handy [thanosbench](https://github.com/thanos-io/thanosbench) project to do so! Let's generate Prometheus data (in form of TSDB blocks) with just 5 series (gauges) that spans from a year ago until now (-6h)!

Execute the following command (should take few seconds):

```
mkdir -p /root/prom-eu1 && docker run -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="eu1"' --max-time=6h | docker run -v /root/prom-eu1:/prom-eu1 -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir prom-eu1
```

On successful block creation you should see following log lines:

```
level=info ts=2020-10-20T18:28:42.625041939Z caller=block.go:87 msg="all blocks done" count=13
level=info ts=2020-10-20T18:28:42.625100758Z caller=main.go:118 msg=exiting cmd="block gen"
```

Run the below command to see dozens of generated TSDB blocks:

```
ls -lR /root/prom-eu1
```

## Prometheus Configuration File

Here, we will prepare configuration files for the Prometheus instance that will run with our pre-generated data. It will also scrape our components we will use in this tutorial.

Click `Copy To Editor` for config to propagate the configs to file.

```
Copy to Editorglobal:
  scrape_interval: 5s
  external_labels:
    cluster: eu1
    replica: 0
    tenant: team-eu # Not needed, but a good practice if you want to grow this to multi-tenant system some day.

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9090']
  - job_name: 'sidecar'
    static_configs:
      - targets: ['127.0.0.1:19090']
  - job_name: 'minio'
    metrics_path: /minio/prometheus/metrics
    static_configs:
      - targets: ['127.0.0.1:9000']
  - job_name: 'querier'
    static_configs:
      - targets: ['127.0.0.1:9091']
  - job_name: 'store_gateway'
    static_configs:
      - targets: ['127.0.0.1:19091']
```

## Starting Prometheus Instance

Let's now start the container representing Prometheus instance.

Note `-v /root/prom-eu1:/prometheus \` and `--storage.tsdb.path=/prometheus` that allows us to place our generated data in Prometheus data directory.

Let's deploy Prometheus now. Note that we disabled local Prometheus compactions `storage.tsdb.max-block-duration` and `min` flags. Currently, this is important for the basic object storage backup scenario to avoid conflicts between the bucket and local compactions. Read more [here](https://thanos.io/tip/components/sidecar.md/#sidecar).

We also extend Prometheus retention: `--storage.tsdb.retention.time=1000d`. This is because Prometheus by default removes all data older than 2 weeks. And we have a year (:

### Deploying "EU1"

```
docker run -d --net=host --rm \
    -v /root/editor/prometheus0_eu1.yml:/etc/prometheus/prometheus.yml \
    -v /root/prom-eu1:/prometheus \
    -u root \
    --name prometheus-0-eu1 \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:9090 \
    --web.external-url=https://2886795300-9090-simba08.environments.katacoda.com \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

## Setup Verification

Once started you should be able to reach the Prometheus instance here and query.. 1 year of data!

- [Prometheus-0 EU1](https://2886795300-9090-simba08.environments.katacoda.com/graph?g0.range_input=1y&g0.expr=continuous_app_metric0&g0.tab=0)

## Thanos Sidecar & Querier

Similar to previous course, let's setup global view querying with sidecar:

```
docker run -d --net=host --rm \
    --name prometheus-0-eu1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.24.0 \
    sidecar \
    --http-address 0.0.0.0:19090 \
    --grpc-address 0.0.0.0:19190 \
    --prometheus.url http://127.0.0.1:9090
```

And Querier. As you remember [Thanos sidecar](https://thanos.io/tip/components/query.md/) exposes `StoreAPI` so we will make sure we point the Querier to the gRPC endpoints of the sidecar:

```
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.24.0 \
    query \
    --http-address 0.0.0.0:9091 \
    --query.replica-label replica \
    --store 127.0.0.1:19190
```

## Setup verification

Similar to previous course let's check if the Querier works as intended. Let's look on [Querier UI Store page](https://2886795300-9091-simba08.environments.katacoda.com/stores).

This should list the sidecar, including the external labels.

On graph you should also see our 5 series for 1y time, thanks to Prometheus and sidecar StoreAPI: [Graph](https://2886795300-9091-simba08.environments.katacoda.com/graph?g0.range_input=1y&g0.max_source_resolution=0s&g0.expr=continuous_app_metric0&g0.tab=0).

#### Thanos Sidecars

# Step 2 - Object Storage Continuous Backup

Maintaining one year of data within your Prometheus is doable, but not easy. It's tricky to resize, backup or maintain this data long term. On top of that Prometheus does not do any replication, so any unavailability of Prometheus results in query unavailability.

This is where Thanos comes to play. With a single configuration change we can allow Thanos Sidecar to continuously upload blocks of metrics that are periodically persisted to disk by the Prometheus.

> NOTE: Prometheus when scraping data, initially aggregates all samples in memory and WAL (on-disk write-head-log). Only after 2-3h it "compacts" the data into disk in form of 2h TSDB block. This is why we need to still query Prometheus for latest data, but overall with this change we can keep Prometheus retention to minimum. It's recommended to keep Prometheus retention in this case at least 6 hours long, to have safe buffer for a potential event of network partition.

## Starting Object Storage: Minio

Let's start simple S3-compatible Minio engine that keeps data in local disk:

```
mkdir /root/minio && \
docker run -d --rm --name minio \
     -v /root/minio:/data \
     -p 9000:9000 -e "MINIO_ACCESS_KEY=minio" -e "MINIO_SECRET_KEY=melovethanos" \
     minio/minio:RELEASE.2019-01-31T00-31-19Z \
     server /data
```

Create `thanos` bucket:

```
mkdir /root/minio/thanos
```

## Verification

To check if the Minio is working as intended, let's [open Minio server UI](https://2886795300-9000-simba08.environments.katacoda.com/minio/)

Enter the credentials as mentioned below:

**Access Key** = `minio` **Secret Key** = `melovethanos`

## Sidecar block backup

All Thanos components that use object storage uses the same `objstore.config` flag with the same "little" bucket config format.

Click `Copy To Editor` for config to propagate the configs to the file `bucket_storage.yaml`:

```
Copy to Editortype: S3
config:
  bucket: "thanos"
  endpoint: "127.0.0.1:9000"
  insecure: true
  signature_version2: true
  access_key: "minio"
  secret_key: "melovethanos"
```

Let's restart sidecar with updated configuration in backup mode.

```
docker stop prometheus-0-eu1-sidecar
```

[Thanos sidecar](https://thanos.io/tip/components/sidecar.md/) allows to backup all the blocks that Prometheus persists to the disk. In order to accomplish this we need to make sure that:

- Sidecar has direct access to the Prometheus data directory (in our case host's /root/prom-eu1 dir) (`--tsdb.path` flag)
- Bucket configuration is specified `--objstore.config-file`
- `--shipper.upload-compacted` has to be set if you want to upload already compacted blocks when sidecar starts. Use this only when you want to upload blocks never seen before on new Prometheus introduced to Thanos system.

Let's run sidecar:

```
docker run -d --net=host --rm \
    -v /root/editor/bucket_storage.yaml:/etc/thanos/minio-bucket.yaml \
    -v /root/prom-eu1:/prometheus \
    --name prometheus-0-eu1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.24.0 \
    sidecar \
    --tsdb.path /prometheus \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --shipper.upload-compacted \
    --http-address 0.0.0.0:19090 \
    --grpc-address 0.0.0.0:19190 \
    --prometheus.url http://127.0.0.1:9090
```

## Verification

We can check whether the data is uploaded into `thanos` bucket by visitng [Minio](https://2886795300-9000-simba08.environments.katacoda.com/minio/). It will take couple of seconds to synchronize all blocks.

Once all blocks appear in the minio `thanos` bucket, we are sure our data is backed up. Awesome!

#### Thanos Store Gateway

# Step 3 - Fetching metrics from Bucket

In this step, we will learn about Thanos Store Gateway and how to deploy it.

## Thanos Components

Let's take a look at all the Thanos commands:

```
docker run --rm quay.io/thanos/thanos:v0.24.0 --help
```

You should see multiple commands that solve different purposes, block storage based long-term storage for Prometheus.

In this step we will focus on thanos `store gateway`:

```
  store [<flags>]
    Store node giving access to blocks in a bucket provider
```

## Store Gateway:

- This component implements the Store API on top of historical data in an object storage bucket. It acts primarily as an API gateway and therefore does not need significant amounts of local disk space.
- It keeps a small amount of information about all remote blocks on the local disk and keeps it in sync with the bucket. This data is generally safe to delete across restarts at the cost of increased startup times.

You can read more about [Store](https://thanos.io/tip/components/store.md/) here.

### Deploying store for "EU1" Prometheus data

```
docker run -d --net=host --rm \
    -v /root/editor/bucket_storage.yaml:/etc/thanos/minio-bucket.yaml \
    --name store-gateway \
    quay.io/thanos/thanos:v0.24.0 \
    store \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --http-address 0.0.0.0:19091 \
    --grpc-address 0.0.0.0:19191
```

## How to query Thanos store data?

In this step, we will see how we can query Thanos store data which has access to historical data from the `thanos` bucket, and play with this setup a bit.

Currently querier does not know about store yet. Let's change it by adding Store Gateway gRPC address `--store 127.0.0.1:19191` to Querier:

```
docker stop querier && \
docker run -d --net=host --rm \
   --name querier \
   quay.io/thanos/thanos:v0.24.0 \
   query \
   --http-address 0.0.0.0:9091 \
   --query.replica-label replica \
   --store 127.0.0.1:19190 \
   --store 127.0.0.1:19191
```

Click on the Querier UI `Graph` page and try querying data for a year or two by inserting metrics `continuous_app_metric0` ([Query UI](https://2886795300-9091-simba08.environments.katacoda.com/graph?g0.range_input=1y&g0.max_source_resolution=0s&g0.expr=continuous_app_metric0&g0.tab=0)). Make sure `deduplication` is selected and you will be able to discover all the data fetched by Thanos store.

![img](/Users/xingminwang/data/wayne/virtualhosts/thanos/images/query-graph01.png)

Also, you can check all the active endpoints located by thanos-store by clicking on [Stores](https://2886795300-9091-simba08.environments.katacoda.com/stores).

### What we know so far?

We've added Thanos Query, a component that can query a Prometheus instance and Thanos Store at the same time, which gives transparent access to the archived blocks and real-time metrics. The vanilla PromQL Prometheus engine used inside Query deduces what time series and for what time ranges we need to fetch the data for.

Also, StoreAPIs propagate external labels and the time range they have data for, so we can do basic filtering on this. However, if you don't specify any of these in the query (only "up" series) the querier concurrently asks all the StoreAPI servers.

The potential duplication of data between Prometheus+sidecar results and store Gateway will be transparently handled and invisible in results.

### Checking what StoreAPIs are involved in query

Another interesting question here is how to ensure if we query the data from bucket only?

We can check this by visitng the [New UI](https://2886795300-9091-simba08.environments.katacoda.com/new/graph?g0.expr=&g0.tab=0&g0.stacked=0&g0.range_input=1h&g0.max_source_resolution=0s&g0.deduplicate=1&g0.partial_response=0&g0.store_matches=[]), inserting `continuous_app_metric0` metrics again with 1 year time range of graph, and click on `Enable Store Filtering`.

This allows us to filter stores and helps in debugging from where we are querying the data exactly.

Let's chose only `127.0.0.1:19191`, so store gateway. This query will only hit that store to retrieve data, so we are sure that store works.

## Question Time? 🤔

In an HA Prometheus setup with Thanos sidecars, would there be issues with multiple sidecars attempting to upload the same data blocks to object storage?

Think over this 😉

To see the answer to this question click `SHOW SOLUTION` below.

## Next

Voila! In the next step, we will talk about Thanos Compactor, its retention capabilities, and how it improves query efficiency and reduce the required storage size.

#### Thanos Compactor

# Step 4 - Thanos Compactor

In this step, we will install Thanos Compactor which applies the compaction, retention, deletion and downsampling operations on the TSDB block data object storage.

Before, moving forward, let's take a closer look at what the `Compactor` component does:

> Note: Thanos Compactor is currently designed to be run as a singleton. Unavailability (hours) is not problematic as it does not serve any live requests.
>
> ## Compactor

The `Compactor` is an essential component that operates on a single object storage bucket to compact, downsample, apply retention, to the TSDB blocks held inside, thus, making queries on historical data more efficient. It creates aggregates of old metrics (based upon the rules).

It is also responsible for downsampling of data, performing 5m downsampling after 40 hours, and 1h downsampling after 10 days.

If you want to know more about Thanos Compactor, jump [here](https://thanos.io/tip/components/compact.md/).

> Note: Thanos Compactor is mandatory if you use object storage otherwise Thanos Store Gateway will be too slow without using a compactor.

## Deploying Thanos Compactor

Click below snippet to start the Compactor.

```
docker run -d --net=host --rm \
 -v /root/editor/bucket_storage.yaml:/etc/thanos/minio-bucket.yaml \
    --name thanos-compact \
    quay.io/thanos/thanos:v0.24.0 \
    compact \
    --wait --wait-interval 30s \
    --consistency-delay 0s \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --http-address 0.0.0.0:19095
```

The flag `wait` is used to make sure all compactions have been processed while `--wait-interval` is kept in 30s to perform all the compactions and downsampling very quickly. Also, this only works when when `--wait` flag is specified. Another flag `--consistency-delay` is basically used for buckets which are not consistent strongly. It is the minimum age of non-compacted blocks before they are being processed. Here, we kept the delay at 0s assuming the bucket is consistent.

## Setup Verification

To check if compactor works fine, we can look at the [Bucket View](https://2886795300-19095-simba08.environments.katacoda.com/new/loaded).

Now, if we click on the blocks, they will provide us all the metadata (Series, Samples, Resolution, Chunks, and many more things).

## Compaction and Downsampling

When we query large historical data there are millions of samples that we need to go through which makes the queries slower and slower as we retrieve year's worth of data.

Thus, Thanos uses the technique called downsampling (a process of reducing the sampling rate of the signal) to keep the queries responsive, and no special configuration is required to perform this process.

The Compactor applies compaction to the bucket data and also completes the downsampling for historical data.

To expierience this, click on the [Querier](https://2886795300-9091-simba08.environments.katacoda.com/new/graph?g0.expr=&g0.tab=0&g0.stacked=0&g0.range_input=1h&g0.max_source_resolution=0s&g0.deduplicate=1&g0.partial_response=0&g0.store_matches=[]) and insert metrics `continuous_app_metric0` with 1 year time range of graph, and also, click on `Enable Store Filtering`.

Let's try querying `Max 5m downsampling` data, it uses 5m resolution and it will be faster than the raw data. Also, Downsampling is built on top of data, and never done on **young** data.

## Unlimited Retention - Not Challenging anymore?

Having a long time metric retention for Prometheus was always involving lots of complexity, disk space, and manual work. With Thanos, you can make Prometheus almost stateless, while having most of the data in durable and cheap object storage.

## Next

Awesome work! Feel free to play with the setup 🤗

Once done, hit `Continue` for summary.