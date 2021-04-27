# Router and Monitoring Pod Configuration in OF@TEIN Kubernetes Cluster
Repository to provision router and monitoring pods in OF@TEIN++ multiple Kubernetes cluster

## Overview

## Requirements

For monitoring purpose, [Grafana](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#configuration) and [Loki](https://grafana.com/docs/loki/latest/installation/helm/) should deployed prior to this. 

## Configuration

Example of router configuration for UM site worker node `um-router-pod.yaml` is shown below:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: um-sandbox-3-pvc
  namespace: rpki
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: quagga-bgp-um-sandbox-3
  namespace: rpki
  labels:
    app: quagga-bgp-um-sandbox-3
spec:
      nodeName: um-sandbox-3
      containers:
      - name: quagga
        image: osrg/quagga
        ports:
        - containerPort: 179
        - containerPort: 2605
        - containerPort: 2601
        securityContext:
          capabilities:
            add:
              - NET_ADMIN
              - NET_BROADCAST
              - NET_RAW
              - SYS_ADMIN
        volumeMounts:
        - name: um-sandbox-3-storage
          mountPath: /etc/quagga/
      - name: ovs
        image: globocom/openvswitch
        ports:
        - containerPort: 22
        - containerPort: 5566
          protocol: UDP
        securityContext:
          capabilities:
            add:
              - NET_ADMIN
      volumes:
       - name: um-sandbox-3-storage
         persistentVolumeClaim:
           claimName: um-sandbox-3-pvc

```
Access the pod for initial configuration.
```
kubectl exec --stdin --tty -n rpki quagga-bgp-um-sandbox-3 -- /bin/bash
```

After applying the router pod yaml file, copy daemons.txt and debian.conf from /companion-files into /etc/quagga/.Because of using persistent volume to store the configuration file, the original files in the directory is wipe out during mounting the persistent volume, so we need to manually move the 2 files into the directory.

```
chown quagga.quaggavty /etc/quagga/*.conf
chmod 640 /etc/quagga/*.conf
/etc/init.d/quagga start
apt-get install ssh -y
ssh localhost
ovs-vsctl add-br switch

ovs-vsctl add-port switch vxlan-um-sandbox -- set interface vxlan-um-sandbox type=vxlan \
options:remote_ip=10.144.227.165 options:dst_port=5566

ifconfig switch 192.168.100.7 netmask 255.255.255.0 up 
cd /usr/local/bin
apt-get install nano curl -y
sudo curl -fSL -o promtail.gz "https://github.com/grafana/loki/releases/download/v1.6.1/promtail-linux-amd64.zip"
sudo gunzip promtail.gz
sudo chmod a+x promtail
sudo nano config-promtail.yml
```

```yaml
#config-promtail.yaml template		https://sbcode.net/grafana/install-promtail-service/
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: quagga
    entry_parser: raw
    static_configs:
      - targets:
          - localhost
        labels:
          job: quagga-um-sandbox-3
          __path__: /etc/quagga/bgpd.log

```

## Usage

To apply the configuration please use this `kubectl` command:

```shell script
kubectl apply -f um-router-pod.yaml
```

## Troubleshooting
