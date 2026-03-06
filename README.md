
# Flux CD Complete Guide - GitOps with Kubernetes

A comprehensive guide to implementing GitOps with Flux CD on Kubernetes. This project covers Git sources, Helm Controller, Kustomization, OCI registries, image automation, and monitoring.

## Table of Contents

1. [Installation](#installation)
2. [Getting Started](#getting-started)
3. [Git Sources](#git-sources)
4. [Kustomization](#kustomization)
5. [Helm Controller](#helm-controller-important) ⭐ **IMPORTANT**
6. [Object Storage (S3/Bucket)](#object-storage-s3bucket-sources)
7. [OCI Registry](#oci-registry-sources)
8. [Image Automation](#image-automation)
9. [Monitoring](#monitoring)
10. [Sealed Secrets](#sealed-secrets)
11. [Common Commands](#common-commands)
12. [Troubleshooting](#troubleshooting)

---

## Installation

### Install Flux CLI

```powershell
# Windows - Using Chocolatey
choco install flux

# Verify installation
flux version
```

---

## Getting Started

### Setup Environment

```powershell
# Set GitHub Personal Access Token
$env:GITHUB_TOKEN = "ghp_YOUR_TOKEN_HERE"

# Verify token
echo $env:GITHUB_TOKEN
```

### Bootstrap Flux to GitHub

```powershell
flux bootstrap github `
    --token-auth `
    --owner=soumen321 `
    --repository=FluxCDPlaygroud `
    --branch=main `
    --path=clusters/fluxcluster `
    --private=false `
    --personal=true
```

### Verify Bootstrap

```powershell
# Check Flux is running
flux get source git

# View kustomizations
flux get kustomization

# Get all Flux components
kubectl get all -n flux-system
```

---

## Git Sources

### Create a Git Source

```powershell
# Basic Git source from GitHub
flux create source git podinfo \
    --url=https://github.com/stefanprodan/podinfo \
    --branch=master

# Export to YAML file
flux create source git demo2-source-git \
    --url=https://github.com/yogeshraheja/KubernetesVotingApp.git \
    --branch=master \
    --export > demo2-source-git.yaml
```

### Manage Git Sources

```powershell
# List all git sources
flux get source git

# Manually reconcile (force sync)
flux reconcile source git flux-system

# Watch reconciliation logs
flux logs -f

# Suspend source (stop syncing)
flux suspend source git flux-system

# Resume source
flux resume source git flux-system

# Port-forward to access application
kubectl port-forward svc/demoapp-svc 8080:80 -n demoapp-ns
```

---

## Kustomization

### Create a Kustomization

```powershell
# Kustomization from Git source
flux create kustomization kyverno \
    --source=GitRepository/kyverno \
    --path="./config/release" \
    --prune=true \
    --interval=60m \
    --wait=true \
    --health-check-timeout=3m

# Export to YAML file
flux create kustomization demo4-kustomization-git \
    --source=GitRepository/demo4-source-git \
    --prune=true \
    --target-namespace=demons \
    --path=./ \
    --export > demo4-kustomization-git.yaml
```

### Deploy and Monitor Kustomization

```powershell
# First create the Git source
flux create source git demo4-source-git \
    --url=https://github.com/yogeshraheja/kustomizedemo \
    --branch=main \
    --export > demo4-source-git.yaml

# Then create kustomization (as shown above)

# Check kustomizations
flux get kustomization

# View all resources deployed
kubectl get all -n demons

# Watch logs
flux logs -f

# Commit to Git
git add demo4-kustomization-git.yaml demo4-source-git.yaml
git commit -m "Added kustomization file"
git push
```

---

## Helm Controller ⭐ **IMPORTANT**

**Helm Controller** is the most powerful Flux CD feature for managing Helm releases declaratively in Git.

### Create Helm Repository Source

```powershell
flux create source helm demo6-source-helm \
    --url=https://charts.example.com
```

### Create Helm Release from Git Chart

```powershell
# First, create Git source pointing to repository with Helm charts
flux create source git demo5-source-git \
    --url=https://github.com/yogeshraheja/Argo-CD-for-the-Absolute-Beginners.git \
    --branch=main \
    --export > demo5-source-git.yaml

# Create Helm Release from chart stored in Git
flux create hr demo5-hr-git \
    --chart=Section10_Helm_Charts/demotest \
    --source=GitRepository/demo5-source-git \
    --export > demo5-hr-git.yaml

# Monitor
flux get sources all
flux get hr
kubectl get ns
kubectl get all -n demotest-ns
```

### Create Helm Release from Helm Repository

```powershell
# Using Helm repository source with values file
flux create hr demo6-helm-helmrepo \
    --chart=nginx \
    --source=HelmRepository/demo6-source-helm \
    --values=../myhelm_values.yaml \
    --chart-version=22.5.3 \
    --export > demo6-helm-helmrepo.yaml

# Check releases
flux get hr

# View all namespaces
kubectl get ns
```

---

## Object Storage (S3/Bucket Sources)

### Setup S3 Credentials

```powershell
# Create secret with AWS credentials
kubectl create secret generic s3-secret \
    --from-literal=accesskey=AKIAW6232UDFIX2B2NPW \
    --from-literal=secretkey=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 \
    -n flux-system
```

### Create MinIO Bucket Source (On-Cluster)

```powershell
flux create source bucket podinfo \
    --bucket-name=podinfo \
    --endpoint=minio.minio.svc.cluster.local:9000 \
    --insecure=true \
    --access-key=myaccesskey \
    --secret-key=mysecretkey \
    --secret-ref= \
    --interval=10m
```

### Create AWS S3 Bucket Source

```powershell
flux create source bucket demo3-source-s3 \
    --bucket-name=fluxdemobucket321 \
    --endpoint=s3.amazonaws.com \
    --secret-ref=s3-secret \
    --provider=aws \
    --region=us-east-1 \
    --export > demo-source-s3.yaml
```

### Create Kustomization from Bucket Source

```powershell
flux create kustomization demos3-kustomization-s3 \
    --source=Bucket/demo3-source-s3 \
    --target-namespace=default \
    --path=./ \
    --prune \
    --export > demo3-kustomization-s3.yaml
```

---

## OCI Registry Sources

### Login to OCI Registry

```powershell
# Login to GHCR
docker login ghcr.io
```

### Push Manifests to OCI

```powershell
# Navigate to manifest directory first
# Then push as OCI artifact
flux push artifact oci://ghcr.io/soumen321/fluxcdplaygroudmanifest:$(git rev-parse --short HEAD) \
    --path=./ \
    --source="$(git config --get remote.origin.url)" \
    --revision="$(git branch --show-current)@sha1:$(git rev-parse HEAD)"
```

### Create OCI Secret for Authentication

```powershell
flux create secret oci podinfo-auth \
    --url=ghcr.io \
    --username=soumen321 \
    --password=ghp_YOUR_GITHUB_TOKEN \
    --export > repo-auth.yaml
```

### Create OCI Repository Source

```powershell
flux create source oci demo7-source-oci \
    --url=oci://ghcr.io/soumen321/fluxcdplaygroudmanifest \
    --tag=0b1d3fa \
    --secret-ref=ghcr-secret \
    --export > demo7-source-oci.yaml

# Create kustomization from OCI source
flux create kustomization demo7-kustomize-oci \
    --source=OCIRepository/demo7-source-oci \
    --target-namespace=ocidemo-ns \
    --prune=true \
    --export > demo7-kustomize-oci.yaml

# Check OCI sources
flux get sources oci
```

### Package and Push Helm Chart to OCI

```powershell
# Navigate to chart directory
cd ocidemo-chart

# Package Helm chart
helm package ocidemo-chart

# Login to OCI registry
helm registry login ghcr.io --username=soumen321

# Push package to OCI
helm push ocidemo-chart-1.1.1.tgz oci://ghcr.io/soumen321
```

### Create Helm Release from OCI Repository

```powershell
# Create OCI source for Helm chart
flux create source oci demo8-source-oci \
    --url=oci://ghcr.io/soumen321/ocidemo-chart \
    --tag=1.1.1 \
    --secret-ref=ghcr-secret \
    --export > demo8-source-oci.yaml

# Create Helm Release from OCI source
flux create hr demo8-helm-oci \
    --chart-ref=OCIRepository/demo8-source-oci \
    --export > demo8-helm-oci.yaml

# Check OCI sources
flux get source oci
```

---

## Image Automation

### Enable Image Automation Controllers

Enable image reflector and automation controllers during bootstrap:

```powershell
flux bootstrap github \
    --token-auth \
    --owner=soumen321 \
    --repository=FluxCDPlaygroud \
    --branch=main \
    --path=clusters/fluxcluster \
    --private=false \
    --personal=true \
    --components-extra=image-reflector-controller,image-automation-controller
```

### Verify Image Controllers

```powershell
# Check all flux-system components
kubectl get all -n flux-system

# Check API resources
kubectl api-resources

# Check Custom Resource Definitions
kubectl get crd
```

### Create Image Repository

```powershell
# Create image repository to monitor Docker registry
flux create image repository demo9-image-repo \
    --image=yogeshraheja/fluxdemo \
    --export > demo9-image-repo.yaml

# Reconcile to apply
flux reconcile source git flux-system

# Check all image resources
flux get image all

# View image repositories
flux get image repository
```

### Create Image Policy

```powershell
# Define which image tags to consider
flux create image policy demo9-image-policy \
    --image-ref=demo9-image-repo \
    --select-semver=">=1.x.0x" \
    --export > demo9-image-policy.yaml

# Check policies
flux get image policy
```

### Create Image Update Automation

```powershell
# Auto-update Git manifests with new image tags
flux create image update demo9-image-update \
    --git-repo-ref=flux-system \
    --git-repo-path="./clusters/fluxcluster" \
    --checkout-branch=main \
    --author-name=soumen \
    --author-email=bhattacharjee.soumen@gmail.com \
    --push-branch=main \
    --export > demo9-image-update.yaml
```

---

## Monitoring

### Setup Monitoring Infrastructure

Reference: [fluxcd/flux2-monitoring-example](https://github.com/fluxcd/flux2-monitoring-example)

Includes:
- **kube-prometheus-stack**: Prometheus, Alertmanager, Grafana
- **loki-stack**: Log aggregation

### Create Git Source for Monitoring

```powershell
flux create source git demo10-source-monitoring-git \
    --url=https://github.com/fluxcd/flux2-monitoring-example \
    --branch=main \
    --export > demo10-source-monitoring-git.yaml
```

### Deploy kube-prometheus-stack

```powershell
flux create kustomization demo10-kustomization-monitoring-git \
    --source=demo10-source-monitoring-git \
    --path=./monitoring/controllers/kube-prometheus-stack \
    --export > demo10-kustomization-monitoring-git.yaml
```

### Deploy Loki Stack (Logging)

```powershell
flux create kustomization demo10-loki-stack \
    --source=demo10-source-monitoring-git \
    --path=./monitoring/controllers/loki-stack \
    --export > demo10-loki-stack.yaml
```

---

## Sealed Secrets

### About Sealed Secrets

**Sealed Secrets** (by Bitnami) encrypts secrets in Git repositories safely.

### Install Sealed Secrets Controller

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml
```

### Create and Seal a Secret

```bash
# Create secret (dry-run)
kubectl create secret generic mysecret \
    --from-literal=password=mysecretpassword \
    --dry-run=client -o yaml | \
    kubeseal -f - > mysealedsecret.yaml

# Apply sealed secret
kubectl apply -f mysealedsecret.yaml

# Safe to commit to Git
git add mysealedsecret.yaml
git commit -m "Add sealed secret"
git push
```

---

## Common Commands

### Source Commands

```powershell
# List all sources
flux get source all -A

# Git sources
flux get source git
flux get source git -n <namespace>

# Helm sources
flux get source helm

# OCI sources
flux get source oci

# Reconcile source
flux reconcile source git flux-system

# Manually refresh
flux reconcile source git <name> -n <namespace>

# Suspend
flux suspend source git flux-system

# Resume
flux resume source git flux-system
```

### Kustomization Commands

```powershell
# List kustomizations
flux get kustomization

# Get details
kubectl describe kustomization <name>

# Reconcile
flux reconcile kustomization <name>

# View resources
kubectl get all -n <namespace>

# Watch logs
flux logs -f
```

### Helm Release Commands

```powershell
# List all Helm releases
flux get hr
flux get hr -A

# Get release details
kubectl describe hr <name> -n <namespace>

# Check values
kubectl get hr <name> -n <namespace> -o yaml

# Reconcile
flux reconcile hr <name> -n <namespace>
```

### Logging and Debugging

```powershell
# Watch all Flux logs
flux logs -f

# Watch specific resource type
flux logs -f --kind=Kustomization
flux logs -f --kind=HelmRelease
flux logs -f --kind=GitRepository

# All namespaces
flux logs -f --all-namespaces

# Specific namespace
flux logs -f -n flux-system
```

---

## Troubleshooting

### Source Not Syncing

```powershell
# Check Git source status
kubectl describe gitrepository <name> -n flux-system

# View reconciliation logs
flux logs -f --kind=GitRepository

# Force reconcile
flux reconcile source git <name> -n flux-system
```

### Helm Release Failed

```powershell
# Check HelmRelease status
kubectl describe hr <name> -n <namespace>

# Check controller logs
kubectl logs -f -n flux-system -l app=helm-controller --tail=100

# View release details
kubectl get hr <name> -n <namespace> -o yaml
```

### Kustomization Issues

```powershell
# Check kustomization status
flux get kustomization <name>

# View detailed status
kubectl describe kustomization <name> -n flux-system

# Check logs
flux logs -f --kind=Kustomization <name>
```

### Check Flux System Status

```powershell
# All Flux components
kubectl get all -n flux-system

# Check for errors
kubectl get events -n flux-system --sort-by='.lastTimestamp'

# Controller logs
kubectl logs -f -n flux-system <pod-name>
```

---

## Best Practices

1. ✅ Version control everything in Git
2. ✅ Use meaningful commit messages
3. ✅ Implement RBAC for controllers
4. ✅ Monitor reconciliation status regularly
5. ✅ Use Sealed Secrets for sensitive data
6. ✅ Test in dev before prod
7. ✅ Organize by namespaces
8. ✅ Document your GitOps workflow
9. ✅ Automate image updates with Image Automation
10. ✅ Monitor with Prometheus and Grafana

---

## Quick Reference

| Task | Command |
|------|---------|
| Check sources | `flux get source all -A` |
| Check releases | `flux get hr -A` |
| Watch logs | `flux logs -f` |
| Manual reconcile | `flux reconcile source git <name>` |
| Suspend sync | `flux suspend source git <name>` |
| Resume sync | `flux resume source git <name>` |
| View details | `kubectl describe <resource> <name>` |
| Apply changelog | `git add . && git commit -m "msg" && git push` |

---

**Last Updated**: 2026-03-06  
**Version**: 2.0  
**Comprehensively Organized for Easy Learning**





