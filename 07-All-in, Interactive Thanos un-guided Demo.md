This playground was created for `@rawkode` live demo made by `@bwplotka` and David McKay.

You can watch all here: https://www.youtube.com/watch?v=j4TAGO019HU&feature=emb_title&ab_channel=DavidMcKay

Otherwise, have fun with those resources!

#### Yolo Demo Resources

## Global View + HA: Querier with 1y of data.

```
cd /data/editor && CURR_DIR=$(pwd)
```

### Generating data:

- Plan for 1y worth of metrics data:

```
docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --max-time=6h > ${CURR_DIR}/block-spec.yaml
```

- Plan and generate data for `cluster="eu1", replica="0"` Prometheus:

```
mkdir ${CURR_DIR}/prom-eu1-replica0 && docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="eu1"' --max-time=6h | \
    docker run -v ${CURR_DIR}/prom-eu1-replica0:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out
```

- Plan and generate data for `cluster="eu1", replica="1"` Prometheus:

```
mkdir ${CURR_DIR}/prom-eu1-replica1 && docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="eu1"' --max-time=6h | \
    docker run -v ${CURR_DIR}/prom-eu1-replica1:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out
```

- Plan and generate data for `cluster="us1", replica="0"` Prometheus:

```
mkdir ${CURR_DIR}/prom-us1-replica0 && docker run -it --rm quay.io/thanos/thanosbench:v0.2.0-rc.1 block plan -p continuous-365d-tiny --labels 'cluster="us1"' --max-time=6h | \
    docker run -v ${CURR_DIR}/prom-us1-replica0:/out -i quay.io/thanos/thanosbench:v0.2.0-rc.1 block gen --output.dir /out
```

### Create Promethues configs to use:

- `cluster="eu1", replica="0"` Prometheus:

```
cat <<EOF > ${CURR_DIR}/prom-eu1-replica0-config.yaml
global:
  scrape_interval: 5s
  external_labels:
    cluster: eu1
    replica: 0
    tenant: team-eu # Not needed, but a good practice if you want to grow this to multi-tenant system some day.

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9091','127.0.0.1:9092']
EOF
```

- `cluster="eu1", replica="1"` Prometheus:

```
cat <<EOF > ${CURR_DIR}/prom-eu1-replica1-config.yaml
global:
  scrape_interval: 5s
  external_labels:
    cluster: eu1
    replica: 1
    tenant: team-eu # Not needed, but a good practice if you want to grow this to multi-tenant system some day.

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9091','127.0.0.1:9092']
EOF
```

- `cluster="us1", replica="0"` Prometheus:

```
cat <<EOF > ${CURR_DIR}/prom-us1-replica0-config.yaml
global:
  scrape_interval: 5s
  external_labels:
    cluster: us1
    replica: 0
    tenant: team-us

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['127.0.0.1:9093']
EOF
```

### Ports for Prometheus-es/Prometheis/Prometheus

```
PROM_EU1_0_PORT=9091 && \
PROM_EU1_1_PORT=9092 && \
PROM_US1_0_PORT=9093
PROM_EU1_0_EXT_ADDRESS=http://192.168.56.100:${PROM_EU1_0_PORT} && \
PROM_EU1_1_EXT_ADDRESS=http://192.168.56.100:${PROM_EU1_1_PORT} && \
PROM_US1_0_EXT_ADDRESS=http://192.168.56.100:${PROM_US1_0_PORT}
```

### Deploying Prometheus-es/Prometheus/Prometheus instances

```
docker run -it --rm quay.io/prometheus/prometheus:v2.20.0 --help
```

- `cluster="eu1", replica="0"` Prometheus:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-eu1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    -v ${CURR_DIR}/prom-eu1-replica0:/prometheus \
    -u root \
    --name prom-eu1-0 \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:${PROM_EU1_0_PORT} \
    --web.external-url=${PROM_EU1_0_EXT_ADDRESS} \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

