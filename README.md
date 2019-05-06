# Prometheus/Grafana on Kubernetes


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


## Setup NodeExporter
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

