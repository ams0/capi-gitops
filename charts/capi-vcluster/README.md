# CAPI vcluster helm chart

An helm chart to deploy vclusters via CAPI provider

## Prerequisites

Install CAPI and the vcluster infrastrucure provider

```bash
clusterctl init --infrastructure vcluster
```

## Deploy

Create a cluster with:

```bash
kubectl create namespace vc2
helm template -n vc2 --create-namespace --set cluster.name=vc2 vc2 . | kubectl apply -f -
```
