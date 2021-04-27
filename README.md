# Router and Monitoring Pod Configuration in OF@TEIN Kubernetes Cluster
Repository to provision router and monitoring pods in OF@TEIN++ multiple Kubernetes cluster

## Overview

## Requirements

Loki service should deployed in the cluster.

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

## Usage

To apply the configuration please use this `kubectl` command:

```shell script
kubectl apply -f um-router-pod.yaml
```

## Troubleshooting
