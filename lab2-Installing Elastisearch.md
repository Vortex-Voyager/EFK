# 🚀 Lab 2 — Deploying Elasticsearch on Kubernetes

A hands-on guide to deploy a single-node Elasticsearch on Kubernetes using StatefulSet, PersistentVolumes, and Services.

- **Audience:** Beginners learning EFK stack on Kubernetes
- **Estimated time:** 20–40 minutes

## Table of Contents

1. [Overview](#overview)
2. [Why StatefulSet?](#why-statefulset)
3. [Persistent Volumes (PV/PVC)](#persistent-volumes-pvpvc)
4. [Service: NodePort exposure](#service-nodeport-exposure)
5. [Hands-on Steps](#hands-on-steps)
   - [Step 1 — Namespace](#step-1---namespace)
   - [Step 2 — Persistent Volume](#step-2---persistent-volume)
   - [Step 3 — Service (NodePort)](#step-3---service-nodeport)
   - [Step 4 — StatefulSet](#step-4---statefulset)
   - [Step 5 — Verify](#step-5---verify)
6. [Manifests (copy/paste)](#manifests-copypaste)
7. [Notes & Tips](#notes--tips)

---

## Overview

Kubernetes orchestrates containers; Elasticsearch provides search and analytics. Running Elasticsearch on Kubernetes gives you resilience and scalable deployment primitives. This lab shows a minimal, single-node setup for learning purposes.

## Why StatefulSet?

- Stable network identities for pods (e.g., `elasticsearch-0`).
- Ordered deployment and graceful scaling.
- Supports per-pod persistent storage via `volumeClaimTemplates`.

## Persistent Volumes (PV/PVC)

PVs represent cluster storage; PVCs are user requests for storage. The StatefulSet's `volumeClaimTemplates` will create a PVC per pod. In this lab we use a hostPath-backed PV for simplicity.

## Service: NodePort exposure

We expose Elasticsearch with a `NodePort` service so you can curl the HTTP endpoint from the host. The lab uses node ports `30200` (9200) and `30300` (9300).

---

## Hands-on Steps

### Step 1 — Namespace

Create an isolated namespace and set your context:

```bash
kubectl create namespace elastic-stack
kubectl config set-context --current --namespace=elastic-stack
```

### Step 2 — Persistent Volume

Create a PV that maps to `/data/elasticsearch` on the node (5Gi):

```bash
# Save manifest as es-pvolume.yaml
kubectl apply -f es-pvolume.yaml
```

### Step 3 — Service (NodePort)

Create a service to expose ports 9200 and 9300 across the cluster and as NodePorts:

```bash
# Save manifest as es-service.yaml
kubectl apply -f es-service.yaml
```

Access HTTP API locally via:

```bash
curl http://localhost:30200
```

### Step 4 — StatefulSet

Create a StatefulSet that mounts the `es-data` volume and uses the `docker.elastic.co/elasticsearch/elasticsearch:7.1.0` image. This manifest includes an `initContainer` to fix directory permissions.

```bash
# Save manifest as es-statefulset.yaml
kubectl apply -f es-statefulset.yaml
```

> Note: This lab uses a single replica for simplicity (set replicas>1 for multi-node clusters).

### Step 5 — Verify

Check pods and service status:

```bash
kubectl get pods -l app=elasticsearch
kubectl get svc elasticsearch -n elastic-stack
curl http://localhost:30200/_cluster/state?pretty
```

---

## Manifests (copy/paste)

### PersistentVolume (es-pvolume.yaml)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-elasticsearch
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/elasticsearch
```

### Service (es-service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: elastic-stack
spec:
  selector:
    app: elasticsearch
  ports:
  - port: 9200
    targetPort: 9200
    nodePort: 30200
    name: port1
  - port: 9300
    targetPort: 9300
    nodePort: 30300
    name: port2
  type: NodePort
```

### StatefulSet (es-statefulset.yaml)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: elastic-stack
spec:
  serviceName: "elasticsearch"
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.1.0
        ports:
        - containerPort: 9200
          name: port1
        - containerPort: 9300
          name: port2
        env:
        - name: discovery.type
          value: single-node
        volumeMounts:
        - name: es-data
          mountPath: /usr/share/elasticsearch/data
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: es-data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: es-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

---

## Notes & Tips

- The hostPath PV is convenient for labs, but for production use dynamic storage classes or cloud volumes.
- The `initContainer` sets ownership to UID 1000 (Elasticsearch runtime). Adjust if you change images.
- Logs: `kubectl logs -f <pod-name>` to troubleshoot startup issues.
- If NodePort is blocked by firewall, use `kubectl port-forward` as an alternative.

---

## Checklist

- [x] Namespace created
- [x] PV applied
- [x] Service applied
- [x] StatefulSet applied
- [ ] Confirm cluster is healthy

---

Happy learning! If you'd like, I can:

- Convert this into a PDF or HTML for printing
- Create the separate manifest files (es-pvolume.yaml, es-service.yaml, es-statefulset.yaml) in the repository
- Run quick validation commands and show sample output (if you permit terminal access)
