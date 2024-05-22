# Ephemeral Clusters as a Service with vcluster, ClusterAPI and ArgoCD

![License](https://img.shields.io/badge/license-MIT-green.svg)

- Session @KubeconEU 2023 in Amsterdam: [<https://sched.co/1HyXe>](https://sched.co/1HyXe)
- Session @Open Source Summit 2023 in Vancouver: [<https://sched.co/1K5IB>](https://sched.co/1K5IB)

## Overview

Welcome to Ephemeral Clusters as a Service with [vClusters](https://www.vcluster.com), [ClusterAPI](https://cluster-api.sigs.k8s.io) and [ArgoCD](https://argo-cd.readthedocs.io/en/stable/). GitOps has rapidly gained popularity in recent years, with its many benefits over traditional CI/CD tools. However, with increased adoption comes the challenge of managing multiple Kubernetes clusters across different cloud providers. At scale, ensuring observability and security across all clusters can be particularly difficult.

This repository demonstrates how open-source tools, such as ClusterAPI, ArgoCD, and Prometheus+Thanos, can be used to effectively manage and monitor large-scale Kubernetes deployments. We will walk you through a sample that automates the deployment of several clusters and applications securely and with observability in mind.

## Prerequisites

This branch works with a kind cluster, to keep the cost low while experimenting. I use kind with MetalLB and Cilium, to have also working Loadbalancer type of services; I have a handy script on [gist](https://gist.github.com/ams0/4f1063be9e8d5c34fc85a1b4857aed71) that will help you.

- [Helm CLI](https://helm.sh) and `envsubst`

Everything else is installed via ArgoCD, so no need for any extra CLI!

## Step 1: Create an ArgoCD management cluster with kind

```bash
kind_cilium.sh -k v1.30.0 -n capi -i ghcr.io/fluxcd/kindest/node
```


## Step 2:  Install ArgoCD

ArgoCD is an open-source continuous delivery tool designed to simplify the deployment of applications to Kubernetes clusters. It allows developers to manage and deploy applications declaratively, reducing the amount of manual work involved in the process. With ArgoCD, developers can automate application deployment, configuration management, and application rollouts, making it easier to deliver applications reliably to production environments.

You can install ArgoCD on your Kubernetes cluster by running the following commands in your terminal or command prompt. These commands will download and install ArgoCD on your cluster, allowing you to use it for GitOps-based continuous delivery of your applications

> **_NOTE:_** Make sure to update the values for ingress hostname in the various helm charts under `gitops/management` folder; Also make sure to update the external-dns values to include your tenantId and subscriptionId. we will update the readme when we find a better way to dynamically inject these values into the helm charts deployed by ArgoCD

Add ArgoCD Helm Repo:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Edit the `gitops/management/argocd/argocd-kind-values.yaml` with your hostname and domain name for ArgoCD ingress, then install ArgoCD:

```bash
helm upgrade -i -n argocd \
  --version 6.10.2 \
  --create-namespace \
  --values gitops/management/argocd/argocd-kind-values.yaml \
  argocd argo/argo-cd
```

Next, we'll use the [argocd-apps](https://artifacthub.io/packages/helm/argo/argocd-apps) Helm chart to deploy the initial objects (the `infra` AppProject and the app-of-apps Application pointing to `gitops/management` folder in this repo):

```bash
helm upgrade -i -n argocd \
  --version 2.0.0 \
  --create-namespace \
  --values argocd-initial-objects.yaml \
  argocd-apps argo/argocd-apps
```

Verify that ArgoCD is running:

```bash
kubectl get pods -n argocd
```

Access the ArgoCD web UI by running the following command, and then open the URL in a web browser (ingress, external-dns and cert-manager take care of certificates and DNS hostname resolution):

```bash
open https://argocd.$AZURE_DNS_ZONE
open https://dash.$AZURE_DNS_ZONE

```

> **_NOTE:_** ArgoCD is in read-only mode for anonymous users, that should be enough to monitor the installaation progress, but if you want to change things, retrieve the secret with: 
> `kubectl get secret -n argocd argocd-initial-admin-secret  -o=jsonpath='{.data.password}'| base64 -D`

## Step 3:  Bootstrap Management Cluster with ClusterAPI

To initialize the AKS cluster with Cluster API and turn it into the management cluster, follow these instructions. Once initialized, the management cluster will allow you to control and maintain a fleet of ephemeral clusters. Unfortunately this part cannot be automated via ArgoCD just yet, although a promising effort is made in the `capi-operator` [repository](https://github.com/kubernetes-sigs/cluster-api-operator/tree/main):

```bash
# Run the script, passing the namespace as a parameter (the Azure Managed Identity for the workload clusters)

./capz-init.sh 

# Check the providers
kubectl get providers.clusterctl.cluster.x-k8s.io -A
```

## Step 4:  Deploy clusters via Pull Requests

Open a PR against your main branch, modifying the appset-capz.yaml or appset-vcluster.yaml to deploy your ephemeral clusters!


## Clean up

Delete the cluster:

```bash
az aks delete --resource-group $CLUSTER_RG --name $CLUSTER_NAME --no-wait -y
```

Delete the identity:

```bash
az identity delete --id $IDENTITY
```