- `cluster="eu1", replica="1"` Prometheus:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-eu1-replica1-config.yaml:/etc/prometheus/prometheus.yml \
    -v ${CURR_DIR}/prom-eu1-replica1:/prometheus \
    -u root \
    --name prom-eu1-1 \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:${PROM_EU1_1_PORT} \
    --web.external-url=${PROM_EU1_1_EXT_ADDRESS} \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

- `cluster="us1", replica="0"` Prometheus:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-us1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    -v ${CURR_DIR}/prom-us1-replica0:/prometheus \
    -u root \
    --name prom-us1-0 \
    quay.io/prometheus/prometheus:v2.20.0 \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/prometheus \
    --storage.tsdb.retention.time=1000d \
    --storage.tsdb.max-block-duration=2h \
    --storage.tsdb.min-block-duration=2h \
    --web.listen-address=:${PROM_US1_0_PORT} \
    --web.external-url=${PROM_US1_0_EXT_ADDRESS} \
    --web.enable-lifecycle \
    --web.enable-admin-api
```

### Step1: Sidecar

```
docker run -it --rm quay.io/thanos/thanos:v0.24.0 --help
```

- `cluster="eu1", replica="0"` sidecar:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-eu1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    --name prom-eu1-0-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.24.0 \
    sidecar \
    --http-address 0.0.0.0:19091 \
    --grpc-address 0.0.0.0:19191 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:${PROM_EU1_0_PORT}"
```

- `cluster="eu1", replica="1"` sidecar:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-eu1-replica1-config.yaml:/etc/prometheus/prometheus.yml \
    --name prom-eu1-1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.24.0 \
    sidecar \
    --http-address 0.0.0.0:19092 \
    --grpc-address 0.0.0.0:19192 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:${PROM_EU1_1_PORT}"
```

- `cluster="us1", replica="0"` sidecar:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-us1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    --name prom-us1-0-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.24.0 \
    sidecar \
    --http-address 0.0.0.0:19093 \
    --grpc-address 0.0.0.0:19193 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:${PROM_US1_0_PORT}"
```

### Step2: Global View + HA: Querier

```
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.24.0 \
    query \
    --http-address 0.0.0.0:9090 \
    --grpc-address 0.0.0.0:19190 \
    --query.replica-label replica \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192 \
    --store 127.0.0.1:19193
```

