# Prometheus/Grafana on Kubernetes

Install Prometheus/Grafana in Kubernetes cluster with sample app containing promeheus instrumentation. 

## Setup Prometheus 

A metrics namespace for our environment to live in
```
kubectl apply -f namespaces.yml
```

A ClusterRole to give Prometheus access to targets using Service Discovery
```
kubectl apply -f clusterRole.yml 
```

Update node's IP address in "prometheus-config-map.yml" 
```
kubectl apply -f prometheus-config-map.yml
```
A Prometheus Deployment and Service
```
kubectl apply -f prometheus-deployment.yml 
kubectl apply -f prometheus-service.yml
```

Kube State Metrics to get access to metrics on the Kubernetes API
```
kubectl apply -f kube-state-metrics.yml
```


## Setup Grafana

Change the password in `grafana-deployment.yml`. And run following
```
kubectl apply -f grafana-deployment.yml
kubectl apply -f grafana-service.yml
```

Go to Grafana UI. Go to "DataSources -> Add New Datasource". Setup "Prometheus" type datasource with public IP url of Prometheus.   

## Setup NodeExporter 
Follow instruction on each node on cluster to install node-exporter. 

Create the Prometheus user:
```
adduser prometheus
```
Download Node Exporter:
```
cd /home/prometheus
curl -LO "https://github.com/prometheus/node_exporter/releases/download/v0.16.0/node_exporter-0.16.0.linux-amd64.tar.gz"
tar -xvzf node_exporter-0.16.0.linux-amd64.tar.gz
mv node_exporter-0.16.0.linux-amd64 node_exporter
cd node_exporter
chown prometheus:prometheus node_exporter
```

Create service. Add file `/etc/systemd/system/node_exporter.service` with following content

```
[Unit]
Description=Node Exporter

[Service]
User=prometheus
ExecStart=/home/prometheus/node_exporter/node_exporter

[Install]
WantedBy=default.target
```
Reload systemd, enable and start the service. Make sure its active/running:
```
systemctl daemon-reload
systemctl enable node_exporter.service
systemctl start node_exporter.service
systemctl status node_exporter.service
```

## Prometheus Graph QL

Go to Prometheus and click on the Graph tab. 

Memory usage query:
```
((sum(node_memory_MemTotal_bytes) - sum(node_memory_MemFree_bytes) - sum(node_memory_Buffers_bytes) - sum(node_memory_Cached_bytes)) / sum(node_memory_MemTotal_bytes)) * 100
```

## Setup Grafana Dashboard

Import Grafana dashboard from file `grafana/dashboard/Kubernetes All Nodes.json` by importing it with previously created  propmetheus datasource. 


## Prometheus Instrumetation 

Prometheus client libraries are located here: https://prometheus.io/docs/instrumenting/clientlibs/

NodeJS uses swagger-stats library with very simple integration. 

```
var swStats = require('swagger-stats');

app.use(swStats.getMiddleware());
```

### Sample app 
Prometheus instrumentation: https://github.com/divyeshramani/content-kubernetes-prometheus-app

Clone the repo and to run the sample app locally (port 3000)
```
sudo apt-get install -y npm
npm install
npm start
```

Deploy sample app in kubernetes using deployment.yml file from sample app folder. 

```
# To build your own docker image. 
docker build -t your_docker_hub_id/comicbox . 
docker push your_docker_hub_id/comicbox

kubectl apply -f deployment.yml 
```

Import new Grafana swagger stats dashboard using dashboard id 3091 and select prometheus datasource. 


## PromQL

Doc: https://prometheus.io/docs/prometheus/latest/querying/basics/

```
node_cpu_seconds_total{job="node-exporter", mode="idle"}[5m]

# Use regex: Query job that begins with kube
container_cpu_load_average_10s{job=~"^kube.*"}

# Query job that end with -exporter
node_cpu_seconds_total{job=~".*-exporter"}
```

### PromQL Operations & Functions

Operators Doc: https://prometheus.io/docs/prometheus/latest/querying/operators/

Functions Doc: https://prometheus.io/docs/prometheus/latest/querying/functions/

```
# Get a percentage of total memory used by node:
((node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Buffers_bytes - node_memory_Cached_bytes) / node_memory_MemTotal_bytes) * 100

# Get a percentage of total memory used in Cluster:
((sum(node_memory_MemTotal_bytes) - sum(node_memory_MemFree_bytes) - sum(node_memory_Buffers_bytes) - sum(node_memory_Cached_bytes)) / sum(node_memory_MemTotal_bytes)) * 100

# Using Operations, Functions and Grouping in Queries
avg(irate(node_cpu_seconds_total{job="node-exporter", mode="idle"}[5m])) by (instance)
```