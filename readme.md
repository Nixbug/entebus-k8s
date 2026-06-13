# Entebus Server Kubernetes Deployment Guide

This guide explains how to deploy Entebus Server with Kustomize overlays for three scenarios:

- overlays/dev: standard dev deployment through ingress-nginx
- overlays/dev-cloudflared: dev deployment fronted by Cloudflare Tunnel (no public load balancer required)
- overlays/prod: production deployment through ingress-nginx

## Overlay Summary

### base

Base resources include:

- Namespace: entebus
- App deployment: entebus-server
- Service: entebus-server-svc (ClusterIP 80 -> 8080)
- Ingress: entebus-ingress
- Shared dependencies: PostGIS, Redis, MinIO, OpenObserve
- HPA and PodDisruptionBudget

### overlays/dev

Development overlay without Cloudflared components:

- Includes base
- Uses app image tag develop-latest
- Sets namespace label environment=dev
- Uses dev hostnames on ingress:
  - dev-api.entebus.com
  - dev-minio.entebus.com
  - dev-openobserve.entebus.com
- Keeps conservative HPA and small PVC sizes for dev

### overlays/dev-cloudflared

Development overlay with Cloudflare Tunnel side deployment:

- Inherits from overlays/dev (all dev behavior is reused)
- Adds cloudflared deployment and service account
- Adds cloudflared-credentials secret manifest placeholder
- Uses outbound tunnel from cluster to Cloudflare (no public LB for app ingress path)

### overlays/prod

Production overlay:

- Includes base
- Uses production-focused replica and resource settings
- Uses larger PVC sizes

### Image Tag Guidance

- Use `develop-latest` in dev overlays when you want rapid iteration and the newest build automatically.
- Use immutable tags (for example `v1.2.3` or a commit SHA) for reproducible testing and production rollouts.
- Promote by pinning the exact tested image tag in `overlays/prod/kustomization.yaml` before release.

## Prerequisites

- Kubernetes cluster with kubectl access
- kubectl and helm installed
- ingress-nginx installed (for ingress resources)
- Cloudflare account and zone access
- For tunnel mode: an existing Cloudflare Tunnel and credentials JSON
- For TLS termination with ingress: origin.crt and origin.key from Cloudflare Origin CA

Optional sanity checks:

```bash
kubectl version --client
kubectl cluster-info
helm version
```

## Validate Manifests Before Deploying

Run this before any apply:

```bash
kubectl kustomize overlays/dev > /tmp/entebus-dev.yaml
kubectl kustomize overlays/dev-cloudflared > /tmp/entebus-dev-cloudflared.yaml
kubectl kustomize overlays/prod > /tmp/entebus-prod.yaml
```

## Install Or Verify Ingress NGINX

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

## Cloudflare Origin TLS Secret

Create once per cluster/namespace if your ingress TLS uses Cloudflare Origin cert:

```bash
kubectl create secret tls cloudflare-origin-cert \
  --namespace entebus \
  --cert=origin.crt \
  --key=origin.key
```

If the secret already exists and you need to rotate it:

```bash
kubectl delete secret cloudflare-origin-cert -n entebus
kubectl create secret tls cloudflare-origin-cert \
  --namespace entebus \
  --cert=origin.crt \
  --key=origin.key
```

## Deploy Standard Dev (No Tunnel)

```bash
kubectl apply -k overlays/dev
kubectl get all -n entebus
kubectl get ingress -n entebus
```

## Deploy Dev With Cloudflare Tunnel

Use this overlay when you want Cloudflare Tunnel instead of exposing ingress through a typical public load balancer path.

### 1) Create a tunnel in Cloudflare

From Cloudflare Zero Trust dashboard:

1. Create or choose a tunnel for this environment.
2. Configure public hostnames:
   - dev-api.entebus.com
   - dev-minio.entebus.com
   - dev-openobserve.entebus.com
3. Point each hostname to your in-cluster ingress endpoint according to your tunnel routing model.

### 2) Add tunnel credentials into Kubernetes secret manifest

Edit overlays/dev-cloudflared/cloudflared-credentials-secret.yaml and replace REPLACE_WITH_CREDENTIALS_JSON with the raw credentials JSON content.

Security note: avoid committing real credentials to git. Prefer creating the secret out-of-band in real environments.

Safer alternative command:

```bash
kubectl create secret generic cloudflared-credentials \
  --namespace entebus \
  --from-file=credentials.json=./cloudflared-credentials.json \
  --dry-run=client -o yaml | kubectl apply -f -
```

If you use the command above, remove or ignore the placeholder secret manifest in overlays/dev-cloudflared to avoid accidental overwrite.

### 3) Apply the tunnel overlay

```bash
kubectl apply -k overlays/dev-cloudflared
```

### 4) Verify cloudflared and app paths

```bash
kubectl get deploy -n entebus
kubectl get pods -n entebus -l app=cloudflared
kubectl logs -n entebus deploy/cloudflared --tail=200
kubectl get ingress -n entebus
```

### 5) Functional checks

- https://dev-api.entebus.com/docs
- https://dev-minio.entebus.com
- https://dev-openobserve.entebus.com

## Deploy Prod

```bash
kubectl apply -k overlays/prod
kubectl get all -n entebus
kubectl get ingress -n entebus
```

## Rollback

Rollback to previous overlay state:

```bash
kubectl rollout undo deployment/entebus-server -n entebus
kubectl rollout undo deployment/cloudflared -n entebus
```

Or redeploy desired overlay explicitly:

```bash
kubectl apply -k overlays/dev
# or
kubectl apply -k overlays/dev-cloudflared
# or
kubectl apply -k overlays/prod
```

## Troubleshooting

```bash
kubectl describe ingress entebus-ingress -n entebus
kubectl describe pod -n entebus -l app=entebus-server
kubectl describe pod -n entebus -l app=cloudflared
kubectl get events -n entebus --sort-by=.lastTimestamp
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=200
```

Common checks:

- Ingress class is nginx
- TLS secret cloudflare-origin-cert exists in namespace entebus
- cloudflared pod is Running and not restarting
- Cloudflare tunnel hostname routes match the ingress hostnames
- Backend services and pods are Ready
