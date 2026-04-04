# Simple Kubernetes Architecture for Apache App

This repository contains a simple Kubernetes setup for an Apache web app while still covering the main things an app usually needs:

- `Namespace`
- `PersistentVolume`
- `PersistentVolumeClaim`
- `ConfigMap`
- `Deployment`
- `Service`
- `Ingress`
- `HorizontalPodAutoscaler`
- TLS secret template

## Architecture

```text
Browser
  |
  v
Ingress
  |
  v
Service
  |
  v
Deployment
  |
  v
Apache Pods
  |
  v
PersistentVolumeClaim
  |
  v
PersistentVolume
```

## Files

- `1_kind_cluster.yaml` -> local `kind` cluster config
- `2_namespace.yaml` -> app namespace
- `3a_persistent_volume.yaml` -> local persistent volume for Apache content
- `3_configmap.yaml` -> app page and health file
- `3b_persistent_volume_claim.yaml` -> claim used by the Apache pods
- `4_deployment.yaml` -> runs Apache pods
- `5_service.yaml` -> stable internal service for the pods
- `6_hpa.yaml` -> scales pods when CPU usage is high
- `7_ingress.yaml` -> exposes the app outside the cluster
- `8_tls_secret.example.yaml` -> template for TLS cert and key
- `kustomization.yaml` -> deploy everything together

## What Each Resource Does

### Namespace

Keeps the application resources grouped together.

### ConfigMap

Stores the seed content for:

- `index.html`
- `healthz.html`

### PersistentVolume and PersistentVolumeClaim

Provide local persistent storage for the Apache document root in this `kind` lab.

### Deployment

Runs 2 Apache pods and includes:

- resource requests and limits
- startup probe
- readiness probe
- liveness probe
- rolling update strategy
- an init container that copies the ConfigMap files into the PVC only when they do not already exist

### Service

Creates one stable internal endpoint for the Apache pods.

### Ingress

Routes traffic from `apache.prod.local` to the service.

### HPA

Scales the `Deployment` from `2` to `5` pods based on CPU usage.

## Deploy Steps

### 1. Create the kind cluster

```bash
kind create cluster --config 1_kind_cluster.yaml
```

This config mounts `./storage/apache-data` into every kind node so the PV has a local backing path.
If the cluster already exists, recreate it so the new mount is applied.

### 2. Install an ingress controller

Install `ingress-nginx` in the cluster.

Official docs:

- https://kind.sigs.k8s.io/docs/user/ingress/
- https://kubernetes.github.io/ingress-nginx/deploy/

### 3. Install Metrics Server

Metrics Server is needed for HPA.

Official docs:

- https://github.com/kubernetes-sigs/metrics-server

### 4. Create TLS secret

Use your certificate and key:

```bash
kubectl create secret tls apache-tls \
  --namespace apache-prod \
  --cert=tls.crt \
  --key=tls.key
```

You can also edit `8_tls_secret.example.yaml` and use it as a template.

### 5. Deploy the app

```bash
kubectl apply -k .
```

### 6. Add hosts entry for local testing

```text
127.0.0.1 apache.prod.local
```

## Verify

```bash
kubectl get all -n apache-prod
kubectl get pv
kubectl get pvc -n apache-prod
kubectl get ingress -n apache-prod
kubectl describe hpa apache-hpa -n apache-prod
```

## Notes

- `1_kind_cluster.yaml` is a `kind` cluster config file, not a Kubernetes manifest applied with `kubectl apply`.
- The PV in this lab uses a host-backed path mounted into each kind node, which is suitable for local practice but not for production storage design.
- `7_ingress.yaml` expects an ingress class called `nginx`.
- HPA works only after Metrics Server is installed.
- The TLS secret template is not included in `kustomization.yaml` on purpose. Create it separately with real values.
