# Deploying Ogmios and Cardano Node on a Talos Kubernetes Cluster with PVC for Byron Genesis (v1.1.0)

## Abstract

This document provides a comprehensive guide to deploying the Ogmios and Cardano Node services on a Talos Linux Kubernetes cluster running on Proxmox. It addresses the issue of the large Byron genesis file exceeding Kubernetes ConfigMap size limits (1MB) by using a PersistentVolumeClaim (PVC) for the Byron file, while keeping smaller Shelley and Alonzo genesis files in a ConfigMap. The guide supports uploading the Byron genesis file locally from the cloned Ogmios repository, avoiding wget downloads. The Cardano node database PVC is set to 200GB to accommodate blockchain data growth. This guide, versioned as v1.1.0, is tailored for prototyping Cardano blockchain applications on a cluster with three nodes: `talos-master` (192.168.1.42), `talos-worker-1` (192.168.1.41), and `talos-worker-2` (192.168.1.33).

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Deployment Steps](#deployment-steps)
  - [Create PersistentVolumeClaims (PVCs)](#create-persistentvolumeclaims-pvcs)
  - [Create ConfigMap for Configuration Files](#create-configmap-for-configuration-files)
  - [Upload Byron Genesis File Locally](#upload-byron-genesis-file-locally)
  - [Deploy Cardano Node](#deploy-cardano-node)
  - [Deploy Ogmios](#deploy-ogmios)
  - [Expose Ogmios Service](#expose-ogmios-service)
  - [Optional: Use MetalLB for LoadBalancer](#optional-use-metallb-for-loadbalancer)
  - [Verify the Deployment](#verify-the-deployment)
- [Troubleshooting](#troubleshooting)
- [Prototyping with Ogmios](#prototyping-with-ogmios)
- [Conclusion](#conclusion)
- [References](#references)
- [Version History](#version-history)

## Introduction

This guide details the deployment of `cardano-node` and `ogmios` on a Talos Linux Kubernetes cluster hosted on Proxmox, tailored for prototyping Cardano blockchain applications. Talos Linux is a secure, immutable operating system designed for Kubernetes, managed via `talosctl` and `kubectl`. The Byron genesis file, critical for Cardano’s blockchain initialization, exceeds the 1MB ConfigMap limit, necessitating a PVC. The smaller Shelley and Alonzo genesis files are included in a ConfigMap. The Cardano node database PVC is set to 200GB to handle blockchain data growth. The Byron genesis file is uploaded locally using a temporary pod, leveraging the cloned Ogmios repository, replacing the original wget-based approach. Pod affinity ensures `cardano-node` and `ogmios` share the IPC socket on the same node. This document is versioned as v1.1.0 to track updates.

## Prerequisites

Before proceeding, ensure the following:

- Talos Linux VMs are running on Proxmox with static IPs (`192.168.1.42`, `192.168.1.41`, `192.168.1.33`).
- `talosctl` is installed on your Ubuntu machine:
  ```bash
  curl -Lo /tmp/talosctl https://github.com/siderolabs/talos/releases/download/v1.9.5/talosctl-linux-amd64
  chmod +x /tmp/talosctl
  sudo mv /tmp/talosctl /usr/local/bin/talosctl
  ```
- `kubectl` is installed and configured with the cluster’s `kubeconfig`:
  ```bash
  talosctl kubeconfig .
  ```
- The Ogmios repository is cloned with submodules initialized:
  ```bash
  git clone --depth 1 --recursive --shallow-submodules https://github.com/CardanoSolutions/ogmios.git
  cd ogmios
  git submodule update --init --recursive
  ```
- The Byron genesis file (`mainnet-byron-genesis.json`) is available locally, either from the repository or downloaded from [Hydra](https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/index.html).
- A storage class (`local-path` or Longhorn) is available. For Longhorn, install it:
  ```bash
  helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace
  ```

## Deployment Steps

### Create PersistentVolumeClaims (PVCs)

Create three PVCs: one for the Cardano node database (200GB), one for the IPC socket, and one for the Byron genesis file.

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
      storage: 200Gi
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

**Note**: If using Longhorn for better durability, update `storageClassName` to `longhorn`. Ensure your Proxmox host has sufficient storage for the 200GB `cardano-node-db` PVC.

### Create ConfigMap for Configuration Files

Create a ConfigMap for `config.json`, `topology.json`, `shelley.json`, and `alonzo.json`, which are small enough for ConfigMaps.

```bash
kubectl create configmap cardano-mainnet-config \
  --from-file=config.json=server/config/network/mainnet/cardano-node/config.json \
  --from-file=topology.json=server/config/network/mainnet/cardano-node/topology.json \
  --from-file=shelley.json=server/config/network/mainnet/genesis/shelley.json \
  --from-file=alonzo.json=server/config/network/mainnet/genesis/alonzo.json
```

Verify the ConfigMap:
```bash
kubectl describe configmap cardano-mainnet-config
```

**Note**: Ensure the file paths match your local Ogmios repository structure. For testnets, use `server/config/network/testnet`.

### Upload Byron Genesis File Locally

Upload the local `mainnet-byron-genesis.json` file (from the Ogmios repository or [Hydra](https://hydra.iohk.io)) to the `cardano-genesis-pvc` using a temporary pod.

1. **Create a Temporary Pod**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: temp-pod
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: genesis-volume
      mountPath: /genesis
  volumes:
  - name: genesis-volume
    persistentVolumeClaim:
      claimName: cardano-genesis-pvc
```

Apply it:
```bash
kubectl apply -f temp-pod.yaml
```

2. **Copy the Byron Genesis File**:
```bash
kubectl cp /path/to/local/mainnet-byron-genesis.json temp-pod:/genesis/byron.json
```

**Note**: Replace `/path/to/local/mainnet-byron-genesis.json` with the actual path to your local file.

3. **Delete the Temporary Pod**:
```bash
kubectl delete pod temp-pod
```

### Deploy Cardano Node

Deploy `cardano-node` with an InitContainer to consolidate configuration files into `/opt/cardano/cnode/files`.

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
      - name: init-config
        image: busybox
        command: ['sh', '-c', 'cp /config/* /opt/cardano/cnode/files/ && cp /genesis/byron.json /opt/cardano/cnode/files/']
        volumeMounts:
        - name: config-volume
          mountPath: /config
        - name: genesis-volume
          mountPath: /genesis
        - name: cnode-files
          mountPath: /opt/cardano/cnode/files
      containers:
      - name: cardano-node
        image: ghcr.io/intersectmbo/cardano-node:10.1.4
        command: ['cardano-node', 'run', '--config', '/opt/cardano/cnode/files/config.json', '--database-path', '/data/db', '--socket-path', '/ipc/node.socket', '--topology', '/opt/cardano/cnode/files/topology.json']
        volumeMounts:
        - name: cnode-files
          mountPath: /opt/cardano/cnode/files
        - name: node-db
          mountPath: /data
        - name: node-ipc
          mountPath: /ipc
        resources:
          requests:
            memory: "20Gi"
            cpu: "2"
          limits:
            memory: "22Gi"
            cpu: "4"
      volumes:
      - name: config-volume
        configMap:
          name: cardano-mainnet-config
      - name: genesis-volume
        persistentVolumeClaim:
          claimName: cardano-genesis-pvc
      - name: cnode-files
        emptyDir: {}
      - name: node-db
        persistentVolumeClaim:
          claimName: cardano-node-db
      - name: node-ipc
        persistentVolumeClaim:
          claimName: cardano-node-ipc
```

Apply the deployment:
```bash
kubectl apply -f cardano-node.yaml
```

### Deploy Ogmios

Deploy `ogmios` with pod affinity to ensure it runs on the same node as `cardano-node`, sharing the IPC socket PVC.

```yaml
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
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cardano-node
            topologyKey: kubernetes.io/hostname
      containers:
      - name: ogmios
        image: cardanosolutions/ogmios:latest
        command: ['/bin/ogmios', '--host', '0.0.0.0', '--node-socket', '/ipc/node.socket', '--node-config', '/config/config.json']
        ports:
        - containerPort: 1337
        volumeMounts:
        - name: config-volume
          mountPath: /config
        - name: node-ipc
          mountPath: /ipc
        resources:
          requests:
            memory: "1Gi"
            cpu: "2"
          limits:
            memory: "2Gi"
            cpu: "2"
      volumes:
      - name: config-volume
        configMap:
          name: cardano-mainnet-config
      - name: node-ipc
        persistentVolumeClaim:
          claimName: cardano-node-ipc
```

Apply the deployment:
```bash
kubectl apply -f ogmios.yaml
```

### Expose Ogmios Service

Expose `ogmios` using a NodePort service.

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
kubectl apply -f ogmios-service.yaml
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
kubectl apply -f ogmios-service.yaml
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

- **Pods Not Running**: Check logs (`kubectl logs`) or describe pods (`kubectl describe pod`) for errors, such as missing files or configuration issues.
- **Byron Genesis File Issues**: Verify the file was copied correctly to the PVC:
  ```bash
  kubectl run -i --tty --rm debug --image=busybox --restart=Never -- sh
  # Inside the pod:
  ls /genesis
  ```
- **IPC Socket Issues**: Ensure `cardano-node` and `ogmios` are on the same node (`kubectl get pods -o wide`). Verify pod affinity settings.
- **Network Issues**: Confirm connectivity to `192.168.1.42:6443` (Kubernetes API) and the Ogmios service port.
- **Storage Issues**: Ensure your Proxmox host has sufficient storage for the 200GB `cardano-node-db` PVC. Check PVC status:
  ```bash
  kubectl get pvc
  ```
- **Node Health**: Check Talos services:
  ```bash
  talosctl --nodes 192.168.1.42 get services
  ```
- **Proxmox Snapshots**: Take snapshots of VMs before applying manifests for easy rollbacks.

## Prototyping with Ogmios

Interact with Ogmios via its WebSocket API ([Ogmios User Manual](https://ogmios.dev)). Example:
```bash
wscat -c ws://192.168.1.100:1337
{"jsonrpc":"2.0","method":"getChainTip","id":1}
```

Consider deploying Kupo for indexing ([Kupo Documentation](https://cardanosolutions.github.io/kupo)).

## Conclusion

This guide (v1.1.0) provides a complete setup for deploying Ogmios and Cardano Node on a Talos Kubernetes cluster, using a PVC for the large Byron genesis file and a ConfigMap for smaller files. The local upload method leverages the cloned repository, and the 200GB `cardano-node-db` PVC accommodates blockchain data growth. The setup enables prototyping Cardano blockchain applications in a secure, scalable environment.

## References

- [Cardano Configuration Files](https://iohk.zendesk.com/hc/en-us/articles/900001951706-Understanding-the-configuration-files)
- [Ogmios User Manual](https://ogmios.dev)
- [Kupo Documentation](https://cardanosolutions.github.io/kupo)
- [Kubernetes ConfigMaps Documentation](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Talos Linux Documentation](https://www.talos.dev)
- [MetalLB Documentation](https://metallb.universe.tf)
- [Longhorn Documentation](https://longhorn.io/docs)
- [Cardano Node Releases](https://github.com/intersectmbo/cardano-node/releases)
- [Cardano Course: Node Configuration](https://cardano-course.gitbook.io/cardano-course/handbook/protocol-parameters-and-configuration-files/node-configuration-file)

## Version History

- **v1.1.0** (June 5, 2025): Updated to use PVC for Byron genesis only, local file upload via temporary pod, 200GB `cardano-node-db` PVC, pod affinity for IPC socket, and versioning.
- **v1.0.0** (Original): Initial setup with PVC for all genesis files and wget-based InitContainer.