Visit [http://192.168.56.100:9090](http://192.168.56.100:9090) to see Thanos UI.

#### Yolo Demo Resources

## Long term retention!

### Step 3: Let's start object storage first.

In this step, we will configure the object store and change sidecar to upload to the object-store.

## Running Minio

```
mkdir ${CURR_DIR}/minio && \
docker run -d --rm --name minio \
     -v ${CURR_DIR}/minio:/data \
     -p 9000:9000 -e "MINIO_ACCESS_KEY=rawkode" -e "MINIO_SECRET_KEY=rawkodeloveobs" \
     minio/minio:RELEASE.2019-01-31T00-31-19Z \
    server /data
```

Create `thanos` bucket:

```
mkdir ${CURR_DIR}/minio/thanos
```

To check if the Minio is working as intended, let's [open Minio server UI](http://192.168.56.100:9000/minio/)

Enter the credentials as mentioned below:

**Access Key** = `rawkode` **Secret Key** = `rawkodeloveobs`

### Step 4: Configure sidecar to upload blocks.

```
cat <<EOF > ${CURR_DIR}/minio-bucket.yaml
type: S3
config:
  bucket: "thanos"
  endpoint: "127.0.0.1:9000"
  insecure: true
  signature_version2: true
  access_key: "rawkode"
  secret_key: "rawkodeloveobs"
EOF
```

Before moving forward, we need to roll new sidecar configuration with new bucket config.

```
docker stop prom-eu1-0-sidecar
docker stop prom-eu1-1-sidecar
docker stop prom-us1-0-sidecar
```

Now, execute the following command :

- `cluster="eu1", replica="0"` sidecar:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-eu1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    -v ${CURR_DIR}/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    -v ${CURR_DIR}/prom-eu1-replica0:/prometheus \
    --name prom-eu1-0-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.24.0 \
    sidecar \
    --tsdb.path /prometheus \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --shipper.upload-compacted \
    --http-address 0.0.0.0:19091 \
    --grpc-address 0.0.0.0:19191 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:${PROM_EU1_0_PORT}"
```

- `cluster="eu1", replica="1"` sidecar:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-eu1-replica1-config.yaml:/etc/prometheus/prometheus.yml \
    -v ${CURR_DIR}/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    -v ${CURR_DIR}/prom-eu1-replica1:/prometheus \
    --name prom-eu1-1-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.24.0 \
    sidecar \
    --tsdb.path /prometheus \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --shipper.upload-compacted \
    --http-address 0.0.0.0:19092 \
    --grpc-address 0.0.0.0:19192 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:${PROM_EU1_1_PORT}"
```

- `cluster="us1", replica="0"` sidecar:

```shell
docker run -d --net=host --rm \
    -v ${CURR_DIR}/prom-us1-replica0-config.yaml:/etc/prometheus/prometheus.yml \
    -v ${CURR_DIR}/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    -v ${CURR_DIR}/prom-us1-replica0:/prometheus \
    --name prom-us1-0-sidecar \
    -u root \
    quay.io/thanos/thanos:v0.24.0 \
    sidecar \
    --tsdb.path /prometheus \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --shipper.upload-compacted \
    --http-address 0.0.0.0:19093 \
    --grpc-address 0.0.0.0:19193 \
    --reloader.config-file /etc/prometheus/prometheus.yml \
    --prometheus.url "http://127.0.0.1:${PROM_US1_0_PORT}"
```

We can check whether the data is uploaded into `thanos` bucket by visiting [Minio](http://192.168.56.100:9090/minio/) (or `localhost:9000`) It will take a minute to synchronize all blocks. Note that sidecar by default uploads only "non compacted by Prometheus" blocks.

See [this](https://thanos.io/tip/components/sidecar.md/#upload-compacted-blocks) to read more about uploading old data already touched by Prometheus.

Once we see all ~40 blocks appear in the minio, we are sure our data is backed up. Awesome!

### Mhm, how to query those data now?

Let's run Store Gateway server:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    --name store-gateway \
    quay.io/thanos/thanos:v0.24.0 \
    store \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --http-address 0.0.0.0:19094 \
    --grpc-address 0.0.0.0:19194
```

### Let's point query to new StoreAPI!

```
docker stop querier && \
docker run -d --net=host --rm \
    --name querier \
    quay.io/thanos/thanos:v0.24.0 \
    query \
    --http-address 0.0.0.0:9090 \
    --grpc-address 0.0.0.0:19190 \
    --query.replica-label replica \
    --store 127.0.0.1:19191 \
    --store 127.0.0.1:19192 \
    --store 127.0.0.1:19193 \
    --store 127.0.0.1:19194
```

Visit [http://192.168.56.100:9090/](http://192.168.56.100:9090/) to see Thanos UI.

### Long term maintenance, retention, dedup and downsampling:

```
docker run -d --net=host --rm \
    -v ${CURR_DIR}/minio-bucket.yaml:/etc/thanos/minio-bucket.yaml \
    --name compactor \
    quay.io/thanos/thanos:v0.24.0 \
    compact \
    --wait --wait-interval 30s \
    --consistency-delay 0s \
    --objstore.config-file /etc/thanos/minio-bucket.yaml \
    --http-address 0.0.0.0:19095
```

Visit http://192.168.56.100:19095/new/loaded to see Compactor Web UI.

### Data should be immediately downsampled as well for smooth experience!

Visit [http://192.168.56.100:9090](http://192.168.56.100:9090/) to see Thanos UI and query for 1 year.

Check 5m downsampling vs raw data.