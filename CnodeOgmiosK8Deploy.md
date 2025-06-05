# Deploying Ogmios and Cardano Node on a Talos Kubernetes Cluster with PVC for Byron Genesis

## Abstract

This document provides a comprehensive guide to deploying the Ogmios and Cardano Node services on a Talos Linux Kubernetes cluster running on Proxmox. It addresses the issue of the large Byron genesis file exceeding Kubernetes ConfigMap size limits by using a PersistentVolumeClaim (PVC) instead. The guide includes steps for setting up persistent storage, deploying the services with an InitContainer to handle genesis files, and verifying the deployment. The setup is tailored for prototyping Cardano blockchain applications.

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Deployment Steps](#deployment-steps)
  - [Create PersistentVolumeClaims (PVCs)](#create-persistentvolumeclaims-pvcs)
  - [Create ConfigMap for Configuration Files](#create-configmap-for-configuration-files)
  - [Deploy Cardano Node and Ogmios](#deploy-cardano-node-and-ogmios)
  - [Expose Ogmios Service](#expose-ogmios-service)
  - [Optional: Use MetalLB for LoadBalancer](#optional-use-metallb-for-loadbalancer)
  - [Verify the Deployment](#verify-the-deployment)
- [Troubleshooting](#troubleshooting)
- [Prototyping with Ogmios](#prototyping-with-ogmios)
- [Conclusion](#conclusion)
- [References](#references)

## Introduction

This guide details the process of deploying the `cardano-node` and `ogmios` services on a Talos Linux Kubernetes cluster hosted on Proxmox. Talos Linux is a secure, immutable operating system designed for Kubernetes, and this setup leverages its API-driven management with `talosctl` and `kubectl`. The primary modification from the original instructions is the use of a PersistentVolumeClaim (PVC) to handle the large Byron genesis file, which exceeds the 1MB size limit of Kubernetes ConfigMaps. An InitContainer is used to download the genesis files into the PVC, ensuring compatibility with the Cardano node’s configuration.

The cluster consists of three virtual machines:

- **talos-master** (192.168.1.42): Control-plane node, managing the Kubernetes API and cluster state.
- **talos-worker-1** (192.168.1.41): Worker node, running containerized workloads.
- **talos-worker-2** (192.168.1.33): Worker node, providing redundancy and scalability.

## Prerequisites

Before proceeding, ensure the following:

- Talos Linux VMs are running on Proxmox with static IPs (`192.168.1.42`, `192.168.1.41`, `192.168.1.33`).
- `talosctl` is installed on your Ubuntu machine:

  ```bash
  curl -Lo /tmp/talosctl https://github.com/siderolabs/talos/releases/download/v1.9.5/talosctl-linux-amd64
  chmod +x /tmp/talosctl
  sudo mv /tmp/talosctl /usr/local/bin/talosctl
  ```

- `kubectl` is installed and configured with the cluster's `kubeconfig`:

  ```bash
  talosctl kubeconfig .
  ```

- The Ogmios repository is cloned with submodules initialized:

  ```bash
  git clone --depth 1 --recursive --shallow-submodules https://github.com/CardanoSolutions/ogmios.git
  cd ogmios
  git submodule update --init --recursive
  ```

- A storage class (e.g., `local-path` or Longhorn) is available for persistent volumes. If using Longhorn, install it with:

  ```bash
  helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
  ```

## Deployment Steps

### Create PersistentVolumeClaims (PVCs)

Create three PVCs: one for the Cardano node database, one for the IPC socket, and one for the genesis files (including the large Byron genesis file).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cardano-node-db
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: local-path
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cardano-node-ipc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cardano-genesis-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: local-path
```

Apply the PVCs:

```bash
kubectl apply -f pvc.yaml
```

**Note**: If using Longhorn, update `storageClassName` to `longhorn` in the YAML.

### Create ConfigMap for Configuration Files

Create a ConfigMap for `config.json` and `topology.json`, excluding the genesis files due to the Byron file’s size.

```bash
kubectl create configmap cardano-mainnet-config \
  --from-file=config.json=server/config/network/mainnet/cardano-node/config.json \
  --from-file=topology.json=server/config/network/mainnet/cardano-node/topology.json
```

Verify the ConfigMap:

```bash
kubectl describe configmap cardano-mainnet-config
```

**Note**: For non-mainnet networks (e.g., `testnet`), replace `mainnet` with the appropriate network directory (e.g., `server/config/network/testnet`).

### Deploy Cardano Node and Ogmios

Use an InitContainer in the `cardano-node` deployment to download the Alonzo, Shelley, and Byron genesis files into the `cardano-genesis-pvc`. The `ogmios` deployment remains unchanged, using the ConfigMap and IPC socket.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cardano-node
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cardano-node
  template:
    metadata:
      labels:
        app: cardano-node
    spec:
      initContainers:
      - name: download-genesis
        image: busybox
        command: ['sh', '-c', 'wget -O /genesis/alonzo-genesis.json https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/alonzo-genesis.json && wget -O /genesis/shelley-genesis.json https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/shelley-genesis.json && wget -O /genesis/byron-genesis.json https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/byron-genesis.json']
        volumeMounts:
        - name: genesis
          mountPath: /genesis
      containers:
      - name: cardano-node
        image: ghcr.io/intersectmbo/cardano-node:10.1.4
        command:
        - cardano-node
        - run
        - --config
        - /config/config.json
        - --database-path
        - /data/db
        - --socket-path
        - /ipc/node.socket
        - --topology
        - /config/topology.json
        volumeMounts:
        - name: config
          mountPath: /config
        - name: genesis
          mountPath: /genesis
        - name: node-db
          mountPath: /data
        - name: node-ipc
          mountPath: /ipc
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2"
      volumes:
      - name: config
        configMap:
          name: cardano-mainnet-config
      - name: genesis
        persistentVolumeClaim:
          claimName: cardano-genesis-pvc
      - name: node-db
        persistentVolumeClaim:
          claimName: cardano-node-db
      - name: node-ipc
        persistentVolumeClaim:
          claimName: cardano-node-ipc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ogmios
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ogmios
  template:
    metadata:
      labels:
        app: ogmios
    spec:
      containers:
      - name: ogmios
        image: cardanosolutions/ogmios:latest
        command:
        - /bin/ogmios
        - --host
        - 0.0.0.0
        - --node-socket
        - /ipc/node.socket
        - --node-config
        - /config/cardano-node/config.json
        ports:
        - containerPort: 1337
        volumeMounts:
        - name: config
          mountPath: /config/cardano-node
        - name: node-ipc
          mountPath: /ipc
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1"
      volumes:
      - name: config
        configMap:
          name: cardano-mainnet-config
          items:
          - key: config.json
            path: config.json
      - name: node-ipc
        persistentVolumeClaim:
          claimName: cardano-node-ipc
```

Apply the deployments:

```bash
kubectl apply -f deployments.yaml
```

### Expose Ogmios Service

Expose the `ogmios` service using a NodePort to allow external access.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ogmios
  namespace: default
spec:
  selector:
    app: ogmios
  ports:
  - protocol: TCP
    port: 1337
    targetPort: 1337
  type: NodePort
```

Apply the service:

```bash
kubectl apply -f service.yaml
```

Find the NodePort:

```bash
kubectl get svc ogmios
```

Access `ogmios` at `ws://<worker-ip>:<node-port>` (e.g., `ws://192.168.1.41:3xxxx`).

### Optional: Use MetalLB for LoadBalancer

For a stable external IP, install MetalLB and configure a LoadBalancer service.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

Configure MetalLB:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.100-192.168.1.150
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

Apply the configuration:

```bash
kubectl apply -f metallb-config.yaml
```

Update the service to use LoadBalancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ogmios
  namespace: default
spec:
  selector:
    app: ogmios
  ports:
  - protocol: TCP
    port: 1337
    targetPort: 1337
  type: LoadBalancer
```

Apply the updated service:

```bash
kubectl apply -f service.yaml
```

Get the LoadBalancer IP:

```bash
kubectl get svc ogmios
```

Access `ogmios` at `ws://<external-ip>:1337` (e.g., `ws://192.168.1.100:1337`).

### Verify the Deployment

Check pod status:

```bash
kubectl get pods
```

Expected output:

```
NAME                           READY   STATUS    RESTARTS   AGE
cardano-node-xxx   1/1     Running   0          5m
ogmios-xxx         1/1     Running   0          5m
```

Check logs for issues:

```bash
kubectl logs -l app=cardano-node
kubectl logs -l app=ogmios
```

Test Ogmios with a WebSocket client:

```bash
wscat -c ws://<external-ip>:1337
{"jsonrpc":"2.0","method":"getChainTip","id":1}
```

## Troubleshooting

- **Pods Not Running**: Check logs (`kubectl logs`) or describe pods (`kubectl describe pod`) for errors, such as missing genesis files or configuration issues.
- **Genesis File Issues**: Verify the InitContainer downloaded the files correctly by exec-ing into the pod (if possible) or checking the PVC contents via a temporary pod:
  ```bash
  kubectl run -i --tty --rm debug --image=busybox --restart=Never -- sh
  # Inside the pod:
  ls /genesis
  ```
- **Network Issues**: Ensure connectivity to `192.168.1.42:6443` (Kubernetes API) and the Ogmios service port. Verify worker node IPs are accessible.
- **Node Health**: Check Talos node services:
  ```bash
  talosctl --nodes 192.168.1.42 get services
  ```
- **Proxmox Snapshots**: Take snapshots of VMs (`talos-master`, `talos-worker-1`, `talos-worker-2`) before applying manifests for easy rollbacks.

## Prototyping with Ogmios

Interact with Ogmios via its WebSocket API (see [Ogmios User Manual](https://ogmios.dev/)). Example:

```bash
wscat -c ws://192.168.1.100:1337
{"jsonrpc":"2.0","method":"getChainTip","id":1}
```

Consider deploying Kupo for indexing (see [Kupo+Ognios Example](https://cardano-solutions.github.io/kupo/)).

## Conclusion

This guide provides a complete setup for deploying Ogmios and Cardano Node on a Talos Kubernetes cluster, using a PVC to handle the large Byron genesis file. The InitContainer approach ensures all genesis files are accessible, maintaining compatibility with the original configuration while resolving ConfigMap size limitations. This setup enables prototyping Cardano blockchain applications in a secure, scalable environment.

## References

- [Talos Linux Documentation](https://www.talos.dev/)
- [Kubernetes ConfigMaps Documentation](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Ogmios User Manual](https://ogmios.dev/)
- [Kupo+Ognios Example](https://cardano-solutions.github.io/kupo/)
- [MetalLB Documentation](https://metallb.universe.tf/)