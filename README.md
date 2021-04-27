# Router and Monitoring Pod Configuration in OF@TEIN Kubernetes Cluster
Repository to provision router and monitoring pods in OF@TEIN++ multiple Kubernetes cluster

## Overview

## Requirements

For monitoring purpose, [Grafana](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#configuration) and [Loki](https://grafana.com/docs/loki/latest/installation/helm/) should deployed prior to this. 

## Configuration

Example of router configuration for UM site worker node `um-sandbox-3.yaml` is shown below. In this yaml file, it contains 2 containers which is quagga router container and openvswitch(OVS) container. Quagga container bind with a persistent volume for configuration storage. Quagga container doesn't allow changing the vxlan destination port(by default Kubernetes Calico CNI is blocking overlay traffic between pods), hence ovs is used here to form overlay VXLAN tunnels between router pods. 

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
Access the pod for initial configuration. In this case, we are accessing the quagga container.
```
kubectl exec --stdin --tty -n rpki quagga-bgp-um-sandbox-3 -- /bin/bash
```

After applying the router pod yaml file, copy daemons and debian.conf from [/companion-files](https://github.com/skywood123/OFTEIN-Router-and-Monitoring-Pod/tree/temporary/router-pod/companion-files) into /etc/quagga/.Because of using persistent volume to store the configuration file, the original files in the directory is wipe out during mounting the persistent volume, so we need to manually move the 2 files into the directory during initial configuration.

```
chown quagga.quaggavty /etc/quagga/*.conf
chmod 640 /etc/quagga/*.conf
/etc/init.d/quagga start
apt-get install ssh -y
```

Install the promtail binary to scrape the static log file from the quagga software.
```
cd /usr/local/bin
apt-get install nano curl -y
sudo curl -fSL -o promtail.gz "https://github.com/grafana/loki/releases/download/v1.6.1/promtail-linux-amd64.zip"
sudo gunzip promtail.gz
sudo chmod a+x promtail
sudo nano config-promtail.yml
```
Paste the promtail yaml configuration below.

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

Run the promtail binary 
```
sudo ./promtail -config.file ./config-promtail.yml
```
Next, configure the Overlay Vxlan connection between the router pods. If using hub-and-spoke topology, hub router pod need to point to all spoke routers and spoke router pods point back to hub router pod.
The openvswitch container image comes with ssh only. Access the ovs container using ssh.

```
ssh localhost
ovs-vsctl add-br switch

ovs-vsctl add-port switch vxlan-drukren -- set interface vxlan-drukren type=vxlan \
  options:remote_ip=10.144.180.161 options:dst_port=5566 

ifconfig switch 192.168.100.1 netmask 255.255.255.0 up 
```
## Usage

To apply the configuration please use this `kubectl` command:

```shell script
kubectl apply -f um-router-pod.yaml
```

## Troubleshooting
