# Router and Monitoring Pod Configuration in OF@TEIN Kubernetes Cluster
Repository to provision router and monitoring pods in OF@TEIN++ multiple Kubernetes cluster

## Overview

## Requirements

## Configuration

Example of router configuration for UM site worker node `um-router-pod.yaml` is shown below:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: router-pod-um
spec:
  volumes:
    - name: configDir
      mountPath: "..."
  containers:
    - name: quagga
      image:
      command:
      volumeMounts:
        - mountPath: "/config"
          name: configDir
  nodeName: k8s-worker-um-1
```

## Usage

To apply the configuration please use this `kubectl` command:

```shell script
kubectl apply -f um-router-pod.yaml
```

## Troubleshooting