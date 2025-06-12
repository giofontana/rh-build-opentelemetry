# Pre-requisites

1. RHEL server (tested on RHEL 9).
2. Subscribed and with either appstream and baseos repos enabled.
3. For simplicity, selinux is disabled in this demo. To disable it, run the following:

```bash
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
sudo reboot
```

# Install Mimir

Become root and install dependencies:

```bash
sudo -i
dnf install podman wget -y
mkdir -p ~/mimir/data
```

Create Mimir conf file:

```bash
cat <<EOF > ~/mimir/demo.yaml
# Do not use this configuration in production.
# It is for demonstration purposes only.
multitenancy_enabled: false
limits:
  promote_otel_resource_attributes: "k8s.pod.uid,k8s.pod.name,k8s.namespace.name,k8s.container.name,k8s.node.name,k8s.pod.start_time,service.name,host.name"
  ingestion_rate: 1000000
  max_global_series_per_user: 5000000
  max_label_names_per_series: 1000

blocks_storage:
  backend: filesystem
  bucket_store:
    sync_dir: /root/mimir/tsdb-sync
  filesystem:
    dir: /root/mimir/data/tsdb
  tsdb:
    dir: /root/mimir/tsdb

compactor:
  data_dir: /root/mimir/compactor
  sharding_ring:
    kvstore:
      store: memberlist

distributor:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: memberlist

ingester:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: memberlist
    replication_factor: 1

ruler_storage:
  backend: filesystem
  filesystem:
    dir: /root/mimir/rules

server:
  http_listen_port: 9009
  log_level: error

store_gateway:
  sharding_ring:
    replication_factor: 1
EOF
```

Deploy container:

```bash
podman network create grafanet

podman run -d \
  --rm \
  --name mimir \
  --network grafanet \
  --publish 9009:9009 \
  --volume /root/mimir/demo.yaml:/etc/mimir/demo.yaml \
  --volume /root/mimir/data:/root/mimir \
  grafana/mimir:latest \
  --config.file=/etc/mimir/demo.yaml
```

Test install (you should see mimir status page):

```bash
curl http://localhost:9009
```

# Install Grafana

mkdir -p ~/grafana/data
chown -R 472:472 ~/grafana/data

Deploy container:

```bash
podman run -d \
  --rm \
  --name=grafana \
  --network=grafanet \
  --volume /root/grafana/data:/var/lib/grafana \
  -p 3000:3000 grafana/grafana
```

# Install Loki

Create Loki conf file:

```bash
mkdir -p ~/loki/data
chown -R 10001:10001 ~/loki/data
mkdir -p ~/loki/config

cd ~/loki/config
wget https://raw.githubusercontent.com/grafana/loki/v3.4.1/cmd/loki/loki-local-config.yaml -O loki-config.yaml
wget https://raw.githubusercontent.com/grafana/loki/v3.4.1/clients/cmd/promtail/promtail-docker-config.yaml -O promtail-config.yaml
```

Deploy Loki and Promtail containers:

```bash
podman run -d \
  --name loki \
  --network grafanet \
  -v /root/loki/config:/mnt/config \
  -v /root/loki/data:/tmp/loki \
  -p 3100:3100 grafana/loki:3.4.1 \
  -config.file=/mnt/config/loki-config.yaml

podman run -d \
  --name promtail \
  --network grafanet \
  -v /root/loki/config:/mnt/config \
  -v /var/log:/var/log \
  grafana/promtail:3.4.1 \
  -config.file=/mnt/config/promtail-config.yaml
```